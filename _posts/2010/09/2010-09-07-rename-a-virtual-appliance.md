---
layout: post
title: "Rename a virtual appliance"
summary: "How do I change this things name?"
author: samui
thumbnail: /images/2010/09/hostname.png
permalink: /blog/rename-a-virtual-appliance/
date: "2010-09-07"
category: [daily] 
published: true
---

I have come across this scenario several times and would like to share how I resolved it.

I have downloaded and put into production a handful of linux virtual appliances from VMware's Virtual Appliance Library. Due to company naming standards, these virtual appliances had to be renamed. I spent several hours trying to figure out how to do this. Unfortunately, many Virtual Appliance's web GUI(s) are usually not helpful in this area. Most normal linux hostname editing is reset when the VM is rebooted.

For those people that build virtual appliances who have added the ability to rename the host within the GUI - THANK YOU for making this task easier.

The following instructions are for those Linux Virtual Appliances that do not have a way to rename the hostname of the machine within the GUI.  
1) Within the commandline of the VM.

`#nano /etc/init.d/vaos set hostname variable to “preferredhostname”` 

2) Save, Exit, and Reboot
