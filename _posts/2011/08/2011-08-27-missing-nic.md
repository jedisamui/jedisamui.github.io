---
layout: post
title: "Missing NIC"
summary: "Oops, what happened to my NIC?"
author: samui
thumbnail: /images/2011/08/nic.png
permalink: /blog/missing-nic/
date: "2011-08-27"
category: [esx, vmware, howto, troubleshooting, vmnic, vmware, vmware-kb]
published: true
---

Had an interesting event yesterday. A new application build was underway that wasn't going very smoothly. Snapshots were being made and reverted rather frequently. After one one snapshot reversion, progress on the install was being made. Approximately three hours went by and a high priority ticket was raised. Something had gone wrong, and no one could access the VM. It was unresponsive to pings, RDP, etc. Eventually, it was discovered that the NIC was missing. Once the NIC was readded, access to the VM was restored and the installation group was off and running.

A forensic investigation was conducted into the root cause of the missing NIC. It was suggested that one of the snapshots had corrupted. The event logs within virtualcenter were vague - they provided the timeline of what had been occurring, but failed to indicate what had transpired with the NIC. I downloaded the vm logs from the datastore and examined them. I wanted to see if there were problems during a snapshot capture, or a snapshot reversion. There were no problems or failures with snapshot creation or any rollbacks of the snapshots. I did however, come across an entry that had me puzzled.

E1000: Syncing with mode 6.

Of course, Professor Google was up for my challenge and provided me with this bit of info within the VMware Communities: [Network card is removed without any user intervention](http://communities.vmware.com/message/1399846#1399846){:target="_blank"}. I struck gold. One of the commentors,"NMNeto", had hit the nail on the head with this comment: _"The problem is not in VMWARE ESX, it is in the hot plug technology of the network adapter. This device (NIC) can be removed by the user like a usb drive. ... When you click to remove the NIC, VMWare ESX removes this device from the virtual machine hardware."_

![Removable_NIC](/images/Vmware01.jpg)
Safely Remove Hardware Icon
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}

From here, I had a strong lead. Now that I knew what I was looking for, I opened the Virtual Machine's Event Viewer and started itemizing each entry - looking for the device removal. After about five minutes, I found the who, when, where, and how. I felt like I was winning a game of "Clue".

"NMNeto" had also posted an [adjoining link](http://communities.vmware.com/thread/227036){:target="_blank"} within the community for a thread that posted a resolution to prevent the issue going forward. Here's an [image from that link](http://clip2net.com/clip/m19209/1249766623-clip-120kb.png){:target="_blank"} that gives you step by step instructions on how to prevent this on other VMs.

I will take this data and propose a change to add this entry to our VMs so we do not have this event to reoccur.
