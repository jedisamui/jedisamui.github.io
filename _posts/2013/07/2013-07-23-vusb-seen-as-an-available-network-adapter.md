---
layout: post
title: "vusb seen as an available network adapter?"
summary: "Don't be surprised by the vusb adapter"
author: samui
thumbnail: /images/2013/07/IBM_CDC_vSwitch0.png
permalink: /blog/vusb-seen-as-an-available-network-adapter/
date: "2013-07-23"
category: [esx, howto, vmware, commandline, troubleshooting, vmnic]
published: true
---

Recently, I had to install ESXi 4.1 on an IBM x3850 X5. Once the install was completed and I started configuring the host, I noticed something odd. I had a new vswitch and a network adapter that was defined as vusb0.

WTF? At first, I thought I was imagining things. I opened the network adapters view, and sure enough there it was. I thought how could this be. Of course, I've never seen this before, so a little googling was in order. Lo and behold, I found this little jewel ([VMware Forum Post: "Extra NICs showing up as vusb?"](http://communities.vmware.com/message/1734714){:target="_blank"}). Apparently, this is something that seems to be common with the IBM uEFI BIOS and ESX/ESXi4 and can cause issues with the console and the vmkernel ([as referenced here from VMFAQ](http://vmfaq.com/?View=entry&EntryID=70){:target="_blank"}).

**Here's how to fix it.**

Option A) Follow these instructions: (This is what I did to correct)

1. Boot into the BIOS, then select "System Settings".

![pic1](/images/2013/07/pic1.png)

5. Select "Devices and I/O Ports".

![pic2](/images/2013/07/pic2.png)

9. Locate "PCIe Gen1/Gen2 Speed Selection" and set the speed to "Gen1".

![pic3](/images/2013/07/pic3.png)

13. Save the setting change and reboot.

or

Option B) Update the BIOS to version 1.03 or newer. I grabbed this from [L. Troen's blog, VMFAQ](http://vmfaq.com/?View=entry&EntryID=70){:target="_blank"}.

Updating the uEFI BIOS can be done as follows:

1. Put the host in maintenance mode
2. Put the new uefi bios file somewhere on the local file system of the service console (will not run from vmfs)
3. Run the command: ./ibm\_fw\_uefi\_d6e128a\_linux\_32-64.bin -u
4. It will fail with the message "Error flashing firmware: 3"
5. Run the same command again: ./ibm\_fw\_uefi\_d6e128a\_linux\_32-64.bin -u
6. This time it flashed successfully.
7. This install creates a new virtual switch named IBM\_CDC\_vSwitch0. This will break VMware HA so you should remove it before leaving maintenance mode.
8. If you didn't enter maintenance mode before flashing, this new virtual switch (the one that breaks HA) will not be visible until you reboot the host.

**Note:** With newer BIOS revisions, you do not need to run the BIOS update command twice.
