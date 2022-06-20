---
title: "Windows Autopilot: Retrofitting Autopilot Group Tags"
date: 2022-06-19T15:55:02+01:00
draft: false
description: ""
tags: ["endpoint", "intune", "autopilot","windows","powershell", "graph"]
categories: ["windows", "autopilot", "administration"]
ShowToc: true
cover:
    image: "/img/autopilot-bulk-tag.png" # image path/url
    alt: "" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---

Now I don't think I promised that I'd cover off bulk tagging Autopilot devices in a [previous post](https://memv.ennbee.uk/posts/autopilot-power-of-group-tags/), but you know, I was running low on things to write about. So here we are.

As I like to practice what I preach, I'd left myself the task of updating 1000's of Autopilot devices with a new [Group Tag](https://techcommunity.microsoft.com/t5/intune-customer-success/support-tip-using-group-tags-to-import-devices-into-intune-with/ba-p/815336) after a successful Proof-of-Concept implementation of a suitable convention and syntax. Thanks past me.

So what does any good consultant do? ~~Run away?~~ ~~Come up with a janky script that you'll only ever run once, that contains nested 'foreach' loops?~~ Write a perfectly reusable and digestible PowerShell script using Graph API to update existing Autopilot devices...

# The Approach
Now I needed an easy way to tag specific devices with Group Tags, and then anything that didn't have a specific Group Tag to get a default one...so we're working with two distinct use cases here.

# Ready, Aim
For the first, we'll start with exporting the Autopilot Devices from the tenant, as this CSV file will contain the Serial Numbers we need further down the line.

![Image](/img/autopilot-bulk-tag-export.png#left)

Now with the CSV file, we really only need two headings, Serial Number and Group Tag. So open up your favourite editor and bin off every other heading. 

Whilst you're there, be a darling and rename 'Serial Number' to 'SerialNumber' and 'Group Tag' to 'GroupTag', like so:

```csv
SerialNumber,GroupTag
VMware-56 4d 71 00 31 a9 92 c6-ca 79 52 44 c0 3a 83 22,AJ-LT-U-ADM-IT-UK
```

Now that you've got **all** the Autopilot devices, go ahead and laboriously update the CSV file with Group Tags and remove any devices that you want tagging with the default one.

# Repeat Offenders
We need a way to not only set the Group Tag, but fetch the ID of the Autopilot device, and as we're doing this potentially 100's of times, and I did say this was a repeatable and reusable script, we should create a PowerShell function or two.

## Handshake Time
Lets steal the PowerShell Authentication Function from the [Intune PowerShell Samples](https://github.com/microsoftgraph/powershell-intune-samples) GitHub repo to allow us to connect to Graph.

### Authentication Function

```PowerShell
function Get-AuthToken {

    <#
    .SYNOPSIS
    This function is used to authenticate with the Graph API REST interface
    .DESCRIPTION
    The function authenticate with the Graph API Interface with the tenant name
    .EXAMPLE
    Get-AuthToken
    Authenticates you with the Graph API interface
    .NOTES
    NAME: Get-AuthToken
    #>
    
    [cmdletbinding()]
    
    param
    (
        [Parameter(Mandatory = $true)]
        $User
    )
    
    $userUpn = New-Object "System.Net.Mail.MailAddress" -ArgumentList $User
    
    $tenant = $userUpn.Host
    
    Write-Host "Checking for AzureAD module..."
    
    $AadModule = Get-Module -Name "AzureAD" -ListAvailable
    
    if ($null -eq $AadModule) {
    
        Write-Host "AzureAD PowerShell module not found, looking for AzureADPreview"
        $AadModule = Get-Module -Name "AzureADPreview" -ListAvailable
    
    }
    
    if ($null -eq $AadModule) {
        write-host
        write-host "AzureAD Powershell module not installed..." -f Red
        write-host "Install by running 'Install-Module AzureAD' or 'Install-Module AzureADPreview' from an elevated PowerShell prompt" -f Yellow
        write-host "Script can't continue..." -f Red
        write-host
        exit
    }
    
    # Getting path to ActiveDirectory Assemblies
    # If the module count is greater than 1 find the latest version
    
    if ($AadModule.count -gt 1) {
    
        $Latest_Version = ($AadModule | Select-Object version | Sort-Object)[-1]
    
        $aadModule = $AadModule | Where-Object { $_.version -eq $Latest_Version.version }
    
        # Checking if there are multiple versions of the same module found
    
        if ($AadModule.count -gt 1) {
    
            $aadModule = $AadModule | select -Unique
    
        }
    
        $adal = Join-Path $AadModule.ModuleBase "Microsoft.IdentityModel.Clients.ActiveDirectory.dll"
        $adalforms = Join-Path $AadModule.ModuleBase "Microsoft.IdentityModel.Clients.ActiveDirectory.Platform.dll"
    
    }
    
    else {
    
        $adal = Join-Path $AadModule.ModuleBase "Microsoft.IdentityModel.Clients.ActiveDirectory.dll"
        $adalforms = Join-Path $AadModule.ModuleBase "Microsoft.IdentityModel.Clients.ActiveDirectory.Platform.dll"
    
    }
    
    [System.Reflection.Assembly]::LoadFrom($adal) | Out-Null
    
    [System.Reflection.Assembly]::LoadFrom($adalforms) | Out-Null
    
    $clientId = "d1ddf0e4-d672-4dae-b554-9d5bdfd93547"
    
    $redirectUri = "urn:ietf:wg:oauth:2.0:oob"
    
    $resourceAppIdURI = "https://graph.microsoft.com"
    
    $authority = "https://login.microsoftonline.com/$Tenant"
    
    try {
    
        $authContext = New-Object "Microsoft.IdentityModel.Clients.ActiveDirectory.AuthenticationContext" -ArgumentList $authority
    
        # https://msdn.microsoft.com/en-us/library/azure/microsoft.identitymodel.clients.activedirectory.promptbehavior.aspx
        # Change the prompt behaviour to force credentials each time: Auto, Always, Never, RefreshSession
    
        $platformParameters = New-Object "Microsoft.IdentityModel.Clients.ActiveDirectory.PlatformParameters" -ArgumentList "Auto"
    
        $userId = New-Object "Microsoft.IdentityModel.Clients.ActiveDirectory.UserIdentifier" -ArgumentList ($User, "OptionalDisplayableId")
    
        $authResult = $authContext.AcquireTokenAsync($resourceAppIdURI, $clientId, $redirectUri, $platformParameters, $userId).Result
    
        # If the accesstoken is valid then create the authentication header
    
        if ($authResult.AccessToken) {
    
            # Creating header for Authorization token
    
            $authHeader = @{
                'Content-Type'  = 'application/json'
                'Authorization' = "Bearer " + $authResult.AccessToken
                'ExpiresOn'     = $authResult.ExpiresOn
            }
    
            return $authHeader
    
        }
    
        else {
    
            Write-Host
            Write-Host "Authorization Access Token is null, please re-run authentication..." -ForegroundColor Red
            Write-Host
            break
    
        }
    
    }
    
    catch {
    
        write-host $_.Exception.Message -f Red
        write-host $_.Exception.ItemName -f Red
        write-host
        break
    
    }
    
}

```

### Authentication Token
As well as the call to connect to Graph.

```PowerShell
#region Authentication

write-host

# Checking if authToken exists before running authentication
if ($global:authToken) {

    # Setting DateTime to Universal time to work in all timezones
    $DateTime = (Get-Date).ToUniversalTime()

    # If the authToken exists checking when it expires
    $TokenExpires = ($authToken.ExpiresOn.datetime - $DateTime).Minutes

    if ($TokenExpires -le 0) {

        write-host "Authentication Token expired" $TokenExpires "minutes ago" -ForegroundColor Yellow
        write-host

        # Defining User Principal Name if not present

        if ($null -eq $User -or $User -eq "") {

            $User = Read-Host -Prompt "Please specify your user principal name for Azure Authentication"
            Write-Host

        }

        $global:authToken = Get-AuthToken -User $User

    }
}

# Authentication doesn't exist, calling Get-AuthToken function

else {

    if ($null -eq $User -or $User -eq "") {

        $User = Read-Host -Prompt "Please specify your user principal name for Azure Authentication"
        Write-Host

    }

    # Getting the authorization token
    $global:authToken = Get-AuthToken -User $User

}

#endregion
```

> *If you haven't authenticated to Graph in your tenant previously, you'll probably be asked to grant Admin Consent, you want to do this, consent is important.*

## Autopilot Functions
Now we can have a nice authenticated chat to Graph, we need to talk to the Autopilot section, [deviceManagement/windowsAutopilotDeviceIdentities](https://docs.microsoft.com/en-us/graph/api/resources/intune-enrollment-windowsautopilotdeviceidentity?view=graph-rest-beta) in fact. 

So let's get all Autopilot devices, as we'll need this to:
1. Get all Autopilot devices without a Group Tag set
2. Get the Autopilot device Id for the entries in the CSV file we created

### Getting Autopilot Devices
```PowerShell
Function Get-AutopilotDevices() {

    <#
    .SYNOPSIS
    This function is used to get autopilot devices via the Graph API REST interface
    .DESCRIPTION
    The function connects to the Graph API Interface and gets any autopilot devices
    .EXAMPLE
    Get-AutopilotDevices
    Returns any autopilot devices
    .NOTES
    NAME: Get-AutopilotDevices
    #>
    
    $graphApiVersion = "Beta"
    $Resource = "deviceManagement/windowsAutopilotDeviceIdentities"
    
    try {
    
        $uri = "https://graph.microsoft.com/$graphApiVersion/$($Resource)"
            (Invoke-RestMethod -Uri $uri -Headers $authToken -Method Get).Value
        
    }
    
    catch {
    
        $ex = $_.Exception
        $errorResponse = $ex.Response.GetResponseStream()
        $reader = New-Object System.IO.StreamReader($errorResponse)
        $reader.BaseStream.Position = 0
        $reader.DiscardBufferedData()
        $responseBody = $reader.ReadToEnd();
        Write-Host "Response content:`n$responseBody" -f Red
        Write-Error "Request to $Uri failed with HTTP Status $($ex.Response.StatusCode) $($ex.Response.StatusDescription)"
        write-host
        break
    
    }
    
}
```

### Update Autopilot Device Attributes
Now we can use the below function to set the Autopilot Group Tag (I ~~should~~ could probably update this to set other device attributes).

```PowerShell
Function Set-AutopilotDevice() {

    <#
    .SYNOPSIS
    This function is used to set autopilot devices properties via the Graph API REST interface
    .DESCRIPTION
    The function connects to the Graph API Interface and sets autopilot device properties
    .EXAMPLE
    Set-AutopilotDevice
    Returns any autopilot devices
    .NOTES
    NAME: Set-AutopilotDevice
    #>

    [CmdletBinding()]
    param(
        $Id,
        $GroupTag
    )
    
    $graphApiVersion = "Beta"
    $Resource = "deviceManagement/windowsAutopilotDeviceIdentities/$Id/updateDeviceProperties"

    try {

        if (!$id) {
            write-host "No Autopilot device Id specified, specify a valid Autopilot device Id" -f Red
            break
        }

        if (!$GroupTag) {
            $GroupTag = Read-host "No Group Tag specified, specify a Group Tag"
        }

        $Autopilot = New-Object -TypeName psobject
        $Autopilot | Add-Member -MemberType NoteProperty -Name 'groupTag' -Value $GroupTag

        $JSON = $Autopilot | ConvertTo-Json -Depth 3
        # POST to Graph Service
        $uri = "https://graph.microsoft.com/$graphApiVersion/$($Resource)"
        Invoke-RestMethod -Uri $uri -Headers $authToken -Method Post -Body $JSON -ContentType "application/json"
        write-host "Successfully added '$GroupTag' to device" -ForegroundColor Green
        
    }
    
    catch {
    
        $ex = $_.Exception
        $errorResponse = $ex.Response.GetResponseStream()
        $reader = New-Object System.IO.StreamReader($errorResponse)
        $reader.BaseStream.Position = 0
        $reader.DiscardBufferedData()
        $responseBody = $reader.ReadToEnd();
        Write-Host "Response content:`n$responseBody" -f Red
        Write-Error "Request to $Uri failed with HTTP Status $($ex.Response.StatusCode) $($ex.Response.StatusDescription)"
        write-host
        break
    
    }
    
}
```

# Scary Stuff Lies Ahead
So we now have the functions in order to connect, get and set all the things we need. Here comes the fun part, PowerShell logic driven by four cups of coffee.

## Script Parameters
Before that, some lovely parameters to ensure you can't really mess things up.

- **Method**: Set to be either 'CSV' or 'Online', used for loading that beautiful CSV created earlier, or to just grab all the Autopilot devices in the tenant with a blank Group Tag
- **DefaultGroupTag**: The 'Catch All' Group Tag for those devices that don't have one set already

## The Logic?
Here we have the guts of the script, designed so that you run it through once with the `CSV` option, then a second time using the `Online` option. Don't ask me why I did it this way, four coffees remember.

I was also kind enough to capture any of the devices in the CSV file missing a Group Tag and prompt to enter one in. Very kind.

```PowerShell
# Script Start
# Get Devices
if ($Method -eq 'CSV') {
    $CSVPath = Read-host "Please provide the path to the CSV file containing a list of device serial numbers and new Group Tag  e.g. C:\temp\devices.csv"

    if (!(Test-Path "$CSVPath")) {
        Write-Host "Import Path for CSV file doesn't exist" -ForegroundColor Red
        Write-Host "Script can't continue" -ForegroundColor Red
        Write-Host
        break
        
    }
    else {
        $AutopilotDevices = Import-Csv -Path $CSVPath
    }
}
elseif ($Method -eq 'Online') {
    Write-Host "Getting all Autopilot devices without a Group Tag" -ForegroundColor Cyan
    $AutopilotDevices = Get-AutopilotDevices | Where-Object { ($null -eq $_.groupTag) -or ($_.groupTag) -eq ''  }
}

# Sets Group Tag
foreach ($AutopilotDevice in $AutopilotDevices) {

    $id = $AutopilotDevice.id
    if (!$id) {
        Write-host "No Autopilot Device Id, getting Id from Graph" -ForegroundColor Cyan
        $id = (Get-AutopilotDevices | Where-Object { ($_.serialNumber -eq $AutopilotDevice.serialNumber) }).id
        Write-Host "ID:'$Id' found for device with serial '$($AutopilotDevice.Serialnumber)'" -ForegroundColor Green
    }

    if ($Method -eq 'CSV') {
        $GroupTag = $AutopilotDevice.groupTag
        if (!$GroupTag) {
            Write-host "No Autopilot Device Group Tag found in CSV" -ForegroundColor Cyan
            $GroupTag = Read-Host 'Please enter the group tag for device with serial '$AutopilotDevice.serialNumber' now:'
        }
    }

    elseif ($Method -eq 'Online') {
        $GroupTag = $DefaultGroupTag
    }

    try {
        Set-AutopilotDevice -Id $id -GroupTag $GroupTag
        write-host "Group tag: '$GroupTag' set for device with serial '$($AutopilotDevice.Serialnumber)'" -ForegroundColor Green
    }
    catch {
        write-host "Group tag: '$GroupTag' not set for device with serial '$($AutopilotDevice.Serialnumber)'" -ForegroundColor Red
    }


}
```
# Fire
So bring it all together into a [mega-script](https://github.com/ennnbeee/mem-scripts/blob/main/MEM/OS_Windows/Set-AutopilotGroupTag/Set-AutopilotGroupTag.ps1) we now have a way to update the Autopilot Group Tags. So let's give it a go.

> *P.S. There is no `-whatif` command, so I'd start with the CSV of a couple of test devices.*

## Sniper Time
Running the script with the CSV option:
```PowerShell
.\Set-AutopilotGroupTag.ps1 -Method CSV
```
We first have to Authenticate, so enter in your username and the find the Azure AD login window:
![Image](/img/autopilot-bulk-tag-login.png#left)

Now we need to provide the path to the CSV file:
```txt
Please provide the path to the CSV file containing a list of device serial numbers and new Group Tag  e.g. C:\temp\devices.csv:
```

Now the script will run and update all the devices in the CSV file with their corresponding Group Tags:

![Image](/img/autopilot-bulk-tag-csv.png#left)

And if we check in Endpoint Manager:
![Image](/img/autopilot-bulk-tag-csv-tagged.png#left)

Which amazingly, the Group Tag matches the data in the CSV file we created earlier. Too early to call this a win outright, but we're on the way.


## Shotgun Approach
We need to now clear up the remaining devices without Group Tags, this one we can't really test, unless you fancy improving the script.

Similar setup to the CSV run, but this time the arguments look like the below:
```PowerShell
.\Set-AutopilotGroupTag.ps1 -Method Online -DefaultGroupTag 'AJ-LT-U-STD-ALL-UK'
```
We're already authenticated, so we can skip that bit, and we're not using the CSV option so it will get straight to the good stuff:

![Image](/img/autopilot-bulk-tag-online.png#left)

And if we check in Endpoint Manager:
![Image](/img/autopilot-bulk-tag-online-tagged.png#left)

This Group Tag matches the `DefaultGroupTag` parameter we set when running the script. I'd call this one a win.

# Final Thoughts
There might be easier ways of doing this, or a little less caffeine fuelled at least, but if you want to bulk set Group Tags to your existing Autopilot devices, this does seem like a half decent approach.

For new devices, I recommend that you work with your supplier/OEM and get them to tag them as part of the on-boarding.

Also, you can run this many times, so if you do want to re-tag devices, you can use the CSV method to do so.

Also also, you should look at [this post](https://memv.ennbee.uk/posts/autopilot-power-of-group-tags/) about using dynamic groups to ring fence your newly tagged devices.

GLHF