---
title: "Endpoint Manager: Device Filters vs Dynamic Groups"
date: 2022-06-05T10:25:47+01:00
draft: true
description: ""
tags: ["endpoint", "intune", "groups"]
ShowToc: true
cover:
    image: "/img/" # image path/url
    alt: "" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---
Now if you've ever spoken to me about Endpoint Manager and using Dynamic Groups for management of users and devices, I probably would have talked your ears off about the attribute usage, which attributes are suitable, and that moving away from assigned groups to dynamic is the only way forward for Modern Device Management.

What if I told you, that there's something on par with these holy grail groups, and maybe, just *maybe*, even better.

# Limitations of Dynamic Groups

Before we jump into [Device Filters](https://docs.microsoft.com/en-us/mem/intune/fundamentals/filters), let us talk about Dynamic Groups a little...

Firstly, I would highly recommend the use of these groups, especially for grouping devices, whether this be based on enrolment type, ownership, operating system type or version...and if you've got something or someone managing your user attributes, that you use them for user groups. Although the number of attributes that can be used is higher compared with Device Filters, there is a trade off.

This sadly, is how quickly these groups update; Microsoft probably realised that people like me were using these groups in relation to device management, and also realised that the enumeration of the groups was using their precious Computer infrastructure, for free, shock. So Microsoft reduced how often these groups fully updated, to once every 24 hours.

Now this is no good when we want to target settings and restrictions, or even just application deployment to these dynamically populated groups, we end up with delaying installations, configuration settings or even connectivity. Not a fan.

# Bring on the Filters
So where Microsoft take away with one hand, they give with the other, and this is the new world of Device Filters. These, although have fewer attributes to query, allow 



(device.deviceOwnership -eq "Corporate") and (device.enrollmentProfileName -in [""]) and (device.osVersion -startsWith "12.1")

https://github.com/MicrosoftDocs/memdocs/blob/main/memdocs/intune/fundamentals/filters-device-properties.md