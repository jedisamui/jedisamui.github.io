---
layout: post
title: "Genuine NUMA Boot Error"
summary: "Oops... What to do during a NUMA boot error"
author: samui
thumbnail: /images/2011/02/ESX_NUMA.jpg
permalink: /blog/genuine-numa-boot-error/
date: "2011-02-15"
category: [esx, vmware, troubleshooting, vmware]
published: true
---

My co-worker ran into this error this week - _"TSC: 54606870 cpu0:0) NUMA: 706: Can't boot system as genuine NUMA. Booting with 1 fake node(s)"_. He was upgrading a Dell PowerEdge 6850 from ESX 3.5 Update 5 to ESX 4.0 Update 2. He was wiping the machine and doing a fresh install, so this not an upgrade installation. After the installation completed and he booted it up, it generated an error similiar to what's pictured.

I knew the issue involved the CPU as I just read an article concerning [NUMA](http://frankdenneman.nl/2010/12/node-interleaving-enable-or-disable/?utm_source=feedburner&utm_medium=feed&utm_campaign=Feed:+frankdenneman/ZjZC+\(frankdenneman.nl\)&utm_content=Google+Reader){:target="_blank"} just the day before. I did a little research on my end and found a link to the [Dell PowerEdge 6850's ReleaseNotes PDF](http://support.dell.com/support/edocs/software/eslvmwre/VS/docs/Releasenotes/RN_4_2.pdf){:target="_blank"}. The error can be ignored as it will not interfere with the functions of the ESX OS.

Unfortunately, the error looks scarey and will not fly within our environment. Upper management will not like seeing an error on the screen if they do a pop inspection. So the $64,000 question is "How do you get rid of it?" Another google search and minutes later, I found the solution.

1) Use the vSphere Client to connect to the vCenter 
2) Left-click on the ESX server 
3) Click the Configuration tab 
4) (Under Software) Click Advanced Settings 
5) In the new window that pops up, locate and click on Vmkernel 
6) Over on the right-hand side of the window, locate vmkernel.boot.usenumainfo (It should be the 8th entry from the bottom of list) and uncheck 
![vmkernel NUMA Setting](/images/2011/02/NUMA.jpg)
NUMA
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}

7) Reboot the ESX host (Don't forget to put into maintenance mode and evac the VMs)
