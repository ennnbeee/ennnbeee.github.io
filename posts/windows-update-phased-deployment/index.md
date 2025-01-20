# Intelligent Phased Windows Update for Business Deployments


You might have been asked the question, especially from organisations that currently utilise Microsoft Configuration Manager, about how you mimic existing [Device Collections](https://learn.microsoft.com/en-us/mem/configmgr/core/clients/manage/collections/create-collections) used for Software Update deployments in Microsoft Intune.

With Configuration Manager having the backing of Microsoft SQL, and a hardware inventory that collects every granular detail about Windows devices, splitting out your device estate into logical phases is very easy to achieve.

But what about Microsoft Intune, how do we phase these updates in a similar way to Configuration Manager?

## Windows Update for Business Rings

The first thing we need to understand is how many WUfB (Windows Update for Business) rings we should create, usually based on the size of your Windows device estate, for now we'll stick with four update rings:

- **Testing Ring**: This should be devices that are dedicated for testing, not IT machines. Roughly 5% of your device estate.
- **Pilot Ring**: This should be a stratified sample of either users or devices, not friends and family (HR and Finance). Roughly 15% of your device estate.
- **Initial Production Ring**: This should be roughly half of the remaining devices, roughly 30% of your device estate.
- **Final Production Ring**: This should cover all remaining devices, so 50% of the device estate.

These rings can be created in Microsoft Intune with differing deferral periods, installation deadlines, and if you like, grace periods.

### Installation Settings

Taking the Update Rings above, we should ensure that the end result of deferral, deadlines, and grace periods is that that there is no crossover between the timings. What I mean is that you don't want to discover an issue with devices in the Pilot Ring after the Initial Production Ring has already started deploying updates, this sounds like a nightmare.

So we can use settings similar to those in the below table to ensure that one phase doesn't start until the previous has completed.

| Update Ring | Deferral | Deadline | Grace Period | Updates Installed |
| :- | :- | :- | :- | :- |
| Testing Ring | `0 days` | `1 day` | `0 days` | `1 day` |
| Pilot Ring | `2 days` | `1 day` | `1 day` | `4 days` |
| Initial Production Ring | `6 days` | `2 days` | `1 day` | `9 days` |
| Final Production Ring | `10 days` | `2 days` | `1 day` | `13 days` |

{{< admonition type=info >}}
You may have noticed that we're working within a 14-day window here, because we all love {{< reftab href="/posts/macOS-NCSC-Revisited" title="NCSC Guidelines" >}}.
{{< /admonition >}}

So with the Update Rings configured, we now need to look at how these are assigned to Users and Devices.

### Users versus Devices Assignment

Now assignment of Update Rings is no different to any other assignment in Microsoft Intune, and by that I mean that you cannot mix assignment target identities within Include and Exclude targets. We do also need to consider not just the individual Update Ring assignment, but how they all interact together. What we don't want is a device meant to be in the Final Production ring, getting a 0-day deferral because the user is in the Testing Ring.

So we should stick with assigning these rings to a single identity, but which one?

I mean, the answer is Devices, but I'll explain why.

If you're using Users, which seems like the easier approach as finding and managing pilot users is pretty straight forward, it means that the Update Ring will follow them regardless of which device they login to. This is not the functionality we want, updates are delivered to devices, not users, so let's see how we can group devices together in a sensible way.

## Update Ring Groups

So now we need to find a way group devices in a sensible way, so let's create three (yes three) new ~~Azure AD~~ Microsoft Entra ID [Security Groups](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals/how-to-manage-groups) for each of the Update Rings.

- Windows Updates Testing
- Windows Updates Pilot
- Windows Updates Initial Production

With both the Testing and Pilot ring being specifically designated devices, we can use Assigned Groups and add Windows devices to each of those groups, what are we doing with the 'Windows Updates Initial Production' group though?

### Dynamic Groups

Now I know I love [Device Filters](https://learn.microsoft.com/en-us/mem/intune/fundamentals/filters), and do not fear, we'll need one of these later, but as covered {{< reftab href="/posts/endpoint-manager-device-filters" title="previously" >}} Dynamic Groups are more powerful, and for this use case will perfectly fit our need to split our Windows device estate in half-ish.

Device Name is how we're going to split our device estate, whether these are you old-school naming conventions on-premises based on asset tag, serial number or otherwise, Autopilot ~~Azure AD Join~~ MEID Join (*really?*) devices with a prefix and a serial, or god forbid Autopilot Hybrid Joined devices with a prefix and random numbers.

Either way, we should be able use Dynamic Device Groups to capture half of the device estate.

### Start of Device Name Matching

Before we look at how we capture the devices let's look at some device name examples starting with a list of devices with a mixture of device names.

```txt
ENB-BDZX4M3
ENB-3231EQSM
PGT-45E8LN1S
VIP-ec9d473986d9
ITS-CRJ7C39
CON-f6f64b04a9e7
```

As we have some kind of naming convention here with a three letter prefix follow by a hyphen and then some random characters, we should be able to capture half of the devices by pulling back only devices that after the hyphen start with odd numbers (1, 3, 5, 7, 9), or alternate letters (a, c, e, etc.).

Throwing open you're favourite [RegEx tester](https://regex101.com/) we can plonk in the device names and start playing with queries, and if we want to be exact so we only capture devices that match our three character prefix followed by a hyphen and an odd number, we can use an expression like this.

```txt
^[a-zA-Z]{3}-d*[13579]
```

For capturing devices that match the three character prefix followed by a hyphen and alternative lowercase characters, we can use the expression below.

```txt
^[a-zA-Z]{3}-d*[acegikmoqsuwy]
```

For capturing devices that match the three character prefix followed by a hyphen and alternative uppercase characters, we can use the expression below.

```txt
^[a-zA-Z]{3}-d*[ACEGIKMOQSUWY]
```

To give you an insight into the expression:

- `^`: Start of the string
- `[a-zA-Z]{3}-`: First three characters are either uppercase or lowercase letters followed by a hyphen
- `d*[13579]`: One odd number
- `d*[acegikmoqsuwy]`: One alternative lowercase letter from the supplied range
- `d*[ACEGIKMOQSUWY]`: One alternative uppercase letter from the supplied range

This should cover our requirements, we just need to create the Dynamic Group query.

### End of Device Name Matching

What happens if you're still using asset tag naming conventions, most of these devices will all start with the same letter or number, so we need a way to split these in a different way.

Looking at the sample device list below, we can create a new expression that looks at the last number of the device name, but taking into consideration the prefix, hyphen, and only the following six digits.

```txt
LTP-002003
LTP-002042
DTP-001191
DTP-000117
LTP-000008
```

Once again into [RegEx testing](https://regex101.com/) we can plonk in the device names and start playing with queries, this time we just want to match devices with any three character prefix followed by a hyphen, but end in an odd number.

```txt
^[a-zA-Z]{3}-\d{5}[13579]$
```

An overview of the expression:

- `^`: Start of the string
- `[a-zA-Z]{3}-`: First three characters are either uppercase or lowercase letters followed by a hyphen
- `\d{5}`: Matches five digits after the hyphen
- `[13579]$`: The sixth digit is odd

This should work for the sample asset tag named devices.

### Dynamic Group Query

As well as the using one of the above name matching expressions, we could do with capturing whether the device is actually a Windows Corporate-Owned device.

So let's build our [Dynamic Group Query](https://learn.microsoft.com/en-us/azure/active-directory/enterprise-users/groups-dynamic-membership) based on the below requirements.

- `(device.deviceOSType -eq "Windows")`: Any Windows Device
- `(device.deviceOSVersion -startsWith "10")`: Any device with an Operating System that starts with 10
- `(device.deviceOwnership -eq "Company")`: Any device that is corporate owned
- `((device.displayName -match "^[a-zA-Z]{3}-d*[13579]")`: Device name matches our naming convention followed by an odd number
- `(device.displayName -match "^[a-zA-Z]{3}-d*[acegikmoqsuwy]")`: Device name matches our naming convention followed by a lowercase letter in the set
- `(device.displayName -match "^[a-zA-Z]{3}-d*[ACEGIKMOQSUWY]"))`: Device name matches our naming convention followed by a uppercase letter in the set

We need to throw in some `and` and `or` into the mix to ensure we're only grabbing the correct devices, but the full query can be found below.

```PowerShell
(device.deviceOSType -eq "Windows") and (device.deviceOSVersion -startsWith "10") and (device.deviceOwnership -eq "Company") and ((device.displayName -match "^[a-zA-Z]{3}-d*[13579]") or (device.displayName -match "^[a-zA-Z]{3}-d*[acegikmoqsuwy]") or (device.displayName -match "^[a-zA-Z]{3}-d*[ACEGIKMOQSUWY]"))
```

Using our sample of devices names, the above query would capture the highlighted names below.

```txt {hl_lines=[2,4,5]}
ENB-BDZX4M3
ENB-3231EQSM
PGT-45E8LN1S
VIP-ec9d473986d9
ITS-CRJ7C39
CON-f6f64b04a9e7
```

Expanding this over an entire device estate should give us roughly half of all Windows 10 and later corporate owned devices.

## Update Ring Assignment

So have pretty much all we need to actually start assigning our Update Rings to device groups, but hang fire, we've only created three groups and have four Update Rings, what gives?

Well for the final ring, we need to cover all remaining Windows corporate owned devices, so we can use the built-in 'All Devices' group and a Device Filter that only pulls back corporate owned devices, so something like `(device.deviceOwnership -eq "Corporate")`.

Now we can start assigning, but we need to make sure that there is no conflict with the profiles, so we'll make use of both Include and Exclude assignment targets, detailed in the table below.

| Update Ring | Include Group | Filter | Mode | Exclude Groups |
| :--- | :--- | :--- | :--- | :--- |
| Testing Ring | `Windows Updates Testing` | n/a | n/a | n/a |
| Pilot Ring | `Windows Updates Pilot` | n/a | n/a | `Windows Updates Testing` |
| Initial Production Ring | `Windows Updates Initial Production` | n/a | n/a | `Windows Updates Testing`, `Windows Updates Pilot` |
| Final Production Ring | `All Devices` | `Corporate Windows` | `Include` | `Windows Updates Testing`, `Windows Updates Pilot`, `Windows Updates Initial Production` |

Once you've assigned the Update Rings and confirmed that devices are receiving the correct one, you're good to go. Simple really.

## Summary

There you have it, a phased and self maintaining deployment of Windows Updates across your device estate, allowing you to truly control when devices are getting updates and an ability to pause update rings in the event of an issue without taking out all of your device estate at once.

Obviously you can tailor the number of Update Rings and logic behind splitting devices across them, but it was hard enough for me to work out the RegEx for these two examples to be honest with you.

I'd suggest you start with understanding what devices you need to target, their naming conventions if any, and then play around with RegEx to ensure that you are capturing the correct devices, no one wants unexpected Windows Updates.

Once happy with your expressions, you can then create the Dynamic Group and review the membership before going all gung ho and straight up assigning the Update Ring profile to the group, you know, just in case.

