---
title: "Microsoft BitLocker: BitLocker and WinRE on failed Intune Devices"
date: 2022-03-17T16:44:09Z
draft: false
description: ""
tags: ["endpoint", "intune", "windows", "bitlocker"]
ShowToc: true
cover:
    image: "/img/bitlocker.png" # image path/url
    alt: "Enable BitLocker and WinRE on failed Intune Devices" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---
You may have enabled and configure BitLocker for silent encryption on your Windows 10 Autopilot joined devices, but have you had the headache of devices that don't have a Windows Recovery Environment (WinRE) configured? Yep? Me too...

What you'll see in either the Bitlocker-API event log, or within the Encryption Readiness reporting in Endpoint Manager the following, glorious error:
```txt {linenos=false,hl_lines=[1]}
The OS volume is unprotected | Windows Recovery Environment (WinRE) isn't configured
```

# Configuration
So how do we go about enabling WinRE if it exists, setup BitLocker encryption, **and** grab the BitLocker recovery key and ping it to Azure AD?

Here's how...

## Updating the Script
This [Microsoft script](https://docs.microsoft.com/en-us/archive/blogs/showmewindows/how-to-enable-bitlocker-and-escrow-the-keys-to-azure-ad-when-using-autopilot-for-standard-users) has been adapted to check for the WinRE configuration before it continues and attempts to enable BitLocker, the `$HotToTrot` variable is used to denote whether to continue or not.

The below is the added section to check and enable, or attempt to enable, WinRE:

```powershell
$HotToTrot ="false"
#Checks Windows Recovery Environment and enables if disabled
if($WinREStatus -like '*Windows RE status:         Enabled*'){
    $HotToTrot = "True"
    Write-Verbose -Message "WinRE Partition Enabled and good to enable BitLocker $HotToTrot"
}
Else{
    Try{
        $WinREEnable = reagentc.exe /enable
        if($WinREEnable -like '*Operation Successful*'){
            $HotToTrot = "True"
            Write-Verbose -Message "WinRE Partition Enabled and good to enable BitLocker, HotToTrot set to $HotToTrot"
        }
        Else{
            $HotToTrot ="false"
            Write-Verbose -Message "Unable to enabled WinRE, HotToTrot set to $HotToTrot"
        }
    }
    Catch{
        $HotToTrot ="false"
        Write-Verbose -Message "Unable to enabled WinRE"
    }
}
if($HotToTrot -eq 'True')
```
## Fixing the Script
The script has logic in place to escrow the recovery key to Azure AD, using either the `BackupToAAD-BitLockerKeyProtector` commandlet or, if this isn't available, using a call to GraphAPI. The below sections needed to be updated due to where the Azure AD Join information is now stored in the registry:

```powershell {hl_lines=[25 26 28 29]},
# Check if we can use BackupToAAD-BitLockerKeyProtector commandlet
if (Get-Command -Name "BackupToAAD-BitLockerKeyProtector" -ErrorAction "SilentlyContinue") {
    
    # BackupToAAD-BitLockerKeyProtector commandlet exists
    Write-Verbose -Message "Saving Key to AAD using BackupToAAD-BitLockerKeyProtector"
    $BLV = Get-BitLockerVolume -MountPoint $OSDrive | Select-Object *
    If ($Null -ne $BLV.KeyProtector) {
        BackupToAAD-BitLockerKeyProtector -MountPoint $OSDrive -KeyProtectorId $BLV.KeyProtector[1].KeyProtectorId
    }
    Else {
        Write-Error "'Get-BitLockerVolume' failed to retrieve drive encryption details for $OSDrive"
    }
}
else { 
    # BackupToAAD-BitLockerKeyProtector commandlet not available, using other mechanism
    Write-Verbose -Message "BackupToAAD-BitLockerKeyProtector not available"
    Write-Verbose -Message "Saving Key to AAD using Enterprise Registration API"
            
    # Get the AAD Machine Certificate
    $cert = Get-ChildItem -Path "Cert:\LocalMachine\My\" | Where-Object { $_.Issuer -match "CN=MS-Organization-Access" }

    # Obtain the AAD Device ID from the certificate
    $id = $cert.Subject.Replace("CN=", "")

    # Obtain the Tenant ID from the certificate thumbprint
    $tenantid = ($cert.Thumbprint).Replace("-","")

    # Get the tenant name from the registry
    $tenant = (Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\CloudDomainJoin\JoinInfo\$($tenantid)").UserEmail.Split('@')[1]

    # Create the URL to post the data to based on the tenant and device information
    $url = "https://enterpriseregistration.windows.net/manage/$tenant/device/$($id)?api-version=1.0"

    # Generate the body to send to AAD containing the recovery information
    Write-Verbose -Message "Saving key protector to AAD for self-service recovery by manually posting it to:"
    Write-Verbose -Message "`t$url"
            
    # Get the BitLocker key information from WMI
    (Get-BitLockerVolume -MountPoint $OSDrive).KeyProtector | Where-Object { $_.KeyProtectorType -eq 'RecoveryPassword' } | ForEach-Object {
        $key = $_
        $body = "{""key"":""$($key.RecoveryPassword)"",""kid"":""$($key.KeyProtectorId.replace('{','').Replace('}',''))"",""vol"":""OSV""}"
        Write-Verbose -Message "KeyProtectorId : $($key.KeyProtectorId) key: $($key.RecoveryPassword)"
                    
        # Post the data to the URL and sign it with the AAD Machine Certificate
        $req = Invoke-WebRequest -Uri $url -Body $body -UseBasicParsing -Method "Post" -UseDefaultCredentials -Certificate $cert
        $req.RawContent
        Write-Verbose -Message " -- Key save web request sent to AAD - Self-Service Recovery should work"
    }
}
```
## Putting it All Together
The full script can be found [here](https://github.com/ennnbeee/mem-scripts/blob/main/MEM/OS_Windows/Enable-Bitlocker/Enable-WinRE_Bitlocker.ps1), I would **strongly** advise testing this prior to pushing it out via Endpoint Manager.

# Script Deployment
Save the above script and create a new PowerShell script deployment in [Endpoint manager](https://endpoint.microsoft.com/#blade/Microsoft_Intune_DeviceSettings/DevicesWindowsMenu/powershell) using the following configuration settings, then deploy to a test group of devices.

 ![Image](/img/bitlocker-script.png#left)
 
 Bingo! One battle won, onto the next.