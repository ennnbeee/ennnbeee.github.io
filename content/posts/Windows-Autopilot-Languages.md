---
title: "Windows Autopilot: Setting Available User Languages"
date: 2022-04-27T16:22:53+01:00
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
Intro here

# Configuration
Details on the configuration


## Windows User Language List
You may be familiar with the 'old' way of setting user languages in Windows 7, calling `` and assigning an xml file with the required user languages, luckily with Windows 10 comes the [Set-WinUserLanguageList](https://docs.microsoft.com/en-us/powershell/module/international/new-winuserlanguagelist) PowerShell command.



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
This script allows the creation and installation of a User Language List, containing the required languages needed for each user. This seemed all a bit too perfect for this requirement; updatable list of languages, uses native PowerShell commands, can be deployed via Endpoint Manager. That was until I realised this only works correctly under a user context, and I also need it to run each time a user logs on to the machine



## Deployment
Save the above script and create a new PowerShell script deployment in [Endpoint manager](https://endpoint.microsoft.com/#blade/Microsoft_Intune_DeviceSettings/DevicesWindowsMenu/powershell) using the following configuration settings, then deploy to a test group of devices.

![Image](/img/computer-name-script.png#left)
 
 Bingo! another battle won.


# Conclusion
