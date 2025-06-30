---
layout: post
title: "Script: Emergency - Shutdown"
summary: "Reference script to shutdown the lab"
author: samui
thumbnail: /images/2011/01/emergency-shutoff.jpg
permalink: /blog/script-emergency-shutdown/
date: "2011-01-05"
category: [esx, powercli, script, vmware]
published: true
---

Recently I had a vSphere 4.0 U1 environment experience a major outage. I had to bring down our entire environment consisting of 242 guests and 6 hosts. I knew that I had a bash script that I wrote for a 3.5 environment for this scenario, but due to logistical reasons it was unavailable. I had to come up with a quick solution as everyone was waiting on me.

A quick search on a site that is becoming a favorite of mine found me a [solution](http://www.virtu-al.net/2010/01/06/powercli-shutdown-your-virtual-infrastructure/) that I could run in powercli. Alan Renouf created this script. (Thank you Alan.) After the outage, I took another look at his script and wanted to add a few other touches to it - specifically a warning screen with a way to safely stop the script before starting to shutdown VMs. Since I will be sharing this script with co-workers, I needed a way for them to stop the script if they accidentally started running it by mistake.

I am running the following revised script in PowerCLi version 4.1.

```
@"
===============================================================================
Title: 			SHUTDOWN v2.7
Created:		Alan Renouf
Modified by:	J. Sam Aaron 	12/24/2010
Description: 	Shutsdown all VMs and then the ESX Server
Requirements: 	Windows Powershell and the VI Toolkit
Usage:			.\shutdown.ps1
===============================================================================
"@

# Add the VI-Snapin if it isn't loaded already
if ( (Get-PSSnapin -Name "VMware.VimAutomation.Core" -ErrorAction SilentlyContinue) -eq $null )
	{
	Add-PSSnapin -Name "VMware.VimAutomation.Core"
	}

# Variables
Write-Host
$vc = Read-Host "Enter the ESX IP Address"
$VIServer = Connect-VIServer -Server $vc

# vc had some timeouts so retry...
if ($null -eq $VIServer) {
    write-host "Retrying Connect to "$vc
	$VIServer = Connect-VIServer -Server $vc
	}

# Warning Notice
$input = ""
clear
Write-Host "Connected to" $vc
Write-Host
Write-Host "==============================================================================="
Write-Host "===                                                                         ==="
Write-Host "===       WARNING!     WARNING!     WARNING!     WARNING!     WARNING!      ==="
(New-Object -ComObject sapi.spvoice).speak("WARNING! WARNING! WARNING!") | Out-Null
Write-Host "===                                                                         ==="
Write-Host "==============================================================================="
Write-Host
Write-Host "  This script is designed to power off all virtual machines and the ESX host."
Write-Host
Write-Host
$input = Read-Host "  TYPE 'NO' TO STOP THIS SCRIPT AND EXIT NOW."
if ($input -eq "no") {
	 Write-Host
	 Write-Host "==============================================================================="
	 Write-Host "===               THE SHUTDOWN PROCESS HAS BEEN CANCELLED.                  ==="
	 Write-Host "==============================================================================="
	 exit
	}
Write-Host
Write-Host "==============================================================================="
Write-Host "===                    THE SHUTDOWN PROCESS WILL RESUME.                    ==="
Write-Host "==============================================================================="

#Get All the ESX Hosts
$ESXSRV = Get-VMHost
#$ESXSRV = Get-Cluster "Cluster Name" | Get-VMHost
#$ESXSRV = Get-DataCenter “DC1” | Get-VMHost

# For each of the VMs on the ESX hosts
Foreach ($VM in ($ESXSRV | Get-VM | Where { $_.PowerState -eq “poweredOn” } )){
     # Shutdown the guest cleanly
     $VM | Shutdown-VMGuest -Confirm:$false | Out-Null
	 Write-Host
	 Write-Host $VM "is Powering Down"
	}

# Set the amount of time to wait before assuming the remaining powered on guests are stuck
$waittime = 60 #Seconds
$Time = (Get-Date).TimeofDay
 do {
	 # Wait for the VMs to be Shutdown cleanly
	 sleep 1.0
	 $timeleft = $waittime - ($Newtime.seconds)
	 $numvms = ($ESXSRV | Get-VM | Where { $_.PowerState -eq "poweredOn" }).Count
	 write-host
	 Write "  Waiting for shutdown of $numvms VMs or until $timeleft seconds"
	 $Newtime = (Get-Date).TimeofDay - $Time
	 } until ((@($ESXSRV | Get-VM | Where { $_.PowerState -eq "poweredOn" }).Count) -eq 0 -or $timeleft -eq 0) 

# Force VMs off
Foreach ($VM in ($ESXSRV | Get-VM | Where { $_.PowerState -eq “poweredOn” } )){
	 $VM | Stop-VM -Confirm:$false | Out-Null
	 Write-Host
	 Write-Host $VM "has been Powered Off"
	}

# Timer to delay Maintenance Mode
$waittime = 20 #Seconds
$Time = (Get-Date).TimeofDay
 do {
	 sleep 1.0
	 $timeleft = $waittime - ($Newtime.seconds)
	 $Newtime = (Get-Date).TimeofDay - $Time
	} until (($Newtime).Seconds -ge $waittime)

# Place ESX in Maintenance Mode
$ESXSRV | Set-VMHost -State Maintenance | Out-Null
Write-Host
Write-Host   $vc "is in Maintenance Mode"

# Shutdown the ESX Hosts
$ESXSRV | Foreach {Get-View $_.ID} | Foreach {$_.ShutdownHost_Task($TRUE)} | Out-Null
Write-Host
Write-Host "==============================================================================="
Write-Host "===                  THE SHUTDOWN PROCESS HAS COMPLETED.                    ==="
Write-Host "==============================================================================="

# Disconnect from the ESX Server
Write-Host
Disconnect-VIServer -Server $vc -Confirm:$False
Write-Host "  Disconnected from" $vc

#######################################################################
# Changes:
# Version 2.7 - Added the ability to stop the shutdown during the warning
# Version 2.6 - Added Delay for Maintenance Mode
# Version 2.5 - Added Warning Notice
# Version 2.4 - Added Force stop
# Version 2.3 - Fixed Bug in Time: Adjusted waittime to 60 seconds
# Version 2.2 - Added Maintenance Mode
# Version 2.1 - Added a line to print that the VM was powered off
# Version 2.0 - Cleaned up Script
# Version 1.9 - Added Prompt for Host
# Version 1.8 - Added Sanity Check for PSSnapin
# Version 1.7 - Added Countdown timer
# Version 1.6 - Added Sanity Check on VM State
# Version 1.5 - Added VC Disconnection
# Version 1.4 - Added VC Connection
# Version 1.3 - Added VM Shutdown
# Version 1.2 - Added Title Sequence
# Version 1.1 - Added PSSnapin
# Version 1.0 - Initial Creation

```


**A few words of caution: Run this script in a fresh powercli or powershell window. If connecting to vCenter, this script will contact EVERY ESX host and start shutting down ALL VMs. If you need to shut down specific VMs on an ESX server, then connect to that server individually. Or comment out LINE 58 and uncomment LINE 59 or 60 for the specific Cluster or Datacenter that you need powered off.**
