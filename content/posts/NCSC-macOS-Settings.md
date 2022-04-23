---
title: "National Cyber Security Centre (NCSC): MacOS Security Settings"
date: 2022-04-21T11:55:11+01:00
draft: false
description: ""
tags: ["endpoint", "intune","ncsc","macos","security"]
ShowToc: true
cover:
    image: "/img/ncsc-macos.png" # image path/url
    alt: "macOS Security" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
![Image](/img/ncsc-macos.png#center)

If you've ever had to implement baseline security settings, whether this be Centre for Internet Security (CIS), Cyber Security Essentials (CSE), or National Cyber Security Centre (NCSC), you'll probably have encountered some level of pain when it comes to non-Microsoft devices, as the guidance, is, well, *complicated*.

NCSC do provide documented [guides](https://www.ncsc.gov.uk/collection/device-security-guidance/platform-guides/macos) for macOS along with a [GitHub repo](https://github.com/ukncsc/Device-Security-Guidance-Configuration-Packs/tree/main/Apple/macOS) containing a script and a list of recommended security settings, but what do these look like in Endpoint Manager? And what if you need to make changes? What about if you have exceptions?

# Configuration
I'm not going to detail all of these NCSC settings in this post, otherwise I'll do myself out of a job, but I can help with the security settings don't exist natively within the [macOS Device Restriction template](https://docs.microsoft.com/en-us/mem/intune/configuration/device-restrictions-macos).

## Custom Configuration
All the below security settings utilise the **Custom configuration template** for macOS...straight from the horses mouth:
> *"The custom configuration template allows IT admins to assign settings that aren't built into Intune yet. For macOS devices, you can import a .mobileconfig file that you created using Profile Manager or a different tool."*

Sounds perfect right? Details on how to create these profiles are below, but for now, let's look at the NCSC settings and their associated mobileconfig files.

### Automatic Updates
Now the NCSC guidelines advise that Operating System and Software updates should be applied automatically, with no deferal of when they are available or installed. Sadly, there isn't an option in Endpoint Manager to enforce this setting, but we can acheive this using a custom profile and generated mobileconfig file.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1">
  <dict>
    <key>PayloadUUID</key>
    <string>3C307E14-8CDD-4EB8-A696-AC967397CE40</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadOrganization</key>
    <string>ennbee.uk</string>
    <key>PayloadIdentifier</key>
    <string>0250B0DD-84C3-4DA4-91B3-6FA7824F54CD</string>
    <key>PayloadDisplayName</key>
    <string>NCSC Automatic Software Update Settings</string>
    <key>PayloadDescription</key>
    <string>Configures the macOS software update settings to enable all automatic update options.</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
    <key>PayloadEnabled</key>
    <true/>
    <key>PayloadRemovalDisallowed</key>
    <true/>
    <key>PayloadScope</key>
    <string>System</string>
    <key>PayloadContent</key>
    <array>
      <dict>
        <key>PayloadUUID</key>
        <string>A1936EA9-62CC-407B-94B0-1AD14BD513BE</string>
        <key>PayloadType</key>
        <string>com.apple.SoftwareUpdate</string>
        <key>PayloadOrganization</key>
        <string>ennbee.uk</string>
        <key>PayloadIdentifier</key>
        <string>com.apple.SoftwareUpdate.A1936EA9-62CC-407B-94B0-1AD14BD513BE</string>
        <key>PayloadDisplayName</key>
        <string>NCSC Automatic Software Update Settings</string>
        <key>PayloadDescription</key>
        <string>Configures the macOS software update settings to enable all automatic update options.<string/>
        <key>PayloadVersion</key>
        <integer>1</integer>
        <key>PayloadEnabled</key>
        <true/>
        <key>ConfigDataInstall</key>
        <true/>
        <key>CriticalUpdateInstall</key>
        <true/>
        <key>AutomaticCheckEnabled</key>
        <true/>
        <key>restrict-software-update-require-admin-to-install</key>
        <false/>
        <key>AutomaticDownload</key>
        <true/>
        <key>AutomaticallyInstallAppUpdates</key>
        <true/>
        <key>AutomaticallyInstallMacOSUpdates</key>
        <true/>
        <key>CatalogURL</key>
        </string>
        <key>AllowPreReleaseInstallation</key>
        <true/>
      </dict>
    </array>
  </dict>
</plist>
```
### Fast User Switching and Automatic Logon
Apparently Fast User Switching is a bad thing, as is automatically logging in, so they want this disabled too.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>PayloadContent</key>
	<array>
		<dict>
			<key>PayloadContent</key>
			<dict>
				<key>com.apple.loginwindow</key>
				<dict>
					<key>Forced</key>
					<array>
						<dict>
							<key>mcx_preference_settings</key>
							<dict>
								<key>DisableFDEAutoLogin</key>
								<true/>
							</dict>
						</dict>
					</array>
				</dict>
			</dict>
			<key>PayloadDescription</key>
			<string></string>
			<key>PayloadDisplayName</key>
			<string>NCSC Disable Fast User Switching and Autologon</string>
			<key>PayloadEnabled</key>
			<true/>
			<key>PayloadIdentifier</key>
			<string>7243A790-723E-4433-ABA4-629C1A99264C</string>
			<key>PayloadType</key>
			<string>com.apple.ManagedClient.preferences</string>
			<key>PayloadUUID</key>
			<string>7243A790-723E-4433-ABA4-629C1A99264C</string>
			<key>PayloadVersion</key>
			<integer>1</integer>
		</dict>
	</array>
	<key>PayloadDescription</key>
	<string></string>
	<key>PayloadDisplayName</key>
	<string>NCSC Disable Fast User Switching and Autologon</string>
	<key>PayloadEnabled</key>
	<true/>
	<key>PayloadIdentifier</key>
	<string>AF624C73-095F-4C69-89FB-A0A771D1225E</string>
	<key>PayloadRemovalDisallowed</key>
	<true/>
	<key>PayloadScope</key>
	<string>System</string>
	<key>PayloadType</key>
	<string>Configuration</string>
	<key>PayloadUUID</key>
	<string>AF624C73-095F-4C69-89FB-A0A771D1225E</string>
	<key>PayloadVersion</key>
	<integer>1</integer>
</dict>
</plist>
```
### Hiding System Preferences
This one makes more sense, as you don't want users meddling with certain System Preferences such as, iCloud, Sharing, Starup Disk and Security.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
   <key>PayloadContent</key>
   <array>
      <dict>
         <key>PayloadType</key>
         <string>com.apple.systempreferences</string>
         <key>PayloadVersion</key>
         <integer>1</integer>
         <key>PayloadIdentifier</key>
         <string>com.apple.systempreferences.12D5D87D-F18F-4807-84A8-1E77FBF0EA4B</string>
         <key>PayloadUUID</key>
         <string>12D5D87D-F18F-4807-84A8-1E77FBF0EA4B</string>
         <key>PayloadDisplayName</key>
         <string>com.apple.systempreferences</string>
         <key>PayloadDescription</key>
         <string>NCSC Hide System Preferences;iCloud, Security, Startup Disk and Sharing</string>
         <key>PayloadOrganization</key>
         <string>ennbee.uk</string>
         <key>PayloadEnabled</key>
         <true/>
         <key>HiddenPreferencePanes</key>
         <array>
            <string>com.apple.preferences.icloud</string>
            <string>com.apple.preference.security</string>
            <string>com.apple.preference.startupdisk</string>
            <string>com.apple.preferences.sharing</string>
         </array>
      </dict>
   </array>
   <key>PayloadDescription</key>
   <string>NCSC Hide System Preferences;iCloud, Security, Startup Disk and Sharing</string>
   <key>PayloadDisplayName</key>
   <string>NCSC Hide System Preferences</string>
   <key>PayloadIdentifier</key>
   <string>com.apple.systempreferences</string>
   <key>PayloadOrganization</key>
   <string>ennbee.uk</string>
   <key>PayloadUUID</key>
   <string>BE218051-ED0B-4555-ADFB-86AA3072BB35</string>
   <key>PayloadRemovalDisallowed</key>
   <true/>
   <key>PayloadType</key>
   <string>Configuration</string>
   <key>PayloadVersion</key>
   <integer>1</integer>
   <key>PayloadScope</key>
   <string>User</string>
</dict>
</plist>
```
### Time Servers
Not so sure on this one, must be down to ensuring that the system time is correct for validation of certificates...maybe.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>PayloadContent</key>
	<array>
		<dict>
			<key>PayloadContent</key>
			<dict>
				<key>com.apple.MCX</key>
				<dict>
					<key>Forced</key>
					<array>
						<dict>
							<key>mcx_preference_settings</key>
							<dict>
								<key>timeServer</key>
								<string>time.google.com,time.euro.apple.com</string>
							</dict>
						</dict>
					</array>
				</dict>
			</dict>
			<key>PayloadDescription</key>
			<string>Configures Time Servers</string>
			<key>PayloadDisplayName</key>
			<string>NCSC Automatic Time Servers</string>
			<key>PayloadEnabled</key>
			<true/>
			<key>PayloadIdentifier</key>
			<string>4B63BB1E-0D4D-4DD2-BF93-4AAB46523179</string>
			<key>PayloadOrganization</key>
			<string>ennbee.uk</string>
			<key>PayloadType</key>
			<string>com.apple.ManagedClient.preferences</string>
			<key>PayloadUUID</key>
			<string>4B63BB1E-0D4D-4DD2-BF93-4AAB46523179</string>
			<key>PayloadVersion</key>
			<integer>1</integer>
		</dict>
	</array>
	<key>PayloadDescription</key>
	<string>Configures Time Servers</string>
	<key>PayloadDisplayName</key>
	<string>NCSC Automatic Time Servers</string>
	<key>PayloadEnabled</key>
	<true/>
	<key>PayloadIdentifier</key>
	<string>EE4E0945-5074-4EF1-A375-2B9688644D8F</string>
	<key>PayloadOrganization</key>
	<string>ennbee.uk</string>
	<key>PayloadRemovalDisallowed</key>
	<true/>
	<key>PayloadScope</key>
	<string>System</string>
	<key>PayloadType</key>
	<string>Configuration</string>
	<key>PayloadUUID</key>
	<string>EE4E0945-5074-4EF1-A375-2B9688644D8F</string>
	<key>PayloadVersion</key>
	<integer>1</integer>
</dict>
</plist>
```

## Creating Custom Configuration Profiles
Now that we have the required mobileconfig files, and that you've saved them with that file extension (it's basically glorified XML to be honest), we can look at creating these Custom Configuration Profiles:

1. Browse to the [macOS Configuration page](https://endpoint.microsoft.com/#blade/Microsoft_Intune_DeviceSettings/DevicesMacOsMenu/configProfiles) and select 'Create profile'
![Image](/img/ncsc-macos-config.png#left)
2. Under 'Profile type' select 'Templates', then select 'Custom' and then select 'Create'
![Image](/img/ncsc-macos-custom.png#left)
3. Give the Custom profile a suitable name and select 'Next'
![Image](/img/ncsc-macos-name.png#left)
4. Give the Custom configuration profile name a suitable name, ensure the Deployment channel is 'Device channel' and upload the mobileconfig file.
![Image](/img/ncsc-macos-profile.png#left)
5. Select 'Next' and assign the profile to the required Device Group.
6. Repeat for all the mobileconfig files created.

# Summary
This was a bit of a whirlwind tour of Custom Configuration Profiles for macOS, and there is a huge amount that can be controlled, managed, and dictacted using them. If you have a macOS device, you can use Apple Configuration 2 to create these files, if you don't, then have fun with the [Apple Developer Device Management Guide](https://developer.apple.com/documentation/devicemanagement), like I did...