---
title: "Windows Update for Business: 50:50 Production Deployment Rings"
date: 2022-03-27T20:07:23+01:00
draft: false
description: ""
tags: ["endpoint", "intune", "windows","updates"]
ShowToc: true
cover:
    image: "/img/update-rings.png" # image path/url
    alt: "" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
![Image](/img/update-rings.png#center)


You might have been asked the question, especially from SCCM users, about how you split out production groups for Windows Update for Business (WUfB) Update Rings in an intelligent way using Microsoft Endpoint Manager and Azure AD groups.

Well Azure AD dynamic groups are you friend on this one, and they may not be as powerful as SCCM collection queries, but they get the job done.

Here is quick way to split production phased deployments of WUfB rings to dynamic device groups, based based on the device name ending in an odd or even number.

## Sample Devices
Lets use the following device names as examples:
```txt {linenos=false,hl_lines=0}
NC013572
DC013575
PC022445545458
ERESSRC4587877
```

In these examples, **NC** is Notebook Computer and **DC** Desktop Computer, your mileage may vary based on how the devices are named.

## Dynamic Device Membership Rule
For devices that end with even numbers, we're looking for the ones ending with 0, 2, 4, 6, and 8, and odd numbers, well, it's the other numbers...but we could do with a bit more logic than just this, so we'll throw in that the device has a 'C' in the name.

### Even Devices
```txt {linenos=false,hl_lines=0}
(device.deviceOSType -eq "Windows") and (device.deviceOSVersion -startsWith "10") and (device.accountEnabled -eq True) and ((device.displayName -match "^*C*.0$") or (device.displayName -match "^*C*.2$") or (device.displayName -match "^*C*.4$") or (device.displayName -match "^*C*.6$") or (device.displayName -match "^*C*.8$"))
```

This query will match Windows 10 devices that start with any letter or letters then a **'C'** and devices that end in an **even** number, so from the sample devices:

```txt {linenos=false,hl_lines=[1 3]}
NC013572
DC013575
PC02244554545
ERESSRC4587877
```

### Odd Devices
```txt {linenos=false,hl_lines=0}
(device.deviceOSType -eq "Windows") and (device.deviceOSVersion -startsWith "10") and (device.accountEnabled -eq True) and ((device.displayName -match "^*C*.1$") or (device.displayName -match "^*C*.3$") or (device.displayName -match "^*C*.5$") or (device.displayName -match "^*C*.7$") or (device.displayName -match "^*C*.9$"))
```

This query will match Windows 10 devices that start with any letter or letters then a **'C'** and devices that end in an **odd** number, so from the sample devices:

```txt {linenos=false,hl_lines=[2 4]}
NC013572
DC013575
PC02244554545
ERESSRC4587877
```

## Regex Overview
With the Match operator in dynamic groups using [Regex](https://docs.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-language-quick-reference) this can be altered to fit many device or user scenarios, like Autopilot devices with random names or the serial number in the name...you just have to use your brain a bit to find a suitable 50:50 split of devices.

Once you have your groups in place, and they look about right, you can then assign your Production level WUfB Update Rings to the groups, ensuring that not all 10000+ devices are getting, and installing updates at the same time. 

### Overview of Expression

^*C*0$
```txt
^ - starts with
* - repetition operator
C - character to look for
7 - character to look for
$ - end of line
```

