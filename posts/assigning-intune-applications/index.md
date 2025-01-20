# Assigning Intune Mobile Apps Quickly and Consistently


No one, and I mean no one, really wants to manually and individually [assign Mobile Apps](https://learn.microsoft.com/en-us/mem/intune/apps/apps-deploy) to Users or Devices in Intune, especially after you've happily used a script to {{< reftab href="/posts/approving-managed-google-play-apps" title="Approve Managed Google Play Apps" >}} in their 100's as part of migrating to Microsoft Intune from other ~~below par~~ Mobile Device Management solutions.

We could try and use [Policy Sets](https://learn.microsoft.com/en-us/mem/intune/fundamentals/policy-sets) here, but you know, they're in preview, and more importantly they have [issues](https://learn.microsoft.com/en-us/mem/intune/fundamentals/policy-sets#policy-sets-known-issues) like not supporting Managed Google Play or Apple VPP apps, so that's a no go.

So here we are again, throwing together functions, logic, and some kind of user interface, to allow us to assign these mobile apps in bulk with minimal headache and increase to any pre-existing repetitive strain injuries.

{{< admonition type=note >}}
The functions, authentication, and script have now been updated to support the use of the Graph PowerShell SDK.
{{< /admonition >}}

## Mobile App Assignment

First off, before we even start looking at how we bring this all together, is to understand how we can assign an App using Graph. So as we've done before, we could look under the hood and check the Browser {{< reftab href="/posts/apple-ade-profile-assignment/#developer-tool-sneaking" title="Developer Tools" >}}  to see what is actually going on, or we can just check the [Graph documentation](https://learn.microsoft.com/en-us/graph/api/intune-shared-mobileapp-assign?view=graph-rest-beta).

With this we now know that to use the `POST /deviceAppManagement/mobileApps/{mobileAppId}/assign` call, we'll need the `{mobileAppId}`, we're in luck, as we've already got a function {{< reftab href="/posts/creating-assigning-app-categories/#getting-mobile-apps" title="Get-MobileApps" >}} to do this for us.

### Assignment Types

Knowing the call to Graph is the easy bit, building the JSON data we're sending to it using the POST request is a different story, but at least we've got a framework to start us off, thanks [Microsoft](https://learn.microsoft.com/en-us/graph/api/intune-shared-mobileapp-assign?view=graph-rest-beta#request).

```JSON
{
  "mobileAppAssignments": [
    {
      "@odata.type": "#microsoft.graph.mobileAppAssignment",
      "id": "591620b7-20b7-5916-b720-1659b7201659",
      "intent": "required",
      "target": {
        "@odata.type": "microsoft.graph.deviceAndAppManagementAssignmentTarget"
      },
      "settings": {
        "@odata.type": "microsoft.graph.mobileAppAssignmentSettings"
      }
    }
  ]
}
```

We've built JSON data in PowerShell before, so this one isn't any different, we do however need to cater for Installation Intents, Assignment Groups, Device Filters, and Device Filter States. These can be broken down as below, with the ones we actually care about in **bold**.

| Item | Details|
| :- | :- |
| Installation Intents | **Required**, **Available for Enrolled Devices**, Available for Un-enrolled Devices, Uninstall |
| Assignment Groups | **All Users**, **All Devices**, **Groups** |
| Filters States | **Include**, **Exclude** |

We need to allow for the ability to assign Mobile Apps to all permutations of the above, whether this is a Required installation to All Devices, including a Device Filter, or an Available installation to a Group of Users. You get the picture.

### App Assignment JSON

We can now start to build the JSON data we can POST to Graph, but we do need a bit more information when it comes to the content of the JSON data, in particular what is acceptable in the `microsoft.graph.deviceAndAppManagementAssignmentTarget` as well as how we pass through a Device Filter as part of the assignment, what headings are needed depending on the Assignment Target as well as the Install Intent.

The below structure details the setup for the JSON:

- **mobileAppAssignments**
  - **odata.type**
    - #microsoft.graph.mobileAppAssignment: This will be the same for each request
  - **intent**
    - required: Requiring the App to be installed
    - available: Making the App available to be installed
    - uninstall: Uninstalling the App
  - **target**
    - **odata.type**
      - #microsoft.graph.groupAssignmentTarget: Assigns the App the a Group
      - groupId: If using the group assignment, the Id of the Group
      - #microsoft.graph.allLicensedUsersAssignmentTarget: Assigns the App the inbuilt All Users Group
      - #microsoft.graph.allDevicesAssignmentTarget: Assigns the App the inbuilt All Devices Group
    - **deviceAndAppManagementAssignmentFilterId**
      - id: Device Filter Id
    - **deviceAndAppManagementAssignmentFilterType**
      - include: Includes the devices in the filter
      - exclude: Excludes the devices in the filter

We need to build separate objects in PowerShell to complete the JSON hierarchy, and we can happily do this using PowerShell using something like the below:

Building the PSObject for `odata.type` and `intent`, with the App to be an Available Installation Intent:

```PowerShell
$Target = New-Object -TypeName PSObject
$Target | Add-Member -MemberType NoteProperty -Name '@odata.type' -Value '#microsoft.graph.mobileAppAssignment'
$Target | Add-Member -MemberType NoteProperty -Name 'intent' -Value 'available
```

If we're going for a Group based assignment including a Device Filter, we can build this and add to the existing `$Target` object using:

```PowerShell
$TargetGroup = New-Object -TypeName PSObject
$TargetGroup | Add-Member -MemberType NoteProperty -Name '@odata.type' -Value '#microsoft.graph.groupAssignmentTarget'
$TargetGroup | Add-Member -MemberType NoteProperty -Name 'groupId' -Value '{Group.Id}'
$TargetGroup | Add-Member -MemberType NoteProperty -Name 'deviceAndAppManagementAssignmentFilterId' -Value '{FilterId}'
$TargetGroup | Add-Member -MemberType NoteProperty -Name 'deviceAndAppManagementAssignmentFilterType' -Value 'include'
$Target | Add-Member -MemberType NoteProperty -Name 'target' -Value $TargetGroup
```

Then the final PSObject for `mobileAppAssignments`, adding everything together, and converting it to JSON:

```PowerShell
$Output = New-Object -TypeName PSObject
$Output | Add-Member -MemberType NoteProperty -Name 'mobileAppAssignments' -Value @($Target)

$JSON = $Output | ConvertTo-Json -Depth 3
```

Giving us the below output:

```JSON
{
  "mobileAppAssignments": [
    {
      "@odata.type": "#microsoft.graph.mobileAppAssignment",
      "intent": "available",
      "target": {
        "@odata.type": "#microsoft.graph.groupAssignmentTarget",
        "groupId": "{Group.Id}",
        "deviceAndAppManagementAssignmentFilterId": "{FilterId}",
        "deviceAndAppManagementAssignmentFilterType": "include"
      }
    }
  ]
}
```

This looks miraculously like the Microsoft example, but you know, same same but different. If it actually had the correct `{FilterId}` and `{GroupId}` in place, it would allow us to assign an App when pushed to Graph.

We're getting there, bear with me, I thought this post would be shorter.

## Function Building Blocks

There are a few components we need to put together in the form of functions to enable the assignment of Mobile Apps, if we're being complete about this, and for once we are (*ish*), we need to consider the following:

- Getting Mobile Apps
- Getting Device Filters
- Getting Groups
- Getting existing Assignments
- Dealing with existing Assignments

All of the above will alter the way we deal with the JSON data we are creating, so one for future me to deal with. For now, on to the functions.

Remember we'll need to authenticate to Graph using `Connect-MgGraph` for all of these.

### Getting Mobile Apps

This one has come straight from an existing script, but we'll need to filter the data gathered, otherwise we're dragging back every Mobile App, including those referenced in [App Protection Policies](https://learn.microsoft.com/en-us/mem/intune/apps/app-protection-policy), and we don't want to do anything with those today.

```PowerShell
Function Get-MobileApps() {
    [cmdletbinding()]
    $graphApiVersion = 'Beta'
    $Resource = 'deviceAppManagement/mobileApps'
    try {
        $uri = "https://graph.microsoft.com/$graphApiVersion/$($Resource)"
            (Invoke-MgGraphRequest -Uri $uri -Method Get).Value
    }
    catch {
        Write-Error $Error[0].ErrorDetails.Message
        break
    }
}
```

### Getting Device Filters

To fix our missing `{FilterId}` issue, we need to be able to capture this data. This is a quick one, as we just need a call to [`GET deviceManagement/assignmentFilters`](https://learn.microsoft.com/en-us/graph/api/intune-policyset-deviceandappmanagementassignmentfilter-list?view=graph-rest-beta) resource, once we've authenticated to Graph of course.

```PowerShell
Function Get-DeviceFilter() {

    $graphApiVersion = 'beta'
    $Resource = 'deviceManagement/assignmentFilters'

    try {
        $uri = "https://graph.microsoft.com/$graphApiVersion/$($Resource)"
            (Invoke-MgGraphRequest -Uri $uri -Method Get).Value
    }
    catch {
        Write-Error $Error[0].ErrorDetails.Message
        break
    }
}
```

This function will pull back all Device Filters in the Intune tenant, so we might need to filter this down later on.

### Getting Groups

Ah yes, Groups, I'd almost forgotten about these, but Dynamic Groups still have a place in Intune, and we need the `{GroupId}` to make this work, so we should find a way to grab these as well. As with the Device Filters, this is a call to Graph, but using the [`GET groups`](https://learn.microsoft.com/en-us/graph/api/group-list?view=graph-rest-1.0&tabs=http) resource to capture the group `id` we need.

It was supposed to be so easy, however as we need to limit the number of Groups being pulled back, we need a way to search for a Group name. Here comes the [Search Query Parameter](https://learn.microsoft.com/en-us/graph/search-query-parameter?tabs=http), and importantly the need for `ConsistencyLevel` in the headers in the connection `Invoke-MgGraphRequest -Headers @{ConsistencyLevel = 'eventual' }`.

```txt
Name                           Value
----                           -----
Content-Type                   application/json
ConsistencyLevel               eventual
ExpiresOn                      09/02/2023 11:49:47 +00:00
Authorization                  Bearer eyJ0eXAiOiJKV1QiLCJub25jZSI6ImFVUTNp...
```

So eventually we have the `Get-MDMGroup` function, which will require a `$GroupName` parameter to search for the Groups to retrieve.

```PowerShell
Function Get-MDMGroup() {

    [cmdletbinding()]

    param
    (
        [parameter(Mandatory = $true)]
        [string]$GroupName
    )

    $graphApiVersion = 'beta'
    $Resource = 'groups'

    try {
        $authToken['ConsistencyLevel'] = 'eventual'
        $searchterm = 'search="displayName:' + $GroupName + '"'
        $uri = "https://graph.microsoft.com/$graphApiVersion/$Resource`?$searchterm"
        (Invoke-MgGraphRequest -Uri $uri -Method Get).Value
    }
    catch {
        Write-Error $Error[0].ErrorDetails.Message
        break
    }
}
```

Honestly, after that I wish I'd excluded Group based assignments, and we should have used one of the many PowerShell modules to pull back groups, but I like Graph, I like a consistent approach, and I'm using PowerShell Core 7.

### Getting Existing Assignments

We could be brutal with this script, and strip out the existing App assignments when assigning new ones, which is the default behaviour with `POST`, but I'm pretty sure that would make for some unhappy users. I couldn't see a `PATCH` option for assignments, so we are going to have to write something ourselves to sort this at some point.

Here is the function we can use to pull back the existing assignment data, I honestly can't remember why I've done it this way, I think it was due to the lovely format in which the assignments were displayed, but here we only need an App `$Id` to make this one work.

```PowerShell
Function Get-ApplicationAssignment() {

    [cmdletbinding()]

    param
    (
        [parameter(Mandatory = $true)]
        $Id
    )

    $graphApiVersion = 'Beta'
    $Resource = "deviceAppManagement/mobileApps/$Id/?`$expand=categories,assignments"

    try {
        $uri = "https://graph.microsoft.com/$graphApiVersion/$($Resource)"
        (Invoke-MgGraphRequest -Uri $uri -Method Get)
    }
    catch {
        Write-Error $Error[0].ErrorDetails.Message
        break
    }
}
```

### Removing Existing Assignments

Even if we're not accidentally removing existing assignments, we should give ourselves the option to undo any mistakes that we make when assigning Mobile Apps, so a quick tour of Graph gives us the [`DELETE mobileAppAssignment`](https://learn.microsoft.com/en-us/graph/api/intune-apps-mobileappassignment-delete?view=graph-rest-1.0) resource, and with a couple of parameters needed for both the App `$Id` and the Assignment `$AssignmentId`, we've got ourselves a function to get us out of trouble if we need it.

```PowerShell
Function Remove-ApplicationAssignment() {

    [cmdletbinding()]

    param
    (
        [parameter(Mandatory = $true)]
        $Id,
        [parameter(Mandatory = $true)]
        $AssignmentId
    )

    $graphApiVersion = 'Beta'
    $Resource = "deviceAppManagement/mobileApps/$Id/assignments/$AssignmentId"

    try {
        $uri = "https://graph.microsoft.com/$graphApiVersion/$($Resource)"
        (Invoke-MgGraphRequest -Uri $uri -Method Delete)
    }
    catch {
        Write-Error $Error[0].ErrorDetails.Message
        break
    }
}
```

## App Assignment Function

Finally we have all the parts to build the function, using the parameters below we can pass through the information needed to build the JSON data, pass through options as to whether we're adding or replacing the assignment, and using some level of logic make sure we're not messing this data up along the way.

### Parameters

I've niced myself here and given options for the application of the function, covering all our requirements for assignment. A break down of the parameters can be found below:

| Parameter | Description | Required | Data |
| :- | :- | :- | :- |
| `Id` | The Id of the Mobile App, obtained via `Get-MobileApps` | True | String |
| `TargetGroupId` | The Id of a Group if assigning to a Group obtained via `Get-MDMGroup` | False | String |
| `InstallIntent` | Whether the Assignment is set as Available or Required | True | Available/Required |
| `FilterID` | The ID of a Device Filter if assigning using Device Filters obtained via `Get-DeviceFilter` | False | String |
| `FilterMode` | The Filter mode if assigning using Device Filters | False | Include/Exclude |
| `All` | Used if assigning to the in-built 'All Users' or 'All Devices' groups | False | Devices/Users |
| `Action` | Whether to Add to existing Assignments, or to Replace them | True | Add/Replace |

### Add Application Assignment

Here is the function in all it's janky glory, ready to be used and abused by whatever I call PowerShell and acceptable user interfaces. We've got the ability to capture existing assignments, as well as logic on duplicate assignment methods, and of course adding new ones to the Mobile App in question.

```PowerShell
Function Add-ApplicationAssignment() {

    [cmdletbinding()]

    param
    (
        [parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        $Id,

        [parameter(Mandatory = $false)]
        [ValidateNotNullOrEmpty()]
        $TargetGroupId,

        [parameter(Mandatory = $true)]
        [ValidateSet('Available', 'Required')]
        [ValidateNotNullOrEmpty()]
        $InstallIntent,

        $FilterID,

        [ValidateSet('Include', 'Exclude')]
        $FilterMode,

        [parameter(Mandatory = $false)]
        [ValidateSet('Users', 'Devices')]
        [ValidateNotNullOrEmpty()]
        $All,

        [parameter(Mandatory = $true)]
        [ValidateSet('Replace', 'Add')]
        $Action
    )

    $graphApiVersion = 'beta'
    $Resource = "deviceAppManagement/mobileApps/$Id/assign"

    try {
        $TargetGroups = @()

        If ($Action -eq 'Add') {
            # Checking if there are Assignments already configured
            $Assignments = (Get-ApplicationAssignment -Id $Id).assignments
            if (@($Assignments).count -ge 1) {
                foreach ($Assignment in $Assignments) {

                    If (($null -ne $TargetGroupId) -and ($TargetGroupId -eq $Assignment.target.groupId)) {
                        Write-Host 'The App is already assigned to the Group' -ForegroundColor Yellow
                    }
                    ElseIf (($All -eq 'Devices') -and ($Assignment.target.'@odata.type' -eq '#microsoft.graph.allDevicesAssignmentTarget')) {
                        Write-Host 'The App is already assigned to the All Devices Group' -ForegroundColor Yellow
                    }
                    ElseIf (($All -eq 'Users') -and ($Assignment.target.'@odata.type' -eq '#microsoft.graph.allLicensedUsersAssignmentTarget')) {
                        Write-Host 'The App is already assigned to the All Users Group' -ForegroundColor Yellow
                    }
                    Else {
                        $TargetGroup = New-Object -TypeName psobject

                        if (($Assignment.target).'@odata.type' -eq '#microsoft.graph.groupAssignmentTarget') {
                            $TargetGroup | Add-Member -MemberType NoteProperty -Name '@odata.type' -Value '#microsoft.graph.groupAssignmentTarget'
                            $TargetGroup | Add-Member -MemberType NoteProperty -Name 'groupId' -Value $Assignment.target.groupId
                        }

                        elseif (($Assignment.target).'@odata.type' -eq '#microsoft.graph.allLicensedUsersAssignmentTarget') {
                            $TargetGroup | Add-Member -MemberType NoteProperty -Name '@odata.type' -Value '#microsoft.graph.allLicensedUsersAssignmentTarget'
                        }
                        elseif (($Assignment.target).'@odata.type' -eq '#microsoft.graph.allDevicesAssignmentTarget') {
                            $TargetGroup | Add-Member -MemberType NoteProperty -Name '@odata.type' -Value '#microsoft.graph.allDevicesAssignmentTarget'
                        }

                        if ($Assignment.target.deviceAndAppManagementAssignmentFilterType -ne 'none') {

                            $TargetGroup | Add-Member -MemberType NoteProperty -Name 'deviceAndAppManagementAssignmentFilterId' -Value $Assignment.target.deviceAndAppManagementAssignmentFilterId
                            $TargetGroup | Add-Member -MemberType NoteProperty -Name 'deviceAndAppManagementAssignmentFilterType' -Value $Assignment.target.deviceAndAppManagementAssignmentFilterType
                        }

                        $Target = New-Object -TypeName psobject
                        $Target | Add-Member -MemberType NoteProperty -Name '@odata.type' -Value '#microsoft.graph.mobileAppAssignment'
                        $Target | Add-Member -MemberType NoteProperty -Name 'intent' -Value $Assignment.intent
                        $Target | Add-Member -MemberType NoteProperty -Name 'target' -Value $TargetGroup
                        $TargetGroups += $Target
                    }
                }
            }
        }

        $Target = New-Object -TypeName psobject
        $Target | Add-Member -MemberType NoteProperty -Name '@odata.type' -Value '#microsoft.graph.mobileAppAssignment'
        $Target | Add-Member -MemberType NoteProperty -Name 'intent' -Value $InstallIntent

        $TargetGroup = New-Object -TypeName psobject
        if ($TargetGroupId) {
            $TargetGroup | Add-Member -MemberType NoteProperty -Name '@odata.type' -Value '#microsoft.graph.groupAssignmentTarget'
            $TargetGroup | Add-Member -MemberType NoteProperty -Name 'groupId' -Value $TargetGroupId
        }
        else {
            if ($All -eq 'Users') {
                $TargetGroup | Add-Member -MemberType NoteProperty -Name '@odata.type' -Value '#microsoft.graph.allLicensedUsersAssignmentTarget'
            }
            ElseIf ($All -eq 'Devices') {
                $TargetGroup | Add-Member -MemberType NoteProperty -Name '@odata.type' -Value '#microsoft.graph.allDevicesAssignmentTarget'
            }
        }

        if ($FilterMode) {
            $TargetGroup | Add-Member -MemberType NoteProperty -Name 'deviceAndAppManagementAssignmentFilterId' -Value $FilterID
            $TargetGroup | Add-Member -MemberType NoteProperty -Name 'deviceAndAppManagementAssignmentFilterType' -Value $FilterMode
        }

        $Target | Add-Member -MemberType NoteProperty -Name 'target' -Value $TargetGroup
        $TargetGroups += $Target
        $Output = New-Object -TypeName psobject
        $Output | Add-Member -MemberType NoteProperty -Name 'mobileAppAssignments' -Value @($TargetGroups)

        $JSON = $Output | ConvertTo-Json -Depth 3

        $uri = "https://graph.microsoft.com/$graphApiVersion/$($Resource)"
        Invoke-MgGraphRequest -Uri $uri -Method Post -Body $JSON -ContentType 'application/json'
    }
    catch {
        Write-Error $Error[0].ErrorDetails.Message
        break
    }
}
```

I can't say I'm proud of this function, but it does cater for whatever I've thrown at it.

### Function Actions

Running this function allows us to finally assign a Mobile App, I'm starting to think I could have just manually assigned the Apps at this point, but I'm confident I'll have to use this script again and again in the future.

- Replacing Assignment for an App, with the intent as **Required** to the **'All Devices'** group, with an **include Device Filter** :

```PowerShell
Add-ApplicationAssignment -Id {AppId} -InstallIntent Required -All All Devices -FilterMode Include -FilterID {FilterId} -Action Replace
```

- Adding an Assignment for an App, with an intent as **Available** to a **Group** with **no Device Filter** would looks something like this:

```PowerShell
Add-ApplicationAssignment -Id {AppId} -InstallIntent Available -TargetGroupId {Group.Id} -Action Add
```

- Adding an Assignment for an App, with the intent as  **Required** to a **Group** with an **exclude Device Filter** would looks something like this:

```PowerShell
Add-ApplicationAssignment -Id {AppId} -InstallIntent Available -TargetGroupId {Group.Id} -FilterMode Exclude -FilterID {FilterId} -Action Add
```

I think we're done here.

### Running, Assigning and Hiding

I've included a whole lot of logic and user interface when it comes to this script, and as most of my time has been spent battling with existing assignments, new functions and the headache that is building JSON data from the existing assignment content, I'm not going to go into any detail about this part of the [script](https://github.com/ennnbeee/oddsandendpoints-scripts/blob/main/Intune/Apps/AssignApps/Invoke-MgAssignIntuneApps.ps1) but you can as always find it in GitHub if you want to have a gander at what consists as good practice in my mind.

### The Payoff

It's got to the point now where I honestly couldn't be bothered to take screenshots of the script in action, plus it would make this already lengthy post even longer, so through the wonder that is screen recording and YouTube you can see the script in action demonstrating the following:

- Adding new Assignments for multiple Apps, with install intent as Required to the All Devices group with a Device Filter
- Adding new Assignments for multiple Apps, with install intent as Available to a Group with a Device Filter
- Replacing Assignments for multiple Apps, with install intent as Available to the All Users group with a Device Filter
- Removing all Assignments for multiple Apps
- Adding new Assignments for multiple Apps, with install intent as Required to the All Devices group with a Device Filter
- Adding Assignments for multiple Apps, with install intent as Available to the All Users group with a Device Filter

I haven't recorded every possible option, as I have coffee to drink and other awful PowerShell to write, but you get the gist.

{{< youtube 36wdQ2JTXAE >}}

## Summary

This felt like a long journey, even with some of the existing foundation functions already at our fingertips, but a worthy one as I'm never going to have to individually assign loads of Mobile Apps again, and thanks to this effort, nor do you.

As much as there is some intelligence to the script you will still have to use your own knowledge of Intune for when is best to use Devices, Users and Groups, but we can't solve all our problems with PowerShell, trust me, I'm trying. The script however does give you an easy way to bulk assign Mobile Apps in Intune, fully strip out assignments, or even just make every Mobile App available to everyone all at once, without too much clicking about.

There is  definitely room for improvement and expansion, and at some point I'll extend this to cover the assignment of Windows and macOS Apps; but for now you'll have to make do with Android and iOS.

As always, please test this one before going large and selecting all your Mobile Apps, I've included warning break points for a reason.

