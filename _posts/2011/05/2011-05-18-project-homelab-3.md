---
layout: post
title: "Project: Homelab"
summary: "Covering lab storage"
author: samui
thumbnail: /images/2011/05/100_1427.jpg 
permalink: /blog/project-homelab-3/
date: "2011-05-18"
category: [howto, lab, project, storage]
published: true
---

### PART 3: Storage I - NAS

I haven't written about my homelab project in some time. So I figured now would be a good time to write about my storage setup - since it is currently undergoing an upgrade. For this part, I will focus on how I am upgrading my current NAS unit.

I am currently using a Dell PowerEdge 2850 running OpenFiler v2.3 with 1.3TB (6x300G: RAID5) and a Dell PowerVault 745N Win2003 Storage Server 1TB NAS (4x250G: RAID5). The PE2850 serves a dual purpose - it houses VMs for my virtual environment and additional iSCSI "drives" for the PV745N. I am wanting to grow and replace my PV745N as it is close to full. In addition, I want to cut back on the amount of heat I generate and the power draw from the PE2850.

I knew that I wanted to have at least 4TB of storage when all was said and done. Several days was spent researching my options. I looked at a number of 4TB NAS units to replace the PV745N. The NetGear ReadyNAS NV+ (http://www.newegg.com/Product/Product.aspx?Item=N82E16822122010){:target="_blank"} was definitely in consideration for my plans. I liked that it could allow me to upgrade my drives down the road using the X-RAID function without having to do a forklift upgrade. (Eventually, this may show up under my tree - *hint, hint Santa*.)

Performance is not that big of an issue as this is my homelab. Therefore, I'm not looking for insane IOPs, nor super fast anything. Most of my data is static, however, I want plenty of room for media, backups, etc.

![Hitachi Hard Drives](/images/2011/05/100_1434-300x225.jpg)
Hitachi Hard Drives
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}

Instead of spending money on a new device, I decided that I would just recycle a machine I had lying around. I pulled out a Dell PowerEdge 800 and added a few components. I removed the existing 80GB SATA Drive and replaced it with (4) 1TB Hitachi 7200RPM SATA II Drives. I wanted to connect these new drives to a decent hardware RAID card, but unfortunately, I couldn't afford to purchase one right now. From my choices at my local MicroCenter, I did some research and picked up a HighPoint RocketRAID 1740 PCI Card. It wasn't until AFTER I got the card home and installed, that I discovered it did software RAID instead of Hardware RAID. (Phooey!) Had I of known this previous to the purchase, I would have stuck to the onboard SATA ports and let the OS do software RAID.

Once I had the gear together, I started to compare what OS to run on the machine. I looked at OpenFiler v.2.99, FreeNAS 8.0, ubuntu 10+, and Windows Server Solutions. I have been running Windows Home Server as a virtual machine for a little while. One thing I like about it is the easy built-in backup solution. As most IT guys know, the reason why we choose some products over others has little to do with advanced feature sets, purchase discounts, etc. Usually, we look at what we need, versus what we want to teach our user-base. In my case, this storage unit was going to be replacing the PV745N NAS unit I was using for media storage (movies, mp3s, and photos). For that reason, I did not want to have to retrain my user-base (ie., my wife) on how to connect to her "shared drive" to get her music, pics, etc. As well, I did not want to have some elaborate method that I knew she would not use for means of backups. My choice was now narrowed to Windows Home Server. And I stuck to what I was using previously. :)

(I realized AFTER I installed Windows Home Server, that a newer version (WHS 2011) was released in April. I will have to check and see if I want to upgrade to it.)

The before and after comparison: **BEFORE** "Recycle" Dell PowerEdge 800 Intel P4 2.8Ghz 800FSB CPU 1GB DDR2 RAM (1) 80GB Matrox 5400RPM SATA I HD

![Dell PowerEdge 800](/images/2011/05/100_1442-300x182.jpg)
Dell PE800
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"} 

**AFTER** "She-Hulk" Dell PowerEdge 800 Intel P4 3.06Ghz 533FSB CPU 4GB DDR2 RAM (1) 160GB Seagate 7200RPM SATA II HD (4) Hitachi 7200RPM 1TB SATA II HDs HighPoint RocketRAID 1740 PCI RAID Controller

I'm sure you are wondering why the change in some of the parts. I changed the CPU as the replacement is 64bit capable. I added more RAM (4GB Total - as this was the max for the board). I also added an additional HD. This drive is for the OS and other non-essential data. Now known as "She-Hulk", the PE800 was brought to life. She purrs and runs just fine. I wonder if I should remove her shell and paint her green. That would be interesting.

PART 3: Storage II will be coming soon. I have plans for the PV745N.
