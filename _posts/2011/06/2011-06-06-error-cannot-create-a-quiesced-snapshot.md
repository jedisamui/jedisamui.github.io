---
layout: post
title: "Error: Cannot create a quiesced snapshot"
summary: "Oops...Why does my snapshot fail?"
author: samui
thumbnail: /images/2011/06/esxi-take-quiesce-snapshot.png
permalink: /blog/error-cannot-create-a-quiesced-snapshot/
date: "2011-06-06"
category: [powershell, script, troubleshooting]
published: true
---

I came across an issue this weekend that I wanted to share. I've written a script that automates the process my company uses to conduct a refresh of a production machine. Recently, we've upgraded our vCenter from 4.0 to 4.1. In addition, PowerCLI was upgraded to 4.1.1. Of course, some things are off on my script now (I'm currently in the process of fixing all the new bugs the upgrade introduced).

Until I get all the kinks worked for the script to run properly again, I decided to run the script locally from my machine. The problem with this is that my workstation is not on the domain. This causes a few portions of the script that needs domain access to fail. One of these sections involves moving the source VM to a different OU so that the security banner is removed at login. I expected this portion to fail, however, I did not expect it to generate an error within the VC and domino out of control.

The error that I got was "Error: Cannot create a quiesced snapshot." I quickly stopped the script and referenced Professor Google on the issue, and found the following link on the matter.

[http://www.cenobite.eu/index.php?option=com\_content&view=article&id=32:cannot-create-a-quiesced-snapshot&catid=8:esx&Itemid=33](http://www.cenobite.eu/index.php?option=com_content&view=article&id=32:cannot-create-a-quiesced-snapshot&catid=8:esx&Itemid=33){:target="_blank"}

I quickly read through the post and verified my VM, the settings, and the script. Then I noticed something. This is the section of my script that caused the error.

```
# DISABLE SECURITY BANNER
$bDisableSecurityBanner = $True
# CALLS BANNER FUNCTION TO MOVE VM TO DIFFERENT OU
banner

# CALLS FUNCTION TO POWER OFF VM
PowerOff-VM $sourcevm
Write-host "Pausing for system shutdown"
# PAUSES TO GIVE THE VC ENOUGH TIME TO REPORT THE VM OFFLINE
Sleep 15
Write-Host "VM Shutdown Complete" 

# CALLS FUNCTION TO POWER ON VM
PowerOn-VM $sourcevm
Write-host "Pausing for system startup"
# PAUSES TO GIVE THE VM ENOUGH TIME TO START AND LOAD VMTOOLS
Sleep 30
Write-Host $sourcevm "Powered On"

```

The problem that occurred here was that the VM was power-cycled, but the desktop stilled showed that it was "Applying New Settings". So enough time had not transpired to provide the VM, the opportunity to complete the boot cycle. Right after this the script removes the destination VM, and then starts the clone process. Since the desktop had not completed the boot cycle, I got the error. When the clone process attempted to take the snapshot so that it could start cloning, the VM was not in a ready state.

To resolve this, I wound up having to revert back to the manual process. During that process, if I waited for the VM to complete its' boot cycle, then all was well. For me to correct my script, I will be implementing the following logic check to verify the status and see if it the powercycle was successful.

```
do 
	{
	sleep 10
	$vmview1 = Get-VM $sourcevm | get-view
	$vmview2 = Get-VM $sourcevm
	$status1 = $vmview1.Guest.ToolsStatus
	$status2 = $vmview2.PowerState
	}until (($status2 -match "PoweredOn") -and ($status1 -match "Ok"))

```

This should loop until the VM is registered as "PoweredOn" and the VMTools are registered as "OK". This will require VMTools being current, else the loop will never end because VMTools will not read "Ok".
