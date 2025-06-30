---
layout: post
title: "Stopping an Unresponsive VM"
summary: "Oops... My VM is unresponsive. How do I fix?"
author: samui
thumbnail: /images/2010/05/vm.png
permalink: /blog/stopping-unresponsive-vm/
date: "2010-05-15"
category: [esx, commandline, vmware]
published: true
---

Sometimes a Virtual Machine can't be stopped via the VIClient. The job just hangs. There are a number of options to stop your Virtual Machine from within the Service Console. Keep in mind that these are last resort options!

Stopping the virtual machine by issuing the command: **vmware-cmd /vmfs/volumes/datastorename/vmname/vmname.vmx stop** This must be done on the ESX host where the Virtual Machine is running!

If this does not work, one can issue the following command: **vmware-cmd /vmfs/volumes/datastorename/vmname/vmname.vmx stop hard** This will try to kill the Virtual Machine instantly.

A final solution is to kill the PID (process ID). Issue the following command: **ps auxfww | grep vmname** to locate the correct PID (BTW: this cannot be done via ESXTOP). The first number to appear in the output is your PID. The PID can be used to terminate the process by issuing the command: **kill -9 PID**.
