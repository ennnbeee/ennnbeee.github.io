---
title: "Android Enterprise: Customising the Enrolment QR Code"
date: 2022-05-11T10:34:53+01:00
draft: false
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
We've already looked at allowing [Android Enterprise enrolment using Mobile Data](https://memv.ennbee.uk/posts/android-enterprise-enrollment-mobile-data/) in a previous post, now it's time to look at some of the other provisioning values that can be used to create a custom enrolment QR Code.

# Configuration
This time, it's adding in a WiFi profile, to allow the devices to auto-connect as part of the enrolment process...anything to make life easier for users. 

## Extracting the QR Code Data
First off you'll need the QR code being used for your Android Enterprise enrolment, you can find this within the Android section of Endpoint Manager. Save this file to your computer and use an [online reader](https://products.aspose.app/barcode/recognize/qr) to get the full QR code data:

```json
{
   "android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME":"com.google.android.apps.work.clouddpc/.receivers.CloudDeviceAdminReceiver",
   "android.app.extra.PROVISIONING_DEVICE_ADMIN_SIGNATURE_CHECKSUM":"I5YvS0O5hXY46mb01BlRjq4oJJGs2kuUcHvVkAPEXlg",
   "android.app.extra.PROVISIONING_DEVICE_ADMIN_PACKAGE_DOWNLOAD_LOCATION":"https://play.google.com/managed/downloadManagingApp?identifier=setup",
   "android.app.extra.PROVISIONING_ADMIN_EXTRAS_BUNDLE":{
      "com.google.android.apps.work.clouddpc.EXTRA_ENROLLMENT_TOKEN":"SELECTED ENROLMENT TOKEN"
   }
}
```
## WiFi Settings
We're now going to add in four new lines of data into the existing JSON content from the [Android Developer Reference Guide](https://developer.android.com/reference/android/app/admin/DevicePolicyManager):

- This is the SSID of the wireless network you want to connect to.
   `"android.app.extra.PROVISIONING_WIFI_SSID":"WIFI_SSID"`
- This is the password of the wireless network.
   `"android.app.extra.PROVISIONING_WIFI_PASSWORD":"WIFI_PASSWORD"`
- This is the security type of the network, select either `none`, `WPA`, `WEP` or `EAP`
   `"android.app.extra.PROVISIONING_WIFI_SECURITY_TYPE":"NONE/WPA/WEP/EAP"`
- And finally whether the network is hidden to broadcast, using either `true` or `false`
   `"android.app.extra.PROVISIONING_WIFI_HIDDEN":true/false`

> *Now, please be aware that these settings are all available in plain text by anyone scanning the QR code, so I would recommend using a guest wireless network if you are going to allow you end users to use a corporate network to go through the enrolment process.*

## Updating the JSON Data
Now that we've got the required strings ready to be added, we need to update the existing JSON data. The below settings, need to go after the `android.app.extra.PROVISIONING_ADMIN_EXTRAS_BUNDLE"` section:

```json
"android.app.extra.PROVISIONING_WIFI_SSID":"corp-guest-wifi",
"android.app.extra.PROVISIONING_WIFI_PASSWORD":"supersecurepassword",
"android.app.extra.PROVISIONING_WIFI_SECURITY_TYPE":"WPA",
"android.app.extra.PROVISIONING_WIFI_HIDDEN": false
```

## Full JSON Data

So the full JSON string should look like the below, with the `SELECTED ENROLMENT TOKEN` the correct one from the original QR code:
	
```json
{
    "android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME":"com.google.android.apps.work.clouddpc/.receivers.CloudDeviceAdminReceiver",
    "android.app.extra.PROVISIONING_DEVICE_ADMIN_PACKAGE_DOWNLOAD_LOCATION":"https://play.google.com/managed/downloadManagingApp?identifier=setup",
    "android.app.extra.PROVISIONING_DEVICE_ADMIN_SIGNATURE_CHECKSUM":"I5YvS0O5hXY46mb01BlRjq4oJJGs2kuUcHvVkAPEXlg",
    "android.app.extra.PROVISIONING_ADMIN_EXTRAS_BUNDLE":{
       "com.google.android.apps.work.clouddpc.EXTRA_ENROLLMENT_TOKEN":"SELECTED ENROLMENT TOKEN"
    },
    "android.app.extra.PROVISIONING_WIFI_SSID":"corp-guest-wifi",
    "android.app.extra.PROVISIONING_WIFI_PASSWORD":"supersecurepassword",
    "android.app.extra.PROVISIONING_WIFI_SECURITY_TYPE":"WPA",
    "android.app.extra.PROVISIONING_WIFI_HIDDEN": false
 }
```   

## Creating a new QR Code
For completions sake, we should validate the JSON formatting using an [online tool](https://jsonlint.com/) before we use a [QR Code Generator](https://www.the-qrcode-generator.com/) to create our new QR code:

![Image](/img/android-custom-qr.png#center)


# Finishing it Up
This new QR Code can then be provided to your users, pending testing, to allow them to enrol their new Android device in Endpoint Manager when in range of your corporate guest wireless network.
