---
layout: post
title: "Cloning a running Virtual Machine using the Service Console"
summary: "Reference - How to clone a VM"
author: samui
thumbnail: /images/2010/05/vmclone.png
permalink: /blog/cloning-a-running-vm-using-the-service-console/
date: "2010-05-08"
category: [esx, vmware]
published: true
---

I found this article and it went straight into my bookmarks. I have used this in an ESX 3.5U5 environment with no problems.  
  
Article: Cloning a running Virtual Machine using the Service Console  
To clone a virtual machine with VirtualCenter you have to power off the guest, but what if your next maintenance window isn’t any time soon, you can’t afford to schedule the outage, or you just need a copy of the VM during normal business hours? Did you know that making a copy of a running, powered on VM is possible? At a high level the process requires a snapshot to freeze the VM’s original disk which in turn allows you to clone the frozen disk. This is essentially the way VCB, vRanger, or any of the live VM backup products work. Therefore, cloning a powered on VM can be accomplished with a little Console command magic.  
I want to acknowledge that researching this method was inspired by the [VMTN Virtualization Roundtable Episode 1 Podcast’s coverage of snapshots. Specifically Eric Siebert mentions that](http://vmetc.com/2008/05/24/virtualization-roundtable-podcast-from-vmtn/) [using VMware Converter as an alternative to committing snapshots](http://vmetc.com/2008/05/21/use-vmware-converter-to-solve-esx-snapshot-issues/) is not the best option and offers the idea of using vmkfstools to do the job.  
Also, this step by step guide is based on the “How can I hot clone a VM without using VMware Converter?” tip found at [Vmware-land.com’s VMware Tips](http://vmware-land.com/Vmware_Tips.html#Net2).  

1. Login to the Service Console (Use Putty for remote Console Access) 

2. Switch to your source VM’s directory.

For example if you are cloning a VM named MyVM1 enter: **#cd /vmfs/volumes/MyVM1**

3. Create a snapshot

Use vmware-cmd - syntax is: “vmware-cmd createsnapshot” **#vmware-cmd MyVM1.vmx createsnapshot “MyVM1 Snapshot” “Clone snapshot” 1 0**

Setting memory to 0 prevents the snapshotting of the VM’s memory which we do not want for the clone. It will return “createsnapshot (MyVM1 Clone snapshot 1 0) = 1” when it successfully creates the snapshot. Optionally you can create the snapshot using the Snapshot Manager in the VI Client.

4. Create a new VM using the VI Client

For this example we will call it MyVM2. Assign the NIC for this VM to an Internal Only vswitch (no physical NIC’s assigned to the vswitch) so it does not conflict with MyVM1. For the hard drive you can accept the 4GB default or make it smaller since you will be deleting it anyway. Do not power on the VM.

5. Back at the Console Delete the .vmdk files of the new VM.

Switch to the MyVM2 folder **#cd /vmfs/volumes/MyVM2**

Delete all vmdks **#rm \*.vmdk**

you will be prompted for deletion confirmation of the two vmdk files for the VM.

6. Copy the original disk to the new VM directory

Switch back to your original VM’s directory **#cd /vmfs/volumes/MyVM1**

Use vmkfstools to copy your original disk to the new VM’s directory, the format is “vmkfstools –i” **#vmkfstools –i MyVM1.vmdk /vmfs/volumes/MyVM2.vmdk**

7. Power on your new VM using the VI Client.

MyVM2 will be in a crash consistent state (hard powered off) so run chkdsk if you can.

8. Delete the snapshot

Remove all snapshots for MyVM1 from the Console. (you should already be in the MyVM1 folder from step 6) **#vmware-cmd MyVM1.vmx removesnapshots**

Optionally you can remove the snapshot using Snapshot Manager in the VI Client.

  
  
Posted on May 26th, 2008 by Rich at [VMETC](http://vmetc.com/2008/05/26/cloning-a-running-virtual-machine-using-the-service-console/).
