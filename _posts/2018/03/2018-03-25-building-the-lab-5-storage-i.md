---
layout: post
title: "Building the lab 5: Storage I"
summary: "Continuing the lab build out. This time focusing on the storage."
author: samui
date: "2018-03-25"
category: [lab, howto, project, storage]
thumbnail: /images/2018/03/XPEnology-225x300.jpeg
permalink: /blog/building-the-lab-5-storage/
published: true
---

This part of my homelab rebuild will touch on something interesting... storage options. Knowing that my lab is going to be a couple levels of nesting, I wanted to look at the different options that are out there.

For years, I have had and use a Synology DS 1512+ storage array. This thing has been running like gangbusters. It serves multiple purposes from backups & fileshares, to NFS storage for the virtual environment.

Over the years, I have upgraded the drives from 1TB to 2TB, to 3TB. Because of this, I have a few drives lying around not in use. I thought that maybe I could spin up a FreeNAS or OpenFiler box for iSCSI within the environment. By creating differing storage arrays, I could introduce Storage Policies within the environment for potential feature consumption down the road.

As I explored the various options out there, I discovered many simulators from various vendors: 3PAR, EMC, NetApp. In addition to these, you have the free options as mentioned above: OpenFiler, FreeNAS, etc. But I also stumbled across this jewel..... XPEnology.

I'm sure you are wondering -- What is XPEnology? Xpenology is a bootloader for Synology’s operating system which is called DSM (Disk Station Manager) and is what they use on their NAS devices. DSM is running on a custom Linux version developed by Synology. Its optimized for running on a NAS server with all of the features you often need in a NAS device. Xpenology creates the possibility to run the Synology DSM on any x86 device like any pc or self-built NAS.

You read that right, it is possible to run XPEnology on bare metal. XPEnology runs as a faux Synology NAS.

Now, before you continue, you should know this. XPEnology is not supported or owned by Synology, micronauts, or anyone in their right mind. The links provided are ONLY for experimentation purposes and should not be used in any production or development environment. It is very possible you could lose all data and put yourself in jeopardy. If you need reliable, dependable storage, then buy a [Synology NAS](https://www.synology.com/en-us/products){:target="_blank"}.

**PROCEED AT YOUR OWN RISK!**

Alex Lopez at ThinkVirtual has a good write up on how to create a Synology Storage VM [https://ithinkvirtual.com/2016/04/30/create-a-synology-vm-with-xpenology/](https://ithinkvirtual.com/2016/04/30/create-a-synology-vm-with-xpenology/){:target="_blank"}

Alex's write up was based on Erik Bussink's build, found here. [https://www.bussink.ch/?p=1672](https://www.bussink.ch/?p=1672){:target="_blank"}

The original XPEnology website walkthrough on how to create a Synology Storage VM. [http://xpenology.me/installing-dsm-5-1-vmware-esxi5-5u1/](http://xpenology.me/installing-dsm-5-1-vmware-esxi5-5u1/){:target="_blank"}

The original Xpenology [website](http://xpenology.me/xpenoboot-5-2-5644-5-released/){:target="_blank"} has become a ghost-town. I'm not sure if it is being maintained, or if the original creator(s) just don't have the time to update it any longer. The last updates came out around DSM 5.2-5644.5 (so a while ago). However, [the XPEnology forums](https://xpenology.com/forum/forum/97-answered-questions/?page=3){:target="_blank"} will provide all kinds of glorious information from the wealth of knowledge within the community.

Additionally, you can get more information from this [new XPEnology info site](https://xpenology.org/){:target="_blank"}. They also have a pretty good walk-through for a storage VM. The video tutorial even covers how to setup ESXi 5.1 ([http://xpenology.org/installation/](http://xpenology.org/installation/){:target="_blank"}).

![](/images/2018/03/XPEnology-225x300.jpeg) 

While having a storage VM is great, I think having XPEnology on baremetal is even better. As you read and research how to do this, you are going to discover that it involves grabbing files stashed all over the internet -- files ranging from a bootloader to PAT files. Make sure that you read EVERYTHING. I reutilized some hardware and some of my old synology drives and built a XPEnology server on bare metal.

I booked marked this site ([https://github.com/XPEnology/JunLoader/wiki/Installing-DSM-6.0-Bare-Metal](https://github.com/XPEnology/JunLoader/wiki/Installing-DSM-6.0-Bare-Metal){:target="_blank"}) as it provides a pretty good walkthrough on how to create a bootable USB drive for the XPEnology OS. I also found this one ([http://blog.pztop.com/2017/07/24/Make-Xpenology-boot-loader-1.02b-for-DSM-6.1-on-Ubuntu/](http://blog.pztop.com/2017/07/24/Make-Xpenology-boot-loader-1.02b-for-DSM-6.1-on-Ubuntu/){:target="_blank"}). For those of you, like myself, who are on a MAC.... you may need this nugget ([https://xpenology.com/forum/topic/1753-create-a-bootable-usb-on-os-x/](https://xpenology.com/forum/topic/1753-create-a-bootable-usb-on-os-x/){:target="_blank"}).

Again, I would like to say, if you need reliable and dependable storage, go purchase a real storage array.
