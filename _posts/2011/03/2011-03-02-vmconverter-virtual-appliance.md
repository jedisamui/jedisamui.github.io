---
layout: post
title: "VMconverter Virtual Appliance"
summary: "Learn how to create a VM Converter appliance"
author: samui
thumbnail: /images/2012/03/VMWare-Converter-01.jpg
permalink: /blog/vmconverter-virtual-appliance/
date: "2011-03-02"
category: [esx, p2v, project, vmware]
published: true
---

Last year, Duncan Epping made a [post about building a VMConverter Virtual Appliance](http://www.yellow-bricks.com/2010/02/22/creating-a-vmware-converter-appliance/){:target="_blank"} using [suse studio](http://susestudio.com/){:target="_blank"}. It was a complete walk-thru from beginning to end on how to build the virtual app. I built one and it worked flawlessly. I used that little booger to run numerous P2V (Physical to Virtual) and V2V (Virtual to Virtual) conversions. The unfortunate thing is that when I left my former employer, I didn't keep the VM, and now I have a need to recreate it.

Well never fear.... I fired up Duncan's page and figured I'd just redo this all over again. Well... a lot has changed in a year. :) I'm going to borrow from Duncan's page and show the changes that I ran into, and what I did to resolve those challenges.

- Go to [susestudio.com](http://susestudio.com/){:target="_blank"} and open an account (Or you can log in with a google or facebook account)
- Click “Create New Appliance”
- Select “GNOME Desktop” and click “Create Appliance”
![susestudio](/images/2011/03/susestudio.jpg) - Change the name of the appliance to something that makes a bit more sense…
- Under the “Software Tab,” add “File Roller”, “GCC”, and “libpng12-0”. These files are needed to install VMware tools and VMConverter.
- Go to the “Configuration Tab” and click on “Appliance”
- Increase the memory to 1024MB for a better running appliance
- [Download VMware Converter Standalone](https://www.vmware.com/tryvmware/?p=converter){:target="_blank"} for Linux and add it as a file in the “Overlay Files” tab
- When uploading is finished select a folder where the tar.gz file should be extracted, I picked “/vmconverter”
- Click on the “Build” Tab, select “OVF” (for ESX environments) and wait for it to complete.
![build](/images/2011/03/build.jpg)

I found that if I chose (.vmdk) for the file format, that the file format was built for VMware Workstation and not the virtual infrastructure. I would then have to run vmconverter to import this new virtual app into my vCenter. Using the OVF format, I can just import it directly.

- Once imported in vCenter, power up the new VM.
- Install VMware Tools: Right click “VM” from the menu bar, and “Install/Upgrade VMware Tools”
- Open a Gnome Terminal session within the VM and type:

```
		su -
		cd /media/VMware Tools
		tar -C /tmp -xvf
		/tmp/vmware-tools-distrib/vmware-install.pl 		

```

*Stick to the defaults

- Now install VMware Converter:

```
		cd /vmwconverter/vmware-converter-distrib
		./vmware-install.pl

```

*Again, stick to the defaults.

- You can add an icon to the desktop by right clicking the desktop and selecting “Create Launcher”
- Select “/usr/bin/vmware-converter-client”
- And add the correct icon! (/usr/share/icons/vmware-converter.png)

By default, the Video displays at "800x600". That just doesn't work for me. Here's what I did to adjust the video settings to 1024x768. Since Suse 11.3 doesn't include SAX2 (which is what I used before to edit the resolution), I had to edit the monitor, screen and device sections from the appropriate files located in /etc/X11/xorg.conf.d.

Here's what my current files look like:

```
# /etc/xorg.conf.d/50-device.conf
######################################
Section "Device"
Identifier "Default Device"
BoardName "VMWARE0405"
Driver "vmware"
VendorName "VMWare Inc"
EndSection

# /etc/xorg.conf.d/50-monitor.conf
######################################
Section "Monitor"
Identifier "Default Monitor"
HorizSync 1-10000
VertRefresh 1-10000
ModelName "LCD"
VendorName "VMWare"
Option "PreferredMode" "1024x768"
ModeLine "1024x768"
EndSection

# /etc/xorg.conf.d/50-screen.conf
######################################
Section "Screen"
Identifier "Default Screen"
Device "Default Device"
Monitor "Default Monitor"
DefaultDepth 24

SubSection "Display"
Depth 15
Modes "1024x768"
EndSubSection
SubSection "Display"
Depth 16
Modes "1024x768"
EndSubSection
SubSection "Display"
Depth 24
Modes "1024x768"
EndSubSection
SubSection "Display"
Depth 8
Modes "1024x768"
EndSubSection
EndSection

```

![vmconverter](/images/2011/03/vmconverter-1024x867.jpg) At this point, as Duncan said, "your appliance is good to go." You can now assign it an IP and hostname and use it everywhere in your virtual infrastructure.
