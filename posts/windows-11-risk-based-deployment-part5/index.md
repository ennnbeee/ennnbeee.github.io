# Risk Based Windows 11 Feature Update Deployment - Improvements


Why are we here again, looking at the deployment of Windows 11 to your Microsoft Intune managed devices?

Well, there's still [not long](https://windowscountdown.com/) to go until Windows 10 gets killed off and you're going to have to move to Windows 11, and ~four~ ~five~ ~six~ **nine** things:

1. We should probably be using [Entra App Registrations](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/walkthrough-register-app-azure-active-directory) for authentication.
2. [Alberto Baratta](https://www.linkedin.com/in/alberto-baratta-697078162) on [LinkedIn](<https://www.linkedin.com/feed/update/urn:li:activity:7178333408111038466/?commentUrn=urn%3Ali%3Acomment%3A(groupPost%3A8761296-7178333407398006785%2C7239494150788689920)&dashCommentUrn=urn%3Ali%3Afsd_comment%3A(7239494150788689920%2Curn%3Ali%3AgroupPost%3A8761296-7178333407398006785)>) kindly pointed out that there was a way to improve the collection of the Readiness Report data.
3. I've improved the processing of the Readiness report data after learning about [hashtables](https://learn.microsoft.com/en-us/powershell/scripting/learn/deep-dives/everything-about-hashtable?view=powershell-7.4).
4. I encountered a situation where existing Windows Update for Business rings were deployed to groups of **users** 😅.
5. If I'm saying you should be re-running the script on a regular basis, I should probably allow for a non-interactive version of the script.
6. I'd better include the option to use [Scope tags](https://learn.microsoft.com/en-us/mem/intune/fundamentals/scope-tags#to-create-a-scope-tag) in the report capture.
7. Device attributes should only be updated if they've changed.
8. Running the script in _whatif_ mode
9. Something about Windows 11 24H2 being released.

So how does that impact this series of posts and the updated [Windows 11 Accelerator script](https://github.com/ennnbeee/oddsandendpoints-scripts/blob/main/Intune/Reports/GetWindows11Ready/Invoke-Windows11AcceleratorUpdate.ps1)?

## Graph Authentication

Honestly I just straight up robbed the [authentication function](https://github.com/andrew-s-taylor/public/blob/main/Powershell%20Scripts/Intune/connect-tograph-function.ps1) that [Andrew Taylor](https://andrewstaylor.com/) created. Is it really stealing if it's community content? 😘

So to allow for the [updated script](https://github.com/ennnbeee/oddsandendpoints-scripts/blob/main/Intune/Reports/GetWindows11Ready/Invoke-Windows11AcceleratorUpdate.ps1) to authenticate to Graph. we now need to create an [Entra App registration](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app?tabs=client-secret#configure-platform-settings), associated [Client Secret](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app?tabs=client-secret), and give the [app permissions](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-configure-app-access-web-apis#application-permission-to-microsoft-graph) to Microsoft Graph.

{{< admonition type=info >}}
The permissions we need for this script are the following: `Group.ReadWrite.All`, `Device.ReadWrite.All`, `DeviceManagementManagedDevices.ReadWrite.All` `DeviceManagementConfiguration.ReadWrite.All`, `User.ReadWrite.All`, and `DeviceManagementRBAC.Read.All`
{{< /admonition >}}

![Windows 11 Accelerator App Reg](img/w11risk-appreg.png "App Registration for authentication in Entra ID.")

Once created, we can now pass through the Application Id and the value of the Client Secret to the script along with the already required tenant Id:

```PowerShell
$tenantId = '36019fe7-a342-4d98-9126-1b6f94904ac7'
$appId = '297b3303-da1a-4e58-bdd2-b8d681d1bd71'
$appSecret = 'g5m8Q~CSedPeRoee4Ld9Uvg2FhR_0Hy7kUpRIbo'

.\Invoke-Windows11AcceleratorUpdate.ps1 -tenantId $tenantId -appId $appId -appSecret $appSecret -target device -featureUpdateBuild 24H2 -extensionAttribute 11
```

Now giving us an easier way to authenticate to Graph, and will allow you to run the script without the need to authenticate with a Global Admin account every time.

Nice.

## Reporting Data Capture

For the improvement to the script performance, well this is where thinking about enterprises with 10,000's of Windows devices sometimes falls in a gap, when I've only got small dev environments to play with 😶.

Previously in the [script](https://github.com/ennnbeee/oddsandendpoints-scripts/blob/main/Intune/Reports/GetWindows11Ready/Invoke-Windows11Accelerator.ps1) and detailed in the below post...

{{< showcase title="Risk Based Windows 11 Feature Update Deployment - Reporting" summary="With Windows 10 support coming to an end sooner than you'd expect, in the first part of this series we look at ways to capture Feature Update Readiness Report data using PowerShell and Graph to help with the rollout of the new Windows 11 operating system." image="/posts/windows-11-risk-based-deployment-part1/featured.png" link="/posts/windows-11-risk-based-deployment-part1/" >}}

...we were calling the report 50 devices at a time, and looping through Graph API calls to the [deviceManagement/reports/cachedReportConfigurations](https://learn.microsoft.com/en-us/graph/api/intune-reporting-devicemanagementcachedreportconfiguration-get?view=graph-rest-beta) endpoint to get the [Feature Update Readiness report](https://learn.microsoft.com/en-us/mem/intune/protect/windows-update-compatibility-reports) data until there were no more devices left.

Which if you've got 80,000 Windows devices, is going to take quite a bit of time, hours in fact.

So the solution [Alberto Baratta](https://www.linkedin.com/in/alberto-baratta-697078162) came up with, was to just use the export report functionality that already exists in Intune, to capture the compressed download of the csv file, containing all the reporting information, using that as the source for the data.

Brilliant.

So now instead of looping through a million Graph calls, and probably hitting a throttle limit, we're just processing the exported file.

Comparing the previous version of the script taking **three minutes** to run (the video has been sped up x5) for **1100 devices**:

{{< video src="img/w11risk-graph" width="800px">}}

To the new version taking about **three seconds**:

{{< video src="img/w11risk-csv" width="800px">}}

We see a marked improvement on the time taken to gather the details of the [Windows feature update compatibility risks report](https://learn.microsoft.com/en-us/mem/intune/protect/windows-update-compatibility-reports), meaning that even with environments with 10,000's of devices, the process will only take seconds not minutes.

Speed nice.

## Hash Tables in PowerShell

Even with the improvement for capturing the readiness report details, when processing the data I was essentially `foreach` looping using `Where-Object` to get the **deviceObjectID** from the Entra data, which in large environments is not big, not clever, and importantly not quick.

So I found out about [hashtables](https://evotec.xyz/how-i-didnt-know-how-powerful-and-fast-hashtables-are/), and implemented them to vastly improve the processing of the report data.

Speedier nice.

## Tagging User Objects

I wasn't sure about the assignment of updates to users either, but the entire device estate only contained Single User Affinity Windows devices, i.e., one device to each user, and from an awareness and communication perspective, it's easy to tell your users when they're going to get updates for Feature Updates than it is working out which devices are.

So a quick amendment to the script, now allows us to get details of the users from Entra, and where the primary user of the device matches one of these Entra users, it allows us to tag the user account with the risk of updating their device to Windows 11.

Running the script with a new parameter, allows us to choose whether we want to assign the risk score to `user` or `device`:

```PowerShell
.\Invoke-Windows11AcceleratorUpdate.ps1 -target user -tenantId $tenantId -appId $appId -appSecret $appSecret -featureUpdateBuild 24H2 -extensionAttribute 11
```

With the script giving use the Dynamic Security Groups to create, as well as tagging primary users of the device:

{{< video src="img/w11risk-user" width="800px">}}

Nice?

{{< admonition type=info >}}
Do not use this unless you are happy that the Windows Update for Business, and Feature Update policies will follow the user around across devices they sign in to. So if you're using Shared Devices, stay away from this one as you can't use [Device Filters](https://learn.microsoft.com/en-us/mem/intune/fundamentals/filters) with Feature Update profiles.
{{< /admonition >}}

## Non-Interactive Script

So after you've run the script for the first time, you should be pretty happy that you've started tagging your devices or users, what about the subsequent runs of the script?

We should have already created our [dynamic device or user groups](https://learn.microsoft.com/en-us/entra/identity/users/groups-dynamic-membership) and confirmed that the extension attributes are not in use for anything but the readiness risk state.

So I've now included a `firstRun` boolean parameter, which if set to `$true` (and is by default), will prompt for interaction, otherwise if set to `$false` will just breeze through the script and update devices with their associated update readiness risk.

```PowerShell
.\Invoke-Windows11AcceleratorUpdate.ps1 -firstRun $false -tenantId $tenantId -target device -featureUpdateBuild 24H2 -extensionAttribute 11
```

Silent nice.

## Intune Scope Tags

I'd forgotten about these if I'm honest, so now I've included the option to select a [Scope Tag](https://learn.microsoft.com/en-us/mem/intune/fundamentals/scope-tags), so with the change to the authentication method also included a new Graph permission of `DeviceManagementRBAC.Read.All`, to allow us to read the existing Scope Tags and make sure the one selected exists.

```PowerShell
.\Invoke-Windows11AcceleratorUpdate.ps1 -scopeTag MVE -tenantId $tenantId -target device -featureUpdateBuild 24H2 -extensionAttribute 11
```

Don't say I didn't think about the more advanced users of Intune.

{{< admonition type=info >}}
If you don't specify a scopeTag, then `default` is used for the Readiness report.
{{< /admonition >}}

RBAC nice.

## Attribute Updates

I was a little lazy with the first iteration of the script, as I just leant on the fact that the call to Graph to add Device Attributes just overwrote them each time.

For more time based improvements, I've added in logic to only update the attributes that have changed.

![Updating Device Attributes](img/w11risk-daupdate.png "Device Attributes only being updated if they have changed.")

{{< admonition type=info >}}
To be honest this was only possible after the implementation of hashtables for speed.
{{< /admonition >}}

Sensible nice.

## Demo Mode

Just a quick one really, and for those who are more cautious than me, you can now run the script in demo mode, where it will do everything **except** tag users or devices.

```PowerShell
.\Invoke-Windows11AcceleratorUpdate.ps1  -demo -tenantId $tenantId -target device -featureUpdateBuild 24H2 -extensionAttribute 11
```

So this will just give you an indication of the current state of your Windows 11 Feature Update readiness, and make sure you're not butchering your environment with tags that don't mean anything.

![Windows 11 Accelerator script in Demo Mode](img/w11risk-demo.png "Windows 11 Accelerator script in Demo Mode.")

The script will still let you know what it is going to do, just not make changes to the extension attributes on devices or users.

Whatif nice.

## Windows 11 24H2

As the Windows 11 24H2 report data is now available, we should be able to query the data in Intune. So a quick update to the calls to Graph to use `GE24H2` for the correct Feature Update version (compared with `NI23H2` for Windows 11 23H2), and we can get the report data and tag things with their risk state:

```PowerShell
.\Invoke-Windows11AcceleratorUpdate.ps1 -featureUpdateBuild 24H2 -tenantId $tenantId -target device -extensionAttribute 11
```

![Windows 11 Accelerator script for 24H2](img/w11risk-24H2.png "Windows 11 Accelerator script for Windows 11 24H2")

Best nice.

## Summary

Yes I know I should have probably looked at these when I first released the series of blog posts, but I didn't. Be thankful I've revisited this series and not just thrown myself into Autopilot v2 or Windows 365 Link posts 😅.

Either way, you should now have no excuse to avoid moving to Windows 11 24H2 or otherwise, so crack on, start using the [Windows 11 Accelerator script](https://github.com/ennnbeee/oddsandendpoints-scripts/blob/main/Intune/Reports/GetWindows11Ready/Invoke-Windows11AcceleratorUpdate.ps1), time is ticking.

