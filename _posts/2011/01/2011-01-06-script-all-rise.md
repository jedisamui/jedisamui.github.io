---
layout: post
title: "Script: All Rise"
summary: "Reference Script: Power everything back up"
author: samui
thumbnail: /images/2011/01/power-on.png
permalink: /blog/script-all-rise/
date: "2011-01-06"
category: [esx, vmware, powercli, script]
published: true
---

Yesterday, I posted a modified script that can be used to shutdown all VMs and ESX hosts in an environment. That script is useful for when you need to bring down an entire environment. It's easier to have a script go in and gracefully power off all VMs and ESX hosts, versus all the right-clicking you would have to do otherwise.

So what do you do when you are ready to bring the environment back online? Today, I'd like to share with you the sister script to the Emergency Shutdown Script - it's the Startup Script. Once the ESX server has been powered on, this script will scan an ESX server and power on all VMs that are in a powered off state.

```
@"
===============================================================================
Title: 			STARTUP v2.1
Created:		J. Sam Aaron 	12/24/2010
Description: 	Powers On all VMs 
Requirements: 	Windows Powershell and the VI Toolkit
Usage:			.\startup.ps1
===============================================================================
"@

# Add the VI-Snapin if it isn't loaded already
if ( (Get-PSSnapin -Name "VMware.VimAutomation.Core" -ErrorAction SilentlyContinue) -eq $null )
	{
	Add-PSSnapin -Name "VMware.VimAutomation.Core"
	}

# Variables
$vc = Read-Host "Enter the ESX IP Address"
$VIServer = Connect-VIServer -Server $vc 

# vc had some timeouts so retry...
if ($null -eq $VIServer) {
    write-host "Retrying Connect to "$vc    
	$VIServer = Connect-VIServer -Server $vc
	}

# Starting Process
Write-Host "Connected to" $vc	
Write-Host
Write-Host "==============================================================================="
Write-Host "===                       STARTUP PROCESS COMMENCING                        ==="
Write-Host "==============================================================================="

# Get All the ESX Hosts 
$ESXSRV = Get-VMHost

# Bring ESX out of Maintenance Mode
Write-Host "Taking $vc out of Maintenance Mode"
$ESXSRV | Set-VMHost -State Connected | Out-Null
Write-Host
Write-Host $vc "is not in Maintenance Mode"

# For each of the VMs on the ESX hosts 
Foreach ($VM in ($ESXSRV | Get-VM | Where { $_.PowerState -eq “poweredOff” } )){
	 # Power on VM
	 Write-Host
	 Write-Host "Powering up" $vm
	 Get-VM $vm | where { $_.PowerState –eq "PoweredOff" } | Start-VM | Out-Null
	 If ($_.PowerState -eq "PoweredOn"){ 
		 Write-Host $vm "is already Powered On."
		}
	}

# Summary
$numvms = ($ESXSRV | Get-VM | Where { $_.PowerState -eq "poweredOn" }).Count
Write-Host
Write-Host "==============================================================================="
Write-Host "===                                 SUMMARY                                 ==="
Write-Host "==============================================================================="
Write-Host
Write-Host "A total of" $numvms "vms have been started."
Write-Host "Please check the KVM console for OS availability."
Write-Host "It may take another minute or two before the OS is ready."
Write-Host
Write-Host "==============================================================================="
Write-Host "===                   THE STARTUP PROCESS HAS COMPLETED.                    ==="
Write-Host "==============================================================================="

	
# Disconnect from the ESX Server
Write-Host
Disconnect-VIServer -Server $vc -Confirm:$False
Write-Host "Disconnected from" $vc
Write-Host

#######################################################################
# Changes:
# Version 2.1 - Added VM count
# Version 2.0 - Corrected typos and cosmetic changes
# Version 1.9 - Added Maintenance Mode
# Version 1.8 - Added a line to print that the VM was powered on
# Version 1.7 - Added Prompt for Host
# Version 1.6 - Added Sanity Check for PSSnapin
# Version 1.5 - Added VC Disconnection
# Version 1.4 - Added VC Connection
# Version 1.3 - Added VM Shutdown
# Version 1.2 - Added Title Sequence
# Version 1.1 - Added PSSnapin
# Version 1.0 - Initial Creation

```
