---
layout: post
title: "SVMotion does not rename files - Duncan Epping"
summary: "Need to rename a VM?"
author: samui
thumbnail: /images/2013/01/vm-naming.png
permalink: /blog/svmotion-does-not-rename-files/
date: "2013-01-30"
category: [esx, vmware, howto, svmotion, troubleshooting, vcenter]
published: true
---

Duncan Epping posted just a few days ago on his blog - [YellowBricks.com](http://www.yellow-bricks.com/2013/01/25/storage-vmotion-does-not-rename-files/){:target="_blank"} of an issue with files not being renamed when you svmotion a VM. This is a royal pain.

Here's a scenario of why this is aggravating. You have a VM named "TestVM". When it was created, it created the folder "TestVM" within a datastore with the VM's folder, the files were labeled "TestVM.vmdk", "TestVM.vmx", etc. If this VM was decommissioned, and you wound up reusing it at later time. Some admins would just rename the VM within vcenter and change the hostname of the virtual machine. Unfortunately, in the storage layer, the VM would still be listed as "TestVM". This can be confusing if you are having to do some cleanup in your datastores. You would come across a folder labeled "TestVM" and not know what VM this belongs to -- without going through each VM or running a powershell script to identify it. Like I said, a royal pain.

In the past, you could svmotion the VM to another datastore, and the svmotion process would rename the files. Unfortunately, this got left out of vSphere 5.0. Duncan's blog gives a fix for this so that you can get back to renaming files with svmotion.

Link: [http://www.yellow-bricks.com/2013/01/25/storage-vmotion-does-not-rename-files/](http://www.yellow-bricks.com/2013/01/25/storage-vmotion-does-not-rename-files/){:target="_blank"}
