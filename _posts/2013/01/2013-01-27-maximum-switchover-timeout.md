---
layout: post
title: "Maximum Switchover Timeout"
summary: "VMs timeout during svmotion"
author: samui
thumbnail: /images/2013/01/svmotion.png 
permalink: /blog/maximum-switchover-timeout/
date: "2013-01-27"
category: [esx, howto, vmware, troubleshooting, vcenter, vmware-kb]
published: true
---

I recently ran into an issue where I was having to svmotion some rather large VMs (1-2TBs) that stretched over multiple datastores. During the svmotion, the VMs would time out at various percentages presenting this error. 

Consulting with Prof. G (Google) presented a [VMware KB Article: 1010045](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1010045){:target="_blank"}. That article states; "This timeout occurs when the maximum amount of time for switchover to the destination is exceeded. This may occur if there are a large number of provisioning, migration, or power operations occurring on the same datastore as the Storage vMotion. The virtual machine's disk files are reopened during this time, so disk performance issues or large numbers of disks may lead to timeouts." Yep, this was me. I was having to svmotion VMs from one datastore to another during a vsphere 5 upgrade.

The KB Article discusses adding a timeout value, called "fsr.maxSwitchoverSeconds" to the VM's VMX file to prevent the timeout.

To modify the fsr.maxSwitchoverSeconds option using the vSphere Client:

1.) Open vSphere Client and connect to the ESX/ESXi host or to vCenter Server. 
2.) Locate the virtual machine in the inventory. 
3.) Power off the virtual machine. 
4.) Right-click the virtual machine and click Edit Settings. 
5.) Click the Options tab. 
6.) Select the Advanced: General section. 
7.) Click the Configuration Parameters button.

Note: The Configuration Parameters button is disabled when the virtual machine is powered on.

8.) From the Configuration Parameters window, click Add Row. 
9.) In the Name field, enter the parameter name:

fsr.maxSwitchoverSeconds

10.) In the Value field, enter the new timeout value in seconds (for example: 150). (I chose a value of 200.) 
11.) Click the OK buttons twice to save the configuration change. 
12.) Power on the virtual machine.

From personal experience, this was a homerun. It resolved my problem.
