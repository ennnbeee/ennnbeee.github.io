# A Flexible Approach to Microsoft Update Deployments


So this isn't the {{< reftab href="/posts/windows-update-phased-deployment" title="first time" >}} we've looked at improving the management of updates using Microsoft Intune, and probably won't be the last time either, especially with [declarative device management](https://learn.microsoft.com/en-us/mem/intune/protect/managed-software-updates-ios-macos) looming, for Apple and hopefully Windows devices, covering configuration of software updates.

But with the introduction of the [Windows Autopatch](https://learn.microsoft.com/en-us/windows/deployment/windows-autopatch/overview/windows-autopatch-overview) service, and getting hands on with it recently, it prompted me to revisit my previous approach to phased update delivery, so I thought I'd share my findings, this time across not just Windows Updates.

## Deployment Approach

Previously, we configured a set of {{< reftab href="/posts/windows-update-phased-deployment/#windows-update-for-business-rings" title="four Update Rings" >}}, but now we'll include an early adopters ring as a pre-production release, kind of a fail safe prior to your production deployment of updates; that final catch before you target the entire device estate:

- **Test** - This should be devices that are dedicated for testing, ~1% of your device estate.
- **Pilot** - This should be a stratified sample of either users or devices, ~5% of your device estate.
- **Pre-Production** -  The early adopters deployment, ~15% of your device estate.
- **Initial Production** - The first production deployment, ~30% of your device estate.
- **Final Production** - The final production deployment, the remaining ~50% of the device estate.

The aim is tht the above groupings are reusable and can be used across not just Windows Updates, but Office Updates, and Driver updates too. Let's look at the membership of these groups.

## Deployment Groups

Despite [Microsoft](https://techcommunity.microsoft.com/t5/intune-customer-success/support-tip-improving-the-efficiency-of-dynamic-group-processing/ba-p/4049394) recommending to stop using the `match` operators in favour of `startsWith` there are times when you might have to ignore them, this could be one of those times when we look at how we split a device estate into the phased breakdowns we're looking to achieve.

{{< admonition type=note >}}
After testing this in the real world, with reset Autopilot Hybrid join devices, you end up with a little bit of conflict on group membership due to there being two computer objects in Entra.

To avoid this, I've added in `(device.deviceManagementAppId -ne null)` to the group rules, only capturing those devices that are actually enrolled in Microsoft Intune.
{{< /admonition >}}

### Inefficient Dynamic Groups

For the members of Test and Pilot groups, these should be targeted members, not just any old device, so ensure you populate these groups with true test devices, and suitable pilot devices for update testing.

You'll notice that this time we're using Dynamic Groups across all the pre-production and production groups, and using a new attribute of `deviceId`, which is a wonderful UUID associated with *all* Entra computer objects, and it being a UUID gives us a nice split of devices based on the queries used.

| Group | Type | Membership |
| :- | :- | :- | :- |
| Test Group | Assigned | TBC |
| Pilot Group | Assigned | TBC |
| Pre-Production Group | Dynamic Device | `(device.deviceManagementAppId -ne null) and (device.deviceOSType -eq "Windows") and (device.deviceOwnership -eq "Company") and (device.deviceId -match "^[0-1,a]")` |
| Initial Production Group | Dynamic Device | `(device.deviceManagementAppId -ne null) and (device.deviceOSType -eq "Windows") and (device.deviceOwnership -eq "Company") and (device.deviceId -match "^[2-4,b-c]")` |
| Final Production Group | Dynamic Device | `(device.deviceManagementAppId -ne null) and (device.deviceOSType -eq "Windows") and (device.deviceOwnership -eq "Company") and (device.deviceId -match "^[5-9,d-f]")` |

With the start of the `deviceId` only ever being in the range of `0-9` or `a-f` we've an easy way to split our devices across the three production level groups, and ensure that we are capturing all devices, using the `match` operator, and the regular expression similar to `^[0-9,a-f]` to detect whether the `deviceId` starts with one of the values in the ranges provided.

{{< admonition type=info >}}
You may want to tweak the rules used here, to alter the size of each of the production groups, by changing the range of the `^[0-9,a-f]` query.
{{< /admonition >}}

### Efficient Dynamic Groups

If you want to please Microsoft Daddy, then you could use these alternative queries for the groups, Test and Pilot are still assigned, so don't forget to populate those members.

| Group | Type | Membership |
| :- | :- | :- | :- |
| Test Group | Assigned | TBC |
| Pilot Group | Assigned | TBC |
| Pre-Production Group | Dynamic Device | `(device.deviceManagementAppId -ne null) and (device.deviceOSType -eq "Windows") and (device.deviceOwnership -eq "Company") and ((device.deviceId -startsWith "0") or (device.deviceId -startsWith "1") or (device.deviceId -startsWith "a"))` |
| Initial Production Group | Dynamic Device | `(device.deviceManagementAppId -ne null) and (device.deviceOSType -eq "Windows") and (device.deviceOwnership -eq "Company") and ((device.deviceId -startsWith "2") or (device.deviceId -startsWith "3") or (device.deviceId -startsWith "4") or (device.deviceId -startsWith "b") or (device.deviceId -startsWith "c"))` |
| Final Production Group | Dynamic Device | `(device.deviceManagementAppId -ne null) and (device.deviceOSType -eq "Windows") and (device.deviceOwnership -eq "Company") and ((device.deviceId -startsWith "5") or (device.deviceId -startsWith "6") or (device.deviceId -startsWith "7") or (device.deviceId -startsWith "8") or (device.deviceId -startsWith "9") or (device.deviceId -startsWith "d") or (device.deviceId -startsWith "e") or (device.deviceId -startsWith "f"))` |

Much more efficient.

### VIP Groups

What about those people who complain about when they're getting updates, I mean you could tell them ~~to jog on~~ that you can't change the behaviour, or you could cater for the more VIP user, and allow for their device to sit in a different update group.

So more assigned groups are required for each potential exception or change to production level deployment options.

| Group | Type | Membership |
| :- | :- | :- |
| VIP Pre-Production Group | Assigned | TBC |
| VIP Initial Production Group | Assigned | TBC |
| VIP Final Production Group | Assigned | TBC |

See, we *can* please the senior members of the company with IT solutions.

## Windows Update Rings

Now with suitable groups at our disposal, we can create our Windows Update Rings in Microsoft Intune for each of the five phases, and after reviewing and experiencing Windows Update Rings personally instead of just recommending them, I've decided to set zero-day installation deadlines, in favour of longer grace periods, still working within the [National Cyber Security Centre](https://www.ncsc.gov.uk/collection/device-security-guidance/platform-guides/windows) 14-day window.

| Update Ring | Deferral | Deadline | Grace Period | Updates Installed |
| :- | :- | :- | :- | :- |
| Test | `0 days` | `0 days` | `1 day` | After 1 day, Wednesday |
| Pilot | `2 days` | `0 days` | `1 day` | After 3 days, Friday |
| Pre-Production | `5 days` | `0 days` | `2 days` | After 7 days, Tuesday |
| Initial Production | `8 days` | `0 days` | `2 days` | After 10 days, Friday |
| Final Production | `11 days` | `0 days` | `3 days` | After 14 days, Tuesday |

You see something else new? Yes, I've avoided device restarts on weekends following a chat with fellow lover of Microsoft Intune, [Jonathan Fallis](https://deploymentshare.com/), meaning that for a weekday based 'working week', updates get installed and devices restarted when the device is actually on, not over a weekend when realistically the device is still in the office and powered off, or being used to watch Netflix in bed.

This also allows for our VIP users to pick which day they actually want their device to restart, based on whatever day they're not playing golf 🏌️‍♀️.

## Office Update Rings

Oooooh this one is new, and straight up robbed from Windows Autopatch, and although doesn't exist as a dedicated blade in Microsoft Intune, we can use the glorious [Settings Catalog](https://learn.microsoft.com/en-us/mem/intune/configuration/settings-catalog) profiles to configure Office Update Channels, deferral, and deadline settings for Microsoft 365 App updates.

To start, we need to configure some baseline Office Update settings. So go create yourself a new Settings Catalog profile, and throw the below settings into it.

| Category | Setting | Value |
| :- | :- | :- |
| Microsoft Office 2016 (Machine) > Updates | Location for updates: (Device) | `http://officecdn.microsoft.com/pr/55336b82-a18d-4dd6-b5f6-9e5095c314a6` |
| Microsoft Office 2016 (Machine) > Updates | Enable Automatic Updates | `Enabled` |
| Microsoft Office 2016 (Machine) > Updates | Hide option to enable or disable updates | `Enabled` |
| Microsoft Office 2016 (Machine) > Updates | Hide Update Notifications | `Disabled` |
| Microsoft Office 2016 (Machine) > Updates | Update Channel | `Enabled` |
| Microsoft Office 2016 (Machine) > Updates | Channel Name (Device) | `Monthly Enterprise Channel` |
| Microsoft Office 2016 (Machine) > Updates | Update Path | `Enabled` |

With the update channel configured to Monthly Enterprise, and a bit of a change to the user experience, let's see what else we can borrow from [Windows Autopatch](https://learn.microsoft.com/en-us/windows/deployment/windows-autopatch/operate/windows-autopatch-microsoft-365-apps-enterprise).

{{< admonition type=info >}}
Make sure you assign this to all your devices in scope of updates, using either an existing group, or the built in 'All Devices' group and a suitable [Device Filter](https://learn.microsoft.com/en-us/mem/intune/fundamentals/filters).
{{< /admonition >}}

### Phased Office Updates

With a similar approach to the Windows Update Rings, we can create corresponding Settings Catalog profiles for our five deployment phases, aligning as best we can the end result, which is installed updates, to the restart of the updates delivered as part of the Windows Update ring configuration.

All settings exist under the same category as before, **Microsoft Office 2016 (Machine) > Updates**.

| Update Ring | Delay downloading and installing updates for Office | Days: (Device) | Update Deadline | Deadline: (Device) | Updates Installed |
| :- | :- | :- | :- | :- | :- |
| Test | `Enabled` | `0` | `Enabled` | `1` | After 1 day, Wednesday |
| Pilot | `Enabled` | `2` | `Enabled` | `1` | After 3 days, Friday |
| Pre-Production | `Enabled` | `5` | `Enabled` | `2` | After 7 days, Tuesday |
| Initial Production | `Enabled` | `8` | `Enabled` | `2` | After 10 days, Friday |
| Final Production | `Enabled` | `11` | `Enabled` | `3` | After 14 days, Tuesday |

As Office Updates are released with the same [update cadence](https://learn.microsoft.com/en-us/deployoffice/updates/overview-update-channels#comparison-of-the-update-channels-for-microsoft-365-apps) as our normal updates on a 'Patch Tuesday', then the installation time falls on the same days, meaning that **all** updates should be installed and the device restarted on the same day, reducing overall disruption to the users.

## Update Ring Assignment

After creating the Windows Update Rings, and our faux Office Update Rings using the above settings, we cover how we assign our phased deployment groups to each suitable Update Ring.

{{< admonition type=info >}}
Feel free to alter the deferral, deadline, and grace periods where applicable to your device estate, but please if you want the experience to be as expected, use the below assignment targets.
{{< /admonition >}}

| Update Ring | Included Groups | Excluded Groups |
| :- | :- | :- |
| Test | Test Group | - |
| Pilot | Pilot Group | Test Group |
| Pre-Production | Pre-Production Group | Test Group, Pilot Group |
| Initial Production | Initial Production Group | Test Group, Pilot Group |
| Final Production | Final Production Group | Test Group, Pilot Group |

I'm assuming here that you aren't putting VIP user devices in Test and Pilot, because you don't want to anger them, maybe do it on your last day .

{{< admonition type=info >}}
This is an improvement on our previous version of Update Ring assignment, as we're not relying on excludes and dynamic groups to update to ensure there are no conflicts.
{{< /admonition >}}

### VIP Ring Assignment

To get the expected behaviour for our VIP groups, we need to consider what happens when we want to ensure a VIP device is in the correct Update Ring, which includes making sure there are no conflicts across the rings.

Changing the assignments of our Update Rings to the below, will allow for the manual assignment based on a devices VIP group membership.

| Update Ring | Included Groups | Excluded Groups |
| :- | :- | :- |
| Test  | Test Group | - |
| Pilot | Pilot Group | Test Group |
| Pre-Production | Pre-Production Group, *VIP Pre-Production* | Test Group, Pilot Group, *VIP Initial Production*, *VIP Final Production* |
| Initial Production | Initial-Production Group, *VIP Initial-Production* | Test Group, Pilot Group, *VIP Pre-Production,* *VIP Final Production* |
| Final Production | Final Production, *VIP Final Production* | Test Group, Pilot Group, *VIP Pre-Production*, *VIP Initial Production* |

For example if device name `VIP-7wa7f4c35`, with deviceId `ec759183-4492-44f4-a2ec-dbc33470bf48` (which will captured by the 'Final Production' dynamic group), is added to group `VIP Initial Production`, it will exclude the device from the 'Final Production' Update Ring, and assign the Update Ring for 'Initial Production', with the device restarting after updates on a Friday.

Adding in not only new include assignments, but exclude assignments, will ensure that devices in these groups are assigned to the correct Update Ring.

## Summary

All of this should give you the option to not only suitably phase Windows, Microsoft, and Office Updates, with each subsequent Update Ring starting only after the next, but also allow the shift of devices around without conflict, whether this is for VIP users or otherwise.

It also provides your end users with clear details of the days of when their device will not only get updated, but also restart, without too much of a headache, or interruption to their casual personal use of their corporate owned device.

