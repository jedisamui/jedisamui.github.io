---
layout: post
title: "Multiple Cores for VMs"
summary: "Learn how to enable multiple cores"
author: samui
thumbnail: /images/2011/03/cpu-cores.png
permalink: /blog/multiple-cores-for-vms/
date: "2011-03-18"
category: [esx, vmware-kb]
published: true
---

Is it possible? To have multiple cores per CPU on a VM?

I recently stumbled upon a [VMware KB #1010184](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1010184){:target="_blank"} that says it is. Holy cow!

Here's the kicker. For this to work, to have to meet the following requirements:

- Running ESX 4.1 or ESXi 4.1
- Virtual Machines on Hardware Version 7 ONLY
- Virtual Machine Operating Systems that support the multi-core vCPU feature
_Most newer OSes support this feature (ie. XP and above, RHEL 5+)_

In the past, a vCPU assigned to a VM appeared to the OS as a single core CPU. You could add more vCPUs to a VM, but they would show as additional single core CPUs.

ESX 4.1 now has a feature that will allow you to set the number of cores per virtual socket within the VM.

For example:

Create an 8 vCPU virtual machine and set cpuid.coresPerSocket = 2. Window Server 2003 SE running in this virtual machine now uses all 8 vCPUs. Under the covers, Windows sees 4 dual-core CPUs. The virtual machine is actually running on 8 physical cores.

To implement this feature: 
1) Power off the virtual machine. 
2) Right-click on the virtual machine and click Edit Settings. 
3) Click Hardware and select CPUs. 
4) Choose the number of virtual processors. 
5) Click the Options tab. 
6) Click General, in the Advanced options section. 
7) Click Configuration Parameters. 
8) Include cpuid.coresPerSocket in the Name column. 
9) Enter a value (try 2, 4, or 8) in the Value column.

Note: Ensure that cpuid.coresPerSocket is divisible by the number of vCPUs in the virtual machine. That is, when you divide cpuid.coresPerSocket by the number of vCPUs in the virtual machine, it must return an integer value. For example, if your virtual machine is created with 8 vCPUs, coresPerSocket can only be 1, 2, 4, or 8.

The virtual machine now appears to the operating system as having multi-core CPUs with the number of cores per CPU given by the value that you provided in step 9.

10) Click OK. 
11) Power on the virtual machine

[Read that VMware KB #1010184 here:](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1010184){:target="_blank"}
