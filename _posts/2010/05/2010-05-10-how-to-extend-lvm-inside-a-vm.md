---
layout: post
title: "How to Extend LVM Inside a VM"
summary: "Reference: Extend LVM inside a linux VM" 
author: samui
thumbnail: /images/2010/05/linux-lvm.png
permalink: /blog/how-to-extend-lvm-inside-a-vm/
date: "2010-05-10"
category: [esx, linux, vmware]
published: true
---

These instructions have been applied to RedHat and CentOS. I have seen some instances, where these instructions broke the VM. (So I make no promises. They worked for me.)

Article: How to Extend LVM Inside a VM

1. Gracefully shutdown virtual machine. 
2. SSH into ESX Server as 'root' and navigate to the datastore containing the vmdk that you need to extend. 
3. vmkfstools -X G virtual.vmdk (Enter the total size of the new HD - 10G) 
4. Start virtual machine and open a terminal window. 
5. fdisk /dev/sda (if only 1 disk on linux) 
6. in fdisk, create new partition with new freespace (n,new partition#,start,stop) 
7. in fdisk, change new partition type to 8e (linux lvm)(t,new partition#,8e) 
8. in fdisk print out partition table to verify (p) 
9. in fdisk write partition and exit (w) 
10. partprobe (shouldn't be necessary, but rereads partitions) 
11. pvcreate /dev/sda3 (if new partition is 3rd partition) 
12. vgextend VolumeGroupName /dev/sda3 (add new partition to VG) 
13. lvextend -L+1G /dev/VolGroup00/LogVol00 (assuming LogVol00 is lv you wish to grow) 
14. ext2online /dev/VolGroup00/LogVol00 (extend an ext3 partition to fill the LV) (if you want a specific size: ext2online -L10G /dev/VolGroup00/LogVol00) (if you want to grow a specific size: ext2online -L+1G /dev/VolGroup00/LogVol00) (Some OSes (CentOS) uses resize2fs: ex. resize2fs /dev/VolGroup00/LogVol00.) 
15. sync
