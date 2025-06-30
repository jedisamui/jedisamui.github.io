---
layout: post
title: "Script: Detailed NIC Information Report"
summary: "Automating NIC Info"
author: samui
thumbnail: /images/2011/08/nic.png
permalink: /blog/script-detailed-nic-information-report/
date: "2011-08-22"
category: [esx, vmware, powershell, script, tools, vmnic, vmware]
published: true
---

Over the last few days, my coworker and I have been working on a script that will go out and collect detailed network information for each NIC on our ESX hosts. Not wanting to reinvent the wheel, I strolled over to Alan Reneuf's website (www.virtu-al.net). I had remembered seeing a script (["More Network Info"](http://www.virtu-al.net/2009/02/13/more-network-info/){:target="_blank"}) he created that I knew could collect the info we wanted. Thank you, Alan.

I copied Alan's script into Notepad++ and started to work with it. One of the things we needed to know was duplex information. Since that wasn't in the original script, I added it. I didn't need to know the number of active virtual switch ports, so I removed that portion.

Here are the end results of this modified script.

**NETWORKINFO.PS1**

##==============================================================================
##==============================================================================
##  SCRIPT.........:  networkinfo.ps1
##  AUTHOR.........:  J. Sam Aaron
##  EMAIL..........:  sam.aaron@micronauts.us
##  VERSION........:  1.0.6
##  DATE...........:  08/22/2011
##  COPYRIGHT......:  2011, Sam Aaron
##  LICENSE........:  Freeware
##  REQUIREMENTS...:  Powershell v2.0 and the VI Toolkit
##	USAGE..........:  ./networkinfo.ps1
##
##  DESCRIPTION....:  This program will collect detailed Network info 
##					  
##
##  NOTES..........:  Modified & Combined code from Alan Reneuf & Jeff Grummons 
## 
##  CUSTOMIZE......:  
##==============================================================================

##==============================================================================
##  START `##==============================================================================  	## LOAD THE VI-SNAPIN 	##============================================================================== 	if ( (Get-PSSnapin -Name "VMware.VimAutomation.Core" -ErrorAction SilentlyContinue) -eq $null ) 		{ 		Add-PSSnapin -Name "VMware.VimAutomation.Core" 		}  	##============================================================================== 	##  VARIABLES 	##============================================================================== 		$vc = "virtualcentername"  		$filename = "c:\POSH\Reports\DetailedNetworkInfo.csv" 		$MyCol = @()  		 	## VIRTUAL CENTER CONNECTION 	##============================================================================== 	Connect-VIServer -Server $vc   	## GATHER HOSTS FROM CLUSTER 	##============================================================================== 	$clusters = get-cluster | Select Name 	foreach ($cluster in $clusters) 		{ 		$h = $cluster 		$vmhosts = Get-Cluster $h.Name | Get-VMHost | Sort Name | Get-View  		 		## COLLECT ENTIRE NETWORK SYSTEM FOR EACH HOST 		##============================================================================== 		foreach($vmhost in $vmhosts) 			{ 			$ESXHost = $vmhost.Name 			Write-host "Collating information for $ESXHost" 			$networkSystem = Get-view $vmhost.ConfigManager.NetworkSystem  			## NETWORK INFORMATION FOR EACH NIC  			##============================================================================== 			foreach($pnic in $networkSystem.NetworkConfig.Pnic) 				{ 				$pnicInfo = $networkSystem.QueryNetworkHint($pnic.Device)  				## FOR EACH NIC COLLATE NEEDED NETWORK INFORMATION  				##============================================================================== 				:Details foreach($Hint in $pnicInfo) 					{ 					$NetworkInfo = "" | select-Object Host, Cluster, vSwitch, PNic, Speed, Duplex, Configured, MAC, DeviceID, PortID, Observed, VLAN 					$NetworkInfo.Host = $vmhost.Name 					$NetworkInfo.Cluster = $h.Name 					$NetworkInfo.PNic = $Hint.Device 					$NetworkInfo.DeviceID = $Hint.connectedSwitchPort.DevId 					$NetworkInfo.PortID = $Hint.connectedSwitchPort.PortId 					If ($NetworkInfo.PortID -eq $null) { continue Details } 					$NetworkInfo.vSwitch = Get-Virtualswitch -VMHost (Get-VMHost ($vmhost.Name)) | where {$_.Nic -eq ($Hint.Device)} 					$record = 0 					Do{ 						If ($Hint.Device -eq $vmhost.Config.Network.Pnic[$record].Device) 							{ 							$NetworkInfo.Speed = $vmhost.Config.Network.Pnic[$record].LinkSpeed.SpeedMb 							$extra = $vmhost.Config.Network.Pnic[$record].LinkSpeed.duplex 							if ($extra -eq $True) {$NetworkInfo.duplex = "Full"} else {$NetworkInfo.duplex = "Half"} 							$NetworkInfo.MAC = $vmhost.Config.Network.Pnic[$record].Mac	 							} 						$record ++ 					 }Until ($record -eq ($vmhost.Config.Network.Pnic.Length)) 					  					if($Hint.ConnectedSwitchPort.FullDuplex -eq $true) 						{ 						$NetworkInfo.Configured = "Full" 						} 						elseif($Hint.ConnectedSwitchPort.FullDuplex -eq $false) 							{ 							$NetworkInfo.Configured = "Half" 							} 					 					foreach ($obs in $Hint.Subnet) 						{ 						$NetworkInfo.Observed += $obs.IpSubnet + " " 						Foreach ($VLAN in $obs.VlanId) 							{ 							If ($VLAN -eq $null) 								{ 								} else { 									$strVLAN = $VLAN.ToString() 									$NetworkInfo.VLAN += $strVLAN + " " 									} 							}	 						} 					$MyCol += $NetworkInfo 					} 				} 			} 		} 		 		 		## DISPLAY OUTPUT & SAVE TO CSV 		##============================================================================== 		$mycol | Out-GridView  		$Mycol | Sort Host, PNic | Export-Csv $filename -NoTypeInformation  		 		## DISCONNECT FROM VC 		##============================================================================== 		Disconnect-VIServer -Server $vc -Confirm:$False 		Write-Host "Disconnected from" $vc 			 ##============================================================================== ##  END` 
##==============================================================================

##==============================================================================
##  REVISED BY.....:  J. Sam Aaron
##  EMAIL..........:  sam.aaron@micronauts.us
##	REVISION.......:  1.0.6
##  REVISION DATE..:  08-22-2011
##  REVISION NOTES.:  Cleanedup Script and added comments
##					  
##
##==============================================================================
# Version 1.0.6 - Cleanedup Script and added comments
# Version 1.0.5 - Fixed misplaced bracket (line 119)
# Version 1.0.4 - Added actual duplex settings (line 91)
# Version 1.0.3 - Check Link status - display if active (line 84)
# Version 1.0.2 - Added Cluster loop info (line 51)
# Version 1.0.1 - Initial Creation

