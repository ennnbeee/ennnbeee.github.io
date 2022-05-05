---
title: "Endpoint Manager Device Filters"
date: 2022-05-05T10:25:47+01:00
draft: true
description: ""
tags: ["endpoint", "intune"]
ShowToc: true
cover:
    image: "/img/" # image path/url
    alt: "" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---

(device.deviceOwnership -eq "Corporate") and (device.enrollmentProfileName -in [""]) and (device.osVersion -startsWith "12.1")

https://github.com/MicrosoftDocs/memdocs/blob/main/memdocs/intune/fundamentals/filters-device-properties.md