---
layout: post
title: "Installing VMware Tools in a Solaris VM"
summary: "Walkthrough of how to install VMTools in Solaris"
author: samui
thumbnail: /images/2010/05/vmware-tools.png
permalink: /blog/installing-vmware-tools-in-a-solaris-vm/
date: "2010-05-25"
category: [esx, solaris, vmtools, vmware]
published: true
---

This section explains how to install VMware Tools in a Solaris virtual machine.

1. Install VMware Tools. On the console window click "**VM**" > "**Install VMware Tools**". A virtual CDrom will be activated with the vmware tools ISO.

2. Using the console, or a terminal window. As Root do the following commands:

**# mount -rF hsfs /dev/dsk/c0t0d0s2 /mnt # cd /tmp # gunzip -c /mnt/vmwaretools/vmware-solaris-tools.tar.gz | tar xf - # umount /mnt # cd vmware-tools-distrib**

3. Run the VMware Tools Installer:

**# ./vmware-install.pl** _Creating a new VMware Tools installer database using the tar4 format._

_

Installing VMware Tools.

In which directory do you want to install the binary files? [/usr/bin]

What is the directory that contains the init directories (rc0.d/ to rc6.d/)? [/etc]

What is the directory that contains the init scripts? [/etc/init.d]

In which directory do you want to install the daemon files? [/usr/sbin]

In which directory do you want to install the library files? [/usr/lib/vmware-tools]

The path "/usr/lib/vmware-tools" does not exist currently. This program is going to create it, including needed parent directories. Is this what you want? [yes]

In which directory do you want to install the documentation files? [/usr/share/doc/vmware-tools]

The path "/usr/share/doc/vmware-tools" does not exist currently. This program is going to create it, including needed parent directories. Is this what you want? [yes]

The installation of VMware Tools 7.8.6 build-185404 for Solaris completed successfully. You can decide to remove this software from your system at any time by invoking the following command: "/usr/bin/vmware-uninstall-tools.pl".

Before running VMware Tools for the first time, you need to configure it by invoking the following command: "/usr/bin/vmware-config-tools.pl". Do you want this program to invoke the command for you now? [yes]

Stopping VMware Tools services in the virtual machine: Guest operating system daemon: done

Detected X.org version 7.2.0.

Please choose one of the following display sizes that X will start with (1 -15):

[1] "640x480" [2] "800x600" [3] "1024x768" [4] "1152x864" [5] "1280x800" [6] "1152x900" [7] "1400x900" [8] "1440x900" [9] "1280x1024" [10] "1376x1032" [11] "1400x1050" [12]< "1680x1050" [13] "1600x1200" [14] "1920x1200" [15] "2364x1773" Please enter a number between 1 and 15:

[12] 3

Starting VMware Tools services in the virtual machine: Switching to guest configuration: done Guest memory manager: done Guest operating system daemon: done

The configuration of VMware Tools 7.8.6 build-185404 for Solaris for this running kernel completed successfully.

You must restart your X session before any mouse or graphics changes take effect.

_

_You can now run VMware Tools by invoking the following command: "/usr/bin/vmware-toolbox" during an X server session._

4. To enable advanced X features (e.g., guest resolution fit, drag and drop, and file and text copy/paste), you will need to do one (or more) of the following:

1. Manually start /usr/bin/vmware-user 
2. Log out and log back into your desktop session; and, 
3. Restart your X session.

The installed vmxnet driver will be used for all vlance and vmxnet network devices on this system. Existing vlance devices will transition from the pcn driver to the vmxnet driver on the next reconfiguration reboot. You will need to verify your network settings accordingly.

If you have configured a pcn interface, the corresponding files are now renamed to use the vmxnet device name to ensure the interface will be brought up properly upon reboot. For example, the following commands were performed: **# mv /etc/hostname.pcn0 /etc/hostname.vmxnet0 # mv /etc/hostname6.pcn0 /etc/hostname6.vmxnet0 # mv /etc/dhcp.pcn0 /etc/dhcp.vmxnet0** and will cause the Solaris Service Management Facility to bring up the first vmxnetX interface using the configuration of your current pcnX interface.
