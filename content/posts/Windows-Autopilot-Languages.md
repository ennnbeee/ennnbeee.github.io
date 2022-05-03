---
title: "Windows Autopilot: Setting Available User Languages"
date: 2022-04-29T16:22:53+01:00
draft: true
description: ""
tags: ["endpoint", "intune", "autopilot", "windows"]
ShowToc: true
cover:
    image: "/img/autopilot-lang.png" # image path/url
    alt: "" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---
Ever wondered how to ensure that a number of languages are available for selection to end users on shared Windows 10 devices? The thought hadn't crossed my mind, but then again, you encounter new use cases and requirement on a weekly basis. This was one of those occasions, needing a Library Kiosk machine to have a set of languages available to users.

# Configuration
There are a number of posts out in the wild on how to fully change the Windows languages on a device by device basis, but in this instance we needed to keep the core language as `en-GB` but allow end users to select their preferred input language, and ensure that this language list can be modified in the event of new languages being required.

Let's crack on shall we...

## Windows User Language List
You may be familiar with the 'old' way of automating the setting user languages in [Windows 7](https://docs.microsoft.com/en-us/troubleshoot/windows-client/deployment/automate-regional-language-settings), calling `control.exe intl.cpl,,/f:"c:\Unattend.xml"` and assigning an [xml file](https://docs.microsoft.com/en-us/troubleshoot/windows-client/deployment/automate-regional-language-settings#sample-xml-answer-file) with the required user languages, luckily with Windows 10 comes the [Set-WinUserLanguageList](https://docs.microsoft.com/en-us/powershell/module/international/new-winuserlanguagelist) PowerShell command.

Microsoft does a great job of detailing how this command is used, so below is the way I've created a new list, and appended the required languages using the [region tag](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/available-language-packs-for-windows?view=windows-11).

This allows for new languages to be added to the script readily.

```PowerShell
# Sets the Windows Languages
$LanguageList = New-WinUserLanguageList -Language 'en-GB'
$Languages  = New-Object -TypeName System.Collections.ArrayList
$Languages.AddRange(@(
        "en-US",
        "ar-SA",
        "zh-HK",
        "zh-CN",
        "zh-TW",
        "el-GR",
        "he-IL",
        "ja-JP",
        "ko-KR",
        "zh-Hant-TW",
        "jp-JP",
        "am-ET",
        "Cy-az-AZ",
        "Lt-az-AZ",
        "fa-IR",
        "ka-GE",
        "el-GR",
        "gu-IN",
        "he-IL",
        "ru-KG",
        "ru-KZ",
        "mn-MN",
        "pa-IN",
        "ta-IN",
        "th-TH",
        "tr-TR",
        "ur-PK",
        "uz-Cyrl",
        "vi-VN"
    ))

Foreach($Language in $Languages){
	$LanguageList.Add($Language)
}

Try{
	Set-WinUserLanguageList -LanguageList $LanguageList -Force
	}
Catch{
	Write-Error "Unable to set the language list $($_.Exception.Message)"
}
```

## Task Scheduling
This script allows the creation and installation of a User Language List, containing the required languages needed for each user who accesses the shared device. This seemed all a bit too perfect for this requirement; updatable list of languages, uses native PowerShell commands, can be deployed via Endpoint Manager. That was until I realised this only works correctly under a user context, and I also need it to run each time a user logs on to the machine.

So now we need a way to create a Scheduled Task and have it run the PowerShell script, without the end user seeing it run.

Luckily, I was using a script to map network drives, generated from [here](https://intunedrivemapping.azurewebsites.net/) to do something similar, time to not reinvent the wheel and just make use of someone else's hard work.

## Deployment
Save the above script and create a new PowerShell script deployment in [Endpoint manager](https://endpoint.microsoft.com/#blade/Microsoft_Intune_DeviceSettings/DevicesWindowsMenu/powershell) using the following configuration settings, then deploy to a test group of devices.

![Image](/img/computer-name-script.png#left)
 
 Bingo! another battle won.

# Conclusion
