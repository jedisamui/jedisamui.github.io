---
layout: post
title: "ESXi BIOS Power Settings Best Practice"
summary: "Recommendations for a physical host's bios settings"
author: samui
thumbnail: /images/2014/02/bios-2-300x168.jpg
permalink: /blog/esx-bios-power-settings-best-practices/
date: "2014-02-05"
category: [esx, vmware, troubleshooting]
published: true
---

There is a movement to change the best practice regarding the bios power settings on an ESXi host. In the past, you would set the power settings in the BIOS to max and be done with it. You did this because previous versions of vSphere did not work well with the various C-States of the processor (ie., When the workload on the host would drop, the CPU would drop to a lower power setting. When it would go to wake and draw the CPU back to full power, the host would crap all over itself.).

vSphere 5.5 introduced a new feature that allows VMware to better utilize the C-States of some of the newer processors. Thus allowing the CPUs to change power and speed states without affecting the performance or behavior of the Host. (YEAH!!)

I know we are happy about this, but not so fast...

[This link](http://blogs.vmware.com/vsphere/2014/01/bios-power-policies-affect-performance.html){:target="_blank"} describes the behavior and a couple of performance benefits from the change -- however, it should also be noted, that this change is dependent on the type of workloads in your environment. If you have a heavy IO Intense workload or a time sensitive workload, you may still want to have that host on max power, high performance, etc.

[http://blogs.vmware.com/vsphere/2014/01/bios-power-policies-affect-performance.html](http://blogs.vmware.com/vsphere/2014/01/bios-power-policies-affect-performance.html){:target="_blank"}
