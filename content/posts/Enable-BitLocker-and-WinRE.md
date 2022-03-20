---
title: "Enable BitLocker and WinRE on failed Intune Devices"
date: 2022-03-17T16:44:09Z
draft: false
description: ""
tags: ["endpoint", "intune", "windows", "bitlocker"]
ShowToc: true
cover:
    image: "/bitlocker.jpg" # image path/url
    alt: "Enable BitLocker and WinRE on failed Intune Devices" # alt text
    caption: "Enable BitLocker and WinRE on failed Intune Devices" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
![Image](/bitlocker.jpg)


You may have enabled and configure BitLocker for silent encryption on your Windows 10 Autopilot joined devices, but have you had the headache of devices that don't have a Windows Recovery Environment (WinRE) configured? Yep? Me too...

What you'll see in either the Bitlocker-API event log, or within the Encryption Readiness reporting in Endpoint Manager the following, glorious error:
```txt {linenos=false}
The OS volume is unprotected | Windows Recovery Environment (WinRE) isn't configured
```

So how do we go about enabling WinRE if it exists, setup BitLocker encryption, **and** grab the BitLocker recovery key and ping it to Azure AD?

Here's how...

## Step 1: Create the Script
This [Microsoft script](https://docs.microsoft.com/en-us/archive/blogs/showmewindows/how-to-enable-bitlocker-and-escrow-the-keys-to-azure-ad-when-using-autopilot-for-standard-users) has been adapted to check for the WinRE configuration before it continues and attempts to enable BitLocker, the **$hottotrot** variable is used to denote whether to continue or not:
```Powershell {linenos=table,hl_lines=["11"],linenostart=1}
$HotToTrot ="false"
#Checks Windows Recovery Environment and enables if disabled
if($WinREStatus -like '*Windows RE status:         Enabled*'){
    $HotToTrot = "True"
    Write-Verbose -Message "WinRE Partion Enabled and good to enable BitLocker $HotToTrot"
}
Else{
    Try{
        $WinREEnable = reagentc.exe /enable
        if($WinREEnable -like '*Operation Successful*'){
            $HotToTrot = "True"
            Write-Verbose -Message "WinRE Partion Enabled and good to enable BitLocker, HotToTrot set to $HotToTrot"
        }
        Else{
            $HotToTrot ="false"
            Write-Verbose -Message "Unable to enabled WinRE, HotToTrot set to $HotToTrot"
        }
    }
    Catch{
        $HotToTrot ="false"
        Write-Verbose -Message "Unable to enabled WinRE"
    }
}
if($HotToTrot -eq 'True')
```
## Step 2: Test the Script

```sql {linenos=true}
SELECT *
FROM v_UpdateContents
WHERE Content_ID = '16816292'
```
## Step 3: Deploy to Everyone :)

## Footnotes
> The Microsoft provided script should no longer be required for just enabling BitLocker, this is entirely handled with Endpoint Manager now.