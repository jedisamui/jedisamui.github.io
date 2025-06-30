---
layout: post
title: "Creating an ISO in ESX from a mounted CD/DVD"
summary: "Reference Create an ISO via commandline"
author: samui
thumbnail: /images/2010/09/dosprompt.png
permalink: /blog/creating-an-iso-in-esx-from-a-mounted-cd/
date: "2010-05-10"
category: [esx, vmware]
published: true
---

I can't remember where I found this article. But it too is being repeated so that I have a permanent copy of it.

Article: Creating an ISO in ESX from a mounted CD/DVD

I am using VMware ESX Server 3.5.0 to host a number of test machines, and I needed to mount the SharePoint Server 2007 CD-ROM so that I could install it. I used PuTTY to connect to the ESX server and mounted the CD at the command line as follows...

**sudo mkdir /media/cdrom [enter] sudo mount /dev/cdrom /media/cdrom [enter]**

That seemed like a good idea at the time, but now I either have to leave the CD in the drive, or run the physical media into the datacenter every time I need it going forward. Having plenty of disk space, I wanted to create an ISO file and save it to the server. While I could create an ISO on my client PC and load it up to the server, I already had the media mounted on the ESX server and figured there must be a way to do this.

There is.

Assuming the disc is mounted to /dev/cdrom, and that there is a directory called /vmwareiso and you want to name the ISO file image1.iso

**sudo dd if=/dev/cdrom of=/vmwareiso/image1.iso [enter]**

Viola! Just don't forget to retrieve the disc or you will be looking for it a month from now and wondering who stole it!
