---
title: "Endpoint Manager: Bulk Adding Device Notes"
date: 2022-05-03T17:22:55+01:00
draft: false
description: ""
tags: ["endpoint", "intune", "graph"]
ShowToc: true
cover:
    image: "/img/bulk-notes.png" # image path/url
    alt: "Intune Device Notes" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---
Ever had to add notes to Intune Managed Devices in bulk? Me either, well not until a few weeks ago when I needed an easy way to update the notes field on 100's of devices.

So luckily I stumbled upon a post by [Paul Wetter](https://wetterssource.com/get-device-notes-from-graph) about getting and setting notes on devices using [Graph API](https://docs.microsoft.com/en-us/graph/use-the-api), and specifically the Beta channel of Graph. Luckily, Paul did the ground work and created a couple of functions, which I have happily ~~stolen~~ borrowed, updated and tweaked to allow for updating notes en masse.

# Configuration
I've broken down each section of the script for a bit of a walk through, the important thing with all of this is making sure you have the Microsoft Graph Intune PowerShell module installed - `Install-Module -Name Microsoft.Graph.Intune` and you've connected to MSGraph - `Connect-MSGraph`.

## Getting Device Notes
The `Get-IntuneDeviceNotes` function leans on the Microsoft.Graph.Intune PowerShell module, to grab the Intune device ID using `Get-IntuneManagedDevice`, then grabs the device properties filtered to the Device Notes property. This will help us in getting individual device notes, as we kinda don't want to lose any that have already been added.

```PowerShell
Function Get-IntuneDeviceNotes{

    [CmdletBinding()]
    param (
        [Parameter(Mandatory=$true)]
        [String]
        $DeviceName
    )
    Try {
        $DeviceID = (Get-IntuneManagedDevice -filter "deviceName eq '$DeviceName'" -ErrorAction Stop).id
    }
    Catch {
        Write-Error $_.Exception.Message
        break
    }
    $Resource = "deviceManagement/managedDevices('$deviceId')"
    $properties = 'notes'
    $uri = "https://graph.microsoft.com/beta/$($Resource)?select=$properties"
    Try{
        (Invoke-MSGraphRequest -HttpMethod GET -Url $uri -ErrorAction Stop).notes
    }
    Catch{
        Write-Error $_.Exception.Message
        break
    }
}
```

## Setting Device Notes
The `Set-IntuneDeviceNotes` function also leans on the Microsoft.Graph.Intune PowerShell module, to grab the Intune device ID using `Get-IntuneManagedDevice`, and posts the formed JSON file to Graph to update the Notes property. Trouble with this, is that it will overwrite any existing notes that exist, so we need a way to sort this out.

```PowerShell
Function Set-IntuneDeviceNotes{
    [CmdletBinding()]
    param (
        [Parameter(Mandatory=$true)]
        [String]
        $DeviceName,
        [Parameter(Mandatory=$false)]
        [String]
        $Notes
    )
    Try {
        $DeviceID = (Get-IntuneManagedDevice -filter "deviceName eq '$DeviceName'" -ErrorAction Stop).id
    }
    Catch{
        Write-Error $_.Exception.Message
        break
    }
    If (![string]::IsNullOrEmpty($DeviceID)){
        $Resource = "deviceManagement/managedDevices('$DeviceID')"
        $GraphApiVersion = "Beta"
        $URI = "https://graph.microsoft.com/$graphApiVersion/$($resource)"
        $JSONPayload = @"
{
notes:"$Notes"
}
"@
        Try{
            Write-Verbose "$URI"
            Write-Verbose "$JSONPayload"
            Invoke-MSGraphRequest -HttpMethod PATCH -Url $uri -Content $JSONPayload -Verbose -ErrorAction Stop
        }
        Catch{
            Write-Error $_.Exception.Message
            break
        }
    }
}
```

## Bulk Setting Device Notes
The `Set-BulkIntuneDeviceNotes` function utilises both of the previous functions; one to get the existing notes and add it to a new array variable, then to update the variable with the new notes in the CSV file that gets imported. There's also some logic around whether there are existing notes using the `-match` operator, which is fun.

```PowerShell
Function Set-BulkIntuneDeviceNotes{
    [CmdletBinding()]
    param (
        [Parameter(Mandatory=$true)]
        [String]
        $DeviceList
    )
    if(Test-Path -Path $DeviceList){
        $Devices = Import-csv $DeviceList
        foreach($Device in $Devices){
            # Add Date stamp to the new notes
            $NewNotes = $Device.Notes
            $Notes = New-Object -TypeName System.Collections.ArrayList
            $Notes.AddRange(@(
                $NewNotes,
                "`n" # Adds a line break
                "`n" # Adds a line break
            ))
            # Get existing device notes
            Try{
                $OldNotes = Get-IntuneDeviceNotes -DeviceName $Device.Device
                If($OldNotes -match '\d' -or $OldNotes -match '\w'){
                    Write-Host "Existing notes found on $($Device.Device), adding to Notes variable" -ForegroundColor Cyan
                    $Notes.AddRange(@(
                        $OldNotes
                    ))
                }
                else{

                }
            }
            Catch{
                Write-Host "Unable to get device notes, ensure you are connected to MSGraph" -ForegroundColor Red
                Break
            }
            # Add the new notes, included the old ones
            Try{
                Set-IntuneDeviceNotes -DeviceName $Device.Device -Notes $Notes
                Write-Host "Notes successfully added to $($Device.Device)" -ForegroundColor Green
            }
            Catch{
                Write-Host "Unable to set device notes, ensure you are connected to MSGraph" -ForegroundColor Red
                Break
            }
        }
    }
    else{
        Write-Host "Unable to access the provided device list, please check the csv file and re-run the script." -ForegroundColor red
        Break
    }
}
```
## Running the Script
The only parameter required is a CSV file, which only has two headers, 'Device' and 'Notes', so feel free to create one yourself, as I don't think you need a template this time. Then just populate the CSV file with the devices and the required notes that need adding.

Sample CSV:
```csv
Device,Notes
ENB-13F278,Here are the New notes about the device.
```

To run the script, firstly install Microsoft Graph Intune PowerShell module and connect to Graph, then run the three functions, finally run the below updating the DeviceList path to your created CSV file:

```PowerShell
Set-BulkIntuneDeviceNotes -DeviceList C:\temp\UpdatedDeviceNotes.csv
```
I'd test this with a single device in the CSV file first before you do this in bulk, you know, just in case.

The full script can be found [here](https://github.com/ennnbeee/mem-scripts/blob/main/MEM/OS_All/Set-MEMDeviceNotes/Set-MEMDeviceNotes.ps1)

## The Results
But once run successfully, you'll have devices with updated notes, so from this...
![Image](/img/bulk-notes-before.png#left)

To this...
![Image](/img/bulk-notes-after.png#left)

# Conclusion
***Yes*** this was a hacky script, ***yes*** I spent more time writing this post about it than putting the script together...but the script does the job and saves having to manually update the notes field. Plus now you know how to do it, you can put whatever you want either in the csv file, or use something else to populate the field. World and Oyster.