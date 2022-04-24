---
title: "Android Enterprise: Enrolling Using Mobile Data"
date: 2022-03-21T18:07:33Z
draft: false
description: ""
tags: ["endpoint", "intune", "android"]
ShowToc: true
cover:
    image: "/img/android-enrol.png" # image path/url
    alt: "Enrolling Android Enterprise devices on mobile data" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---
With the change to Android 10+ requiring a [wireless network](https://support.google.com/work/android/thread/88876136?hl=en) to go through the Fully Managed device enrolment, you may be asking, "Well what if my users don't have access to a wireless network?", don't fret, with a bit of effort you can regenerate a new QR code that allows the use of Mobile Data.

# Configuration
The below sections detail the steps to generate a new QR code for enrolment, allowing the use of Mobile Data.

## Get the QR Code Data
Use QR Reader on an existing phone or using an [online reader](https://zxing.org/w/decode.jspx) to get the full QR code data:

```json {linenos=false}
{"android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME":"com.google.android.apps.work.clouddpc/.receivers.CloudDeviceAdminReceiver","android.app.extra.PROVISIONING_DEVICE_ADMIN_SIGNATURE_CHECKSUM":"I5YvS0O5hXY46mb01BlRjq4oJJGs2kuUcHvVkAPEXlg","android.app.extra.PROVISIONING_DEVICE_ADMIN_PACKAGE_DOWNLOAD_LOCATION":"https://play.google.com/managed/downloadManagingApp?identifier=setup","android.app.extra.PROVISIONING_ADMIN_EXTRAS_BUNDLE":{"com.google.android.apps.work.clouddpc.EXTRA_ENROLLMENT_TOKEN":"TOKENVALUE"}}
```

## Updating the JSON content
Add in the below code snippet before the **"android.app.extra.PROVISIONING_ADMIN_EXTRAS_BUNDLE"** section:

```json {linenos=false}
"android.app.extra.PROVISIONING_USE_MOBILE_DATA":true,
```

So the full JSON string should look like the below, with the **TOKENVALUE** obviously the correct one:
	
```json {linenos=false}
{"android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME":"com.google.android.apps.work.clouddpc/.receivers.CloudDeviceAdminReceiver","android.app.extra.PROVISIONING_DEVICE_ADMIN_SIGNATURE_CHECKSUM":"I5YvS0O5hXY46mb01BlRjq4oJJGs2kuUcHvVkAPEXlg","android.app.extra.PROVISIONING_DEVICE_ADMIN_PACKAGE_DOWNLOAD_LOCATION":"https://play.google.com/managed/downloadManagingApp?identifier=setup","android.app.extra.PROVISIONING_USE_MOBILE_DATA":true,"android.app.extra.PROVISIONING_ADMIN_EXTRAS_BUNDLE":{"com.google.android.apps.work.clouddpc.EXTRA_ENROLLMENT_TOKEN":"TOKENVALUE"}}
```    

## Create a New QR Code    
Copy the string and paste it into an [online QR code generator](https://www.webtoolkitonline.com/qrcode-generator.html) to generate the new QR code. This can then be provided to your users, pending testing, to allow them to enrol their new Android device in Endpoint Manager, whether connected to wireless or mobile data.