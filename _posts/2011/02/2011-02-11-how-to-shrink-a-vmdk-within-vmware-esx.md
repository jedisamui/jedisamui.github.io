---
layout: post
title: "How to Shrink a VMDK within Vmware ESX"
summary: "Reference link to shrink a disk"
author: samui
thumbnail: /images/2011/02/shrink-disk.png
permalink: /blog/how-to-shrink-a-vmdk-within-vmware-esx/
date: "2011-02-11"
category: [esx, vmware, troubleshooting, vmware]
published: true
---

I know. Do we really need yet another article on how to shrink a disk within a VM? Well, this one is for my reference. I needed somewhere to write it down and reference for that rare occasion I have to do it again.

A vmdk file is the file that represents a vmware virtual hard drive. The problem is that reducing the size of a vmdk is a frustrating a pain in the butt. However, I have found a way to re-size the vmdk and reduce the file size in the process. I am going to use Vmware converter to convert the disk from its original size to a reduced size.

Here's the steps I follow to shrink the VMDK file.

1) Stop any critical services on the VM you are going to convert. 
2) Open vmconverter and select the VM that you want to shrink. 
3) When you go through the converter wizard, select "Powered On Machine." 
4) At the end of the wizard, you will be required to review the system information. Click "Edit" next to the hard drives. 
5) Deselect any drives that you do not need to convert. Edit the size of the drive you are converting to the smaller size you desire. 
6) Finish the converter wizard and wait for the conversion to complete. Once it is complete the new disk will be available in your ESX farm (wherever you decided to store it during the conversion). 
7) While the VM is powered on go into Disk Manager (right click "My Computer - select "Manage" - "Disk Management") and right click the disk you are shrinking. Select "Change Drive Letter and Paths" and the select "Remove". 
8) Power off the virtual machine. 
9) Locate the shrunken vmdk file that was just created, and move it to the original location of the VM. 
10) Go into "Edit Settings" of the VM and remove the disk you are shrinking. (don't delete the old disk until you have tested the new disk). 
11) Add in the new disk you just created with vmconverter. 
12) Power on the VM, Go to Disk Manager and initiate the disk (It usually will do this automatically but if it's not done you need to do it.) 
13) Assign the drive letter to the disk and test. If you run into any issues you can always add the old disk back in.
