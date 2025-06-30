---
layout: post
title: "Building the lab 1: Starting from scratch"
summary: "Let's build out our lab."
author: samui
date: "2018-01-25"
category: [lab, project]
thumbnail: /images/2018/01/Screen-Shot-2018-01-31-at-9.33.59-AM.png
permalink: /blog/building-out-the-lab-1-starting-from-scratch/
published: true
---

.....again.

Numerous blogs exist that detail out how to install ESX and vCenter, etc. I don't need to add one more to the ever growing list of community experts, nor do I want to invalidate any of them. Besides I utilize a bunch of these blogs for my own purposes, in addition to using them in my day job.

You will see though, that my home lab build deviates from a normal infrastructure architectural reference that you would likely see within your own environment. I try to incorporate architectural best practices where I can, however, the first layer of this design has quite a few caveats. In every design, you will want to try and identify the big four; Assumptions, Constraints, Risks, and Requirements. I'll try and break down a few from my design.

**Assumptions**

- I'm not going to screw this up.
- There is enough compute resources on hand to do what I need.

**Constraints**

- There are only three physical hosts in my environment.
- Storage will be NFS Storage.
- All work must occur during off hours.
- Only one host will be used for management.

**Risks**

- No backup solution - rebuilds will happen. (how many times have you heard that?)
- A single UPS for power outages.
- Disconnecting the internet between the hours of 7a - 12a will anger the natives.
- Only one host will be used for management.

**Requirements**

- Install vCloud Director.
- Basic functioning vApps (We call them PODs in Hands On Labs) that I can deploy quickly.
- Reuse subnets within the PODs.
- PODs will be consist of Layer 1 & 2 nested virtual machines.

### **Conceptual Design**

Of course, in a real design, this list will be long and highly detailed. But from this list, I can start to formulate a conceptual design.

![Conceptual Design](/images/2018/01/Screen-Shot-2018-01-24-at-5.57.48-PM-300x181.png)


The logical design would come next where you would add more details but still not get too technical. I'm going to jump ahead and discuss my management cluster.

### **Management Cluster**

Technically, this isn't a cluster. I'm logging directly into the host and don't have a management vCenter. My management host is an Intel NUC: NUC6i3SYH. The same model that [William Lam uses in his own Home Lab](https://www.virtuallyghetto.com/2016/03/vsan-6-2-vsphere-6-0-update-2-homelab-on-6th-gen-intel-nuc.html){:target="_blank"} (as well as many other bloggers). I used a similar Build of Materials that he has listed on his site.

### **Build Of Materials**

- [Intel NUC 6th Gen NUC6i3SYH](https://www.amazon.com/gp/product/B018NSAPIM/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B018NSAPIM&linkCode=as2&tag=vghetto-20&linkId=13d6052732123139715b2d46374f8590){:target="_blank"}
- [(2) Crucial 16GB DDR4 RAM](https://www.amazon.com/gp/product/B015YPB6HQ/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B015YPB6HQ&linkCode=as2&tag=vghetto-20&linkId=f61f78f14686d1c400d57802bbd5217a){:target="_blank"}
- (1) 500GB SATA Drive
- [(1) StarTech USB 3.0 to Gigabit Ethernet Adapter](https://www.amazon.com/gp/product/B0095EFXMC/ref=oh_aui_detailpage_o05_s00?ie=UTF8&psc=1){:target="_blank"}

A single NUC with 32GB of RAM and two NICs. Yep, that's my Management Cluster, er... Management Host. I installed the latest version of ESXi 6.5 build 5969303 (at the time, it was ESXi 6.5 U1). The NUC has only one built-in gigabit ethernet NIC. If I tried to run VMs over NFS storage, then I may wind up getting in trouble. While I did get the StarTech USB-Ethernet adapter ([again, thank you Mr Lam](https://www.virtuallyghetto.com/2016/11/usb-3-0-ethernet-adapter-nic-driver-for-esxi-6-5.html){:target="_blank"}),I decided that I did not want (or need) to have bandwidth issues. Therefore, all of the VMs were going to be installed on Local Storage. Normally, I would be running all machines off of shared storage, but for the initial go around - this is the setup.

Knowing that this management cluster/host was going to be strapped for RAM, I had to make some design decisions. I started to play with the numbers. Since my plan is to build vCloud Director as my base starting point, I had to determine what infrastructure components would be needed.

**Windows 2016 Active Directory Domain Controller (4GB RAM) Windows 2016 SQL Server (4GB RAM) NSX Manager (16GB RAM) vCloud Director Cell (5GB RAM) vCenter Server Appliance (10GB RAM)**

I'm sure, like me, you have already realized -- without doing the math, that I don't have enough RAM. I have a couple of choices - I could buy another NUC (which I might), shorten the amount of RAM on the VMs and deal with the slowness, combine management and payload into one cluster and share the resources everywhere.

After careful consideration, I decided to combine the DC and the SQL box into one virtual machine. That cut me down to 4 virtual machines. (I thought about dropping the Domain Controller since the domain is not going to be utilized within the lab, but I needed a SQL box for vCD, and other Domain services, such as an internal CA for other infrastructure services that run outside the lab.

I know I could have used Postgres instead of SQL.... but I'll admit, our documentation isn't all that clear on how to set that up OOTB. (I may upgrade and migrate to Postgres later.) The vCenter server is to manage the Compute Resources. I thought about moving that machine into the Compute Cluster. But in the end, I want to use all of my resources in the Compute Cluster and not have to share and have an inception scenario during any outages or troubleshooting.

Once I started finalizing on the numbers, I was left with this: **Windows 2016 Active Directory Domain Controller & SQL Server (6GB RAM) NSX Manager (16GB RAM) vCloud Director Cell (5GB RAM) vCenter Server Appliance (10GB RAM)**

This amount of RAM is still over the threshold for the NUC. But allocated versus consumed should be low enough that I do not constrain the host. If I need to lower the amount of RAM on a VM, I can adjust each VM individually. With the risks identified, I started to build out my management resources.

### **Domain Controller**

First, I needed to build a Windows Server 2016 Virtual Machine to act as the Domain Controller, Certificate Authority, and SQL Server.

Luckily for me, I found the following websites already exist and walked me through setting up the basics.

- Active Directory - [Link](https://ittutorials.net/microsoft/windows-server-2016/setting-up-active-directory-ad-in-windows-server-2016/){:target="_blank"}
- Certificate Authority - [Link](http://www.careexchange.in/install-and-configure-certificate-authority-in-windows-server-2016/){:target="_blank"}
- SQL Server Installation - [Link\*\*](https://lonesysadmin.net/2011/10/06/how-to-install-microsoft-sql-server-2008-r2-for-vmware-vcenter-5/){:target="_blank"}

**While this is an older article, the steps still work for SQL 2016. Some of them may just look differently cosmetically._

Once the Domain Controller was installed and had SQL Services running, I then focused on setting up the vCSA. I'll use the next post to dive into the vCenter Server Appliance and NSX.
