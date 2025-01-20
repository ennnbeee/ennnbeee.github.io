# Microsoft Intune and the Curious Case of the Converting Firewall Rule Policy


What do you do when someone reaches out on [GitHub](https://github.com/ennnbeee/oddsandendpoints-scripts/issues/1) about one of your scripts no longer working as expected?

Well it's obvious that you just assume that they're doing something wrong...

{{< figure src="img/33bpFN25l6qNW.webp" alt="ridiculous giphy 1" nozoom=true >}}

...what if they're not wrong though?

What if the script you created to convert the old style of Firewall Rule policies created using the now defunct [migration tool](https://learn.microsoft.com/en-us/mem/intune/protect/endpoint-security-firewall-rule-tool) doesn't work because the rules it created are now [Settings Catalog](https://learn.microsoft.com/en-us/mem/intune/protect/endpoint-security-firewall-policy) policies, ruining not just a good PowerShell script, but a blog post whilst it's at it?

{{< showcase title="Modernising Microsoft Intune Firewall Rule Policies" summary="If you've ever experienced the joys of migrating Group Policy and in particular Windows Defender Firewall rules away from Group Policy to Microsoft Intune, you've probably encountered the Rule Migration Tool, and for now this tool has worked well. So what's the catch?" image="/posts/settings-catalog-firewall-rules/featured.png" link="/posts/settings-catalog-firewall-rules/" >}}

Well you write a blog post about it...duh.

{{< figure src="img/duh.png" alt="duhhhhhh" nozoom=true >}}

## The Tool

First off, let's look at the [Migration Tool](https://learn.microsoft.com/en-us/mem/intune/protect/endpoint-security-firewall-rule-tool) which I'm pretty sure we've all used in our time allowing the automatic creation of Firewall Rule policies, taking existing Group Policy applied rules from a target device and punting them straight into Microsoft Intune.

Oh it's gone.

{{< figure src="img/migtool.png" alt="it gone" nozoom=true >}}

Well Microsoft you are correct, the old version of the tool *did* create the old **mdm** Endpoint Security Profiles, and it *did* use the old authentication method, but it also *did* work.

If you're ~~sensible~~ nice enough to follow me on LinkedIn you might have spotted that as a favour to a stranger (not the first time mind you), I updated the migration tool to support the new authentication method, and a couple of other bits as well.

{{< figure src="img/GSE1BzJG4JVbq.webp" alt="ridiculous giphy 2" nozoom=true >}}

## The Script

I'm not using this post to break down exactly *how* I ~~fixed~~ updated the [PowerShell script](https://github.com/ennnbeee/oddsandendpoints-scripts/tree/main/Intune/EndpointSecurity/FirewallRuleMigration) in the short term, but in summary I...

- Removed the reliance on the Microsoft GitHub repo to download content.
- Changed authentication to use the [Microsoft.Graph](https://learn.microsoft.com/en-us/powershell/microsoftgraph/get-started?view=graph-powershell-1.0) PowerShell module.
- Changed the web requests to use [Invoke-MgGraphRequest](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.authentication/invoke-mggraphrequest?view=graph-powershell-1.0) for calls to Graph.
- Forced the PowerShell script to create Endpoint Security templates for firewall rule policies instead of Configuration Profiles.
- Changed the Authentication to Graph to use [device authentication](https://learn.microsoft.com/en-us/powershell/microsoftgraph/authentication-commands?view=graph-powershell-1.0#delegated-access) as I'm a dirty macOS user.
- Disabled sending any telemetry data on success and failure of rule creation to Microsoft, as they have too much data on me already.
- Fixed an issue when checking for profile name matching when there are no existing firewall rule policies in Microsoft Intune.

{{< figure src="img/QJfu2a1r1Ks6OaL7WV.webp" alt="ridiculous giphy 3" nozoom=true >}}

To use the script itself, you:

- Download the [FirewallRuleMigration.zip](https://github.com/ennnbeee/oddsandendpoints-scripts/blob/main/Intune/EndpointSecurity/FirewallRuleMigration/FirewallRuleMigration.zip) file and extract on your Windows device.
- Open PowerShell as Administrator.
- Navigate to the extracted folder, your PowerShell prompt should be in the **FirewallRuleMigration** folder.
- Run `Set-ExecutionPolicy Bypass` accepting all prompts like a good little child.
- Run `./Export-FirewallRules.ps1` with the corresponding switches (`includeDisabledRules`, `includeLocalRules`) if required.
- Authenticate to Graph using a Global Admin account, twice*.
- Enter a new profile name for the Firewall rules policy when prompted.
- Wait for rules to be uploaded to Microsoft Intune.

> *The script will disconnect all existing Graph sessions, and connect twice; once to allow for consent to be provided, the following to allow the script to run following the consent request.

So with this new script in hand, you can still migrate your Group Policy applied Firewall rules to Microsoft Intune, nice eh?

## Firewall Rules 2: The Conversioning

What I did not expect when running the PowerShell script, and after it created the Firewall Rule policies in Microsoft Intune, were them to just *change*.

What I expected, is that I'd have to use my old script to actually convert them from the old versions...

![MDM Firewall Rules](img/mdmrules.png "Firewall Rule Policies created using the Migration Tool.")

...and not just wait ten minutes for them to transition to new rules.

![MDM Firewall Rules](img/senserules.png "Firewall Rule Policies converted using Microsoft secret sauce.")

So wtf is happening?

No clue.

{{< figure src="img/37smQpbdHrBjVsxEhM.webp" alt="ridiculous giphy 89" nozoom=true >}}

## Are Microsoft Magicians?

Yes.

## The Truthening

The original Firewall Rule policies created by my wonderful PowerShell script are still using the legacy templates in Microsoft Intune:

![MDM Firewall Rules](img/ruledetail.png "Firewall Rule Policies the old worse way of dealing with them.")

And to prove I didn't spend my evening somehow manually creating 150 firewall rules in a way you can no longer create them, you can see the [audit log](https://learn.microsoft.com/en-us/mem/intune/fundamentals/monitor-audit-logs) showing that the policies were created at the timestamp:

![Definitely real audit log](img/audit.png "Definitely real and not made up Microsoft Intune audit log.")

But oh what have we found here with our super secret snooping skills?

Clicking on one of the items that were created by the PowerShell script, we get a bit more detail:

![Still real audit log](img/migrating.png "Still a real and not made up Microsoft Intune audit log but with more detail.")

But what's this at the bottom of the entry:

![Still real audit log](img/zoom.png "OMG it's migrating.")

{{< figure src="img/doge.webp" alt="rip doge" nozoom=true >}}

![Still real audit log](img/zoomier.png "Zoom and enhance.")

Yeah this means nothing.

The audit log is showing that a new legacy Firewall Rule policy is being created, how do we know? Well it's still using the old Endpoint Security templateId value of **4356d05c-a4ab-4a07-9ece-739f7c792910**.

![Endpoint Security templateId](img/templateId.png "Another screenshot of some audit stuff.")

Running a quick [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer) **GET** request to:

```txt
https://graph.microsoft.com/beta/deviceManagement/templates?`$filter=(isof(%27microsoft.graph.securityBaselineTemplate%27))
```

and you'll see all the Endpoint Security Templates, and specifically this one:

```JSON
{
    "@odata.type": "#microsoft.graph.securityBaselineTemplate",
    "id": "4356d05c-a4ab-4a07-9ece-739f7c792910",
    "displayName": "Windows Firewall rules",
    "description": "Windows Firewall allows administrators to define granular Firewall rules. Define firewall rules with specific ports, protocols, applications and networks, to allow or block network traffic.",
    "versionInfo": "2006",
    "isDeprecated": false,
    "intentCount": 2,
    "templateType": "securityTemplate",
    "platformType": "windows10AndLater",
    "templateSubtype": "firewall",
    "publishedDateTime": "2020-03-11T00:00:00Z"
}
```

Either way Microsoft is doing some wizardry behind the scenes not available to see with the naked eye.

{{< figure src="img/Sf0lxerEx2eNG.webp" alt="sack magique" nozoom=true >}}

So go check your own Firewall Rule policies, I bet they'll be the new Settings Catalog ones. I'll wait (hint, I won't).

{{< figure src="img/SWVzkIlHdEckF81gnA.webp" alt="pretty sure he's used this one himself" nozoom=true >}}

Or if you like, use the [PowerShell script](https://github.com/ennnbeee/oddsandendpoints-scripts/tree/main/Intune/EndpointSecurity/FirewallRuleMigration) to create some new old ones, then just wait a little bit and see them evolve into Settings Catalog policies.

Who's that ~~Pokémon~~ Firewall Rule policy?

{{< figure src="img/pokemon.png" alt="who's that pokemon" nozoom=true >}}

It's a Settings Catalog policy for Firewall Rules!

![MDM Firewall Rules](img/newruledetail.png "Firewall Rule Policies the new much better way of dealing with them.")

## Summary

Honestly, not sure what you want from me here; there might have been a communication about this in the [What's new in Intune](https://learn.microsoft.com/en-us/mem/intune/fundamentals/whats-new), maybe a blog post, maybe not.

Either way you've got a new version of the [Migration Tool script](https://github.com/ennnbeee/oddsandendpoints-scripts/tree/main/Intune/EndpointSecurity/FirewallRuleMigration), a bit of background on how it works, and a dive into the world of things that just happen without us knowing in Microsoft Intune land.

Off you jog now.

