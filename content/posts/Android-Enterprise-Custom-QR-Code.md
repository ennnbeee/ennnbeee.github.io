---
title: "Android Enterprise: Customising the Enrolment QR Code"
date: 2022-05-04T10:34:53+01:00
draft: true
description: ""
tags: ["endpoint", "intune", "android"]
ShowToc: true
cover:
    image: "/img/android-custom.png" # image path/url
    alt: "Customising Android Enterprise enrolment QR codes" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---
With the change to Android 10+ requiring a [wireless network](https://support.google.com/work/android/thread/88876136?hl=en) to go through the Fully Managed device enrolment, you may be asking, "Well what if my users don't have access to a wireless network?", don't fret, with a bit of effort you can regenerate a new QR code that allows the use of Mobile Data.

[Android Enterprise Enrolment with Mobile Data](https://memv.ennbee.uk/posts/android-enterprise-enrolment-mobile-data/)

# Configuration
The below sections detail the steps to generate a new QR code for enrolment, allowing the use of Mobile Data.

## Get the QR Code Data
Use QR Reader on an existing phone or using an [online reader](https://zxing.org/w/decode.jspx) to get the full QR code data:

> A String extra indicating the security type of the wifi network in `EXTRA_PROVISIONING_WIFI_SSID` and could be one of `NONE, WPA, WEP or EAP`.

[WIFI_SECURITY_TYPE](https://developer.android.com/reference/android/app/admin/DevicePolicyManager#EXTRA_PROVISIONING_WIFI_SECURITY_TYPE)

```json
{
    "android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME":"com.google.android.apps.work.clouddpc/.receivers.CloudDeviceAdminReceiver",
    "android.app.extra.PROVISIONING_DEVICE_ADMIN_PACKAGE_DOWNLOAD_LOCATION":"https://play.google.com/managed/downloadManagingApp?identifier=setup",
    "android.app.extra.PROVISIONING_DEVICE_ADMIN_SIGNATURE_CHECKSUM":"I5YvS0O5hXY46mb01BlRjq4oJJGs2kuUcHvVkAPEXlg",
    "android.app.extra.PROVISIONING_ADMIN_EXTRAS_BUNDLE":{
       "com.google.android.apps.work.clouddpc.EXTRA_ENROLLMENT_TOKEN":"SELECTED ENROLLMENT TOKEN"
    },
    "android.app.extra.PROVISIONING_WIFI_SSID":"WIFI_SSID",
    "android.app.extra.PROVISIONING_WIFI_PASSWORD":"WIFI_PASSWORD",
    "android.app.extra.PROVISIONING_WIFI_SECURITY_TYPE":"NONE/WPA/WEP/EAP",
    "android.app.extra.PROVISIONING_WIFI_HIDDEN":true/false
 }
```

## Updating the JSON content
Add in the below code snippet before the `android.app.extra.PROVISIONING_ADMIN_EXTRAS_BUNDLE` section:

```json
"android.app.extra.PROVISIONING_USE_MOBILE_DATA":true,
```

So the full JSON string should look like the below, with the `TOKENVALUE` obviously the correct one:
	
```json
{
    "android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME":"com.google.android.apps.work.clouddpc/.receivers.CloudDeviceAdminReceiver",
    "android.app.extra.PROVISIONING_DEVICE_ADMIN_PACKAGE_DOWNLOAD_LOCATION":"https://play.google.com/managed/downloadManagingApp?identifier=setup",
    "android.app.extra.PROVISIONING_DEVICE_ADMIN_SIGNATURE_CHECKSUM":"I5YvS0O5hXY46mb01BlRjq4oJJGs2kuUcHvVkAPEXlg",
    "android.app.extra.PROVISIONING_ADMIN_EXTRAS_BUNDLE":{
       "com.google.android.apps.work.clouddpc.EXTRA_ENROLLMENT_TOKEN":"SELECTED ENROLLMENT TOKEN"
    },
    "android.app.extra.PROVISIONING_WIFI_SSID":"",
    "android.app.extra.PROVISIONING_WIFI_PASSWORD":"",
    "android.app.extra.PROVISIONING_WIFI_SECURITY_TYPE":"",
    "android.app.extra.PROVISIONING_WIFI_HIDDEN":"false"
 }
```    

## Create a New QR Code    
Copy the string and paste it into an [online QR code generator](https://www.webtoolkitonline.com/qrcode-generator.html) to generate the new QR code. This can then be provided to your users, pending testing, to allow them to enrol their new Android device in Endpoint Manager, whether connected to wireless or mobile data.
