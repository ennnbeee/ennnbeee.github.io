---
title: "Windows Update for Business: Autopilot 50:50 Production Deployment Rings"
date: 2022-04-10T20:07:23+01:00
draft: false
description: ""
tags: ["endpoint", "intune", "windows","updates","autopilot"]
ShowToc: true
cover:
    image: "/img/update-rings.png" # image path/url
    alt: "" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
![Image](/img/update-rings.png#center)


You might have been asked the question, especially from SCCM users, about how you split out production groups for Windows Update for Business (WUfB) Update Rings in an intelligent way using Microsoft Endpoint Manager and Azure AD groups for Autopilot devices.

Well Azure AD dynamic groups are you friend on this one, and they may not be as powerful as SCCM collection queries, but they get the job done.

Here is quick way to split production phased deployments of WUfB rings to dynamic device groups, based based on the device name ending in an odd or even number.

## Sample Devices
Lets use the following device names as examples:
```txt {linenos=false,hl_lines=0}
AVR-BDZX4M3
AVR-3231EQSM
PGT-45E8LN1S
IT-CRJ7C39
```

In these examples, the computers have been named with the prefix **"AVR-"** as part of the Autopilot deployment, followed by the serial number. Now serial numbers come in all shapes and sizes, but we should be able to split them in half.

## Dynamic Device Membership Rule
We're going to create a Dynamic Device group in Azure AD that looks for Corporate Windows devices that match a pattern, in this case we're looking for odd numbers, and alternative letters. This should cover off all serial number types.

### "Odd" Devices
```txt {linenos=false,hl_lines=0}
(device.deviceOSType -eq "Windows") and (device.deviceOSVersion -startsWith "10") and (device.deviceOwnership -eq "Company") and ((device.displayName -match "^*-1") or (device.displayName -match "^*-3") or (device.displayName -match "^*-5") or (device.displayName -match "^*-7") or (device.displayName -match "^*-9") or (device.displayName -match "^*-A") or (device.displayName -match "^*-C") or (device.displayName -match "^*-E") or (device.displayName -match "^*-G") or (device.displayName -match "^*-I") or (device.displayName -match "^*-K") or (device.displayName -match "^*-M") or (device.displayName -match "^*-O") or (device.displayName -match "^*-Q") or (device.displayName -match "^*-S") or (device.displayName -match "^*-U") or (device.displayName -match "^*-W") or (device.displayName -match "^*-Y"))
```

This query will match Windows 10 devices, corporate owned, that start with any letter or letters then a **'-'** and devices that follow with in an **odd** number or the alternative letters, so from the sample devices:

```txt {linenos=false,hl_lines=[2 4]}
AVR-BDZX4M3
AVR-3231EQSM
PGT-45E8LN1S
IT-CRJ7C39
```

### "Even" Devices
```txt {linenos=false,hl_lines=0}
(device.deviceOSType -eq "Windows") and (device.deviceOSVersion -startsWith "10") and (device.deviceOwnership -eq "Company") and ((device.displayName -match "^*-0") or (device.displayName -match "^*-2") or (device.displayName -match "^*-4") or (device.displayName -match "^*-6") or (device.displayName -match "^*-8") or (device.displayName -match "^*-B") or (device.displayName -match "^*-D") or (device.displayName -match "^*-F") or (device.displayName -match "^*-H") or (device.displayName -match "^*-J") or (device.displayName -match "^*-L") or (device.displayName -match "^*-N") or (device.displayName -match "^*-P") or (device.displayName -match "^*-R") or (device.displayName -match "^*-T") or (device.displayName -match "^*-V") or (device.displayName -match "^*-X") or (device.displayName -match "^*-Z"))
```

This query will match Windows 10 devices, corporate owned, that start with any letter or letters then a **'-'** and devices that follow with in an **even** number or the alternative letters, so from the sample devices:

```txt {linenos=false,hl_lines=[1 3]}
AVR-BDZX4M3
AVR-3231EQSM
PGT-45E8LN1S
IT-CRJ7C39
```


## Regex Overview
With the Match operator in dynamic groups using [Regex](https://docs.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-language-quick-reference) this can be altered to fit many device or user scenarios, like Autopilot devices with random names or the serial number in the name...you just have to use your brain a bit to find a suitable 50:50 split of devices.

Once you have your groups in place, and they look about right, you can then assign your Production level WUfB Update Rings to the groups, ensuring that not all 10000+ devices are getting, and installing updates at the same time.    