---
layout: post
title: "Installing VMware Tools in a Linux Virtual Machine"
summary: "Walkthrough on how to install VMTools on linux"
author: samui
thumbnail: /images/2010/05/vmware-tools.png
permalink: /blog/installing-vmtools-in-a-linux-vm/
date: "2010-05-11"
category: [esx, linux, vmtools, vmware]
published: true
---

This section explains how to install VMware Tools in a Linux virtual machine.

To install VMware Tools in a Linux, FreeBSD, or Solaris Virtual Machine

1. Power on the virtual machine.

2. After the guest operating system has started, prepare your virtual machine to install VMware Tools.

**Choose VM > Install VMware Tools.**

The remaining steps take place inside the virtual machine. Note: You can install VMware Tools either from a terminal in an X window session or in text mode.

3. As root (su -), mount the VMware Tools virtual CD-ROM image, change to a working directory (for example, /tmp), uncompress the installer, and unmount the CD-ROM image.

Note You do not use an actual CD-ROM to install VMware Tools, and you do not need to download the CD-ROM image or burn a physical CD-ROM of this image file. The VMware Server software contains an ISO image that looks like a CD-ROM to your guest operating system. This image contains all the files needed to install VMware Tools in your guest operating system.

a) Using the Tar Installer on Linux Guests: Some Linux distributions use different device names or organize the /dev directory differently. If your CD-ROM drive is not /dev/cdrom or if the mount point for a CD-ROM is not /mnt/cdrom, modify the following commands to reflect the conventions used by your distribution.

Also, some Linux distributions automatically mount CD-ROMs. If your distribution uses automounting, do not use the mount and umount commands below. You still must untar the VMware Tools installer to /tmp.

**mount /dev/cdrom /mnt/cdrom cd /tmp tar zxf /mnt/cdrom/vmware-linux-tools.tar.gz umount /mnt/cdrom Go to step 4.**

b) Using the RPM Installer on Linux Guests: Some Linux distributions use different device names or organize the /dev directory differently. If your CD-ROM drive is not /dev/cdrom or if the mount point for a CD-ROM is not /mnt/cdrom, modify the following commands to reflect the conventions used by your distribution.

Also, some Linux distributions automatically mount CD-ROMs. If your distribution uses automounting, do not use the mount and umount commands below.

**mount /dev/cdrom /mnt/cdrom cp /mnt/cdrom/vmware-linux-tools-.i386.rpm /tmp rpm -Uhv /tmp/vmware-linux-tools-.i386.rpm umount /mnt/cdrom**

where is the build number of the VMware Server release. **Go to step 6.**

c) Solaris Guests: The Solaris volume manager—vold—mounts the CD-ROM under /cdrom/vmwaretools. If the CD-ROM is not mounted, restart the volume manager using the following commands:

**/etc/init.d/volmgt stop /etc/init.d/volmgt start**

After the CD-ROM is mounted, use the following commands to extract VMware Tools.

**cd /tmp gunzip -c /cdrom/vmwaretools/vmware-solaris-tools.tar.gz | tar xf -Go to step 4.**

d) FreeBSD Guests: Some FreeBSD distributions automatically mount CD-ROMs. If your distribution uses automounting, do not use the mount and umount commands below. You still must untar the VMware Tools installer to /tmp.

**mount /cdrom cd /tmp tar zxf /cdrom/vmware-freebsd-tools.tar.gz umount /cdrom**

4. Run the VMware Tools installer.

**cd vmware-tools-distrib ./vmware-install.pl**

5. Answer the questions about default directories.

6. Run the configuration program.

**vmware-config-tools.pl**

7. To change your virtual machine’s display resolution, answer yes, and enter the number that corresponds to the desired resolution.

8. Log off of the root account.

**exit**

9. Start X and your graphical environment. If you installed VMware Tools in an X windows session, restart X windows.

10. In an X terminal, launch the VMware Tools background application.

**vmware-toolbox &**

You can run VMware Tools as root or as a normal user. To shrink virtual disks or to change any VMware Tools scripts, you must run VMware Tools as root (su -).

Note: Always run vmware-toolbox in the guest operating system to ensure you have access to all VMware Tools features, such as copy and paste and mouse ungrab for operating systems for which X display driver is not available.
