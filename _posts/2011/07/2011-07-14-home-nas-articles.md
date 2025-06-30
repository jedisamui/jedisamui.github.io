---
layout: post
title: "Home NAS Articles"
summary: "Information for a home NAS"
author: samui
thumbnail: /images/2011/07/nas.png
permalink: /blog/home-nas-articles/
date: "2011-07-14"
category: [lab, project, howto, storage]
published: true
---

As you know, I wound up building a new NAS for my home lab. This new build replaced a 1TB Dell PowerVault 745N. I was going to replace the Dell PowerEdge 2850, but wound up upgrading and repurposing it (like most companies). During the build of this machine, I found a few articles that I wanted to share (In addition to "bookmarking" them for my own future references). I found these to be interesting as well as pretty informative. I'd like to add my opinion on what I found.

[**Build Your Own NAS Using FreeNAS**](http://geekyprojects.com/nas/build-your-own-nas-using-freenas/){:target="_blank"} **The Likes:** I like this Article as it is very detailed - with lots of screenshots and pics of the actual hardware components, etc. The ability to install FreeNAS on a USB drive, or in the article's case of a CF Card, is very inticing. As this allows you to take full advantage of the drives you have without loosing any space for the OS.

**The Dislikes:** I've actually used FreeNAS in the past in a corporate lab environment. Maybe it was the version I was using, maybe it was the hardware that we were running it on. But I had issues were the storage would disconnect repeatedly from the ESX hosts it was connected to. Since that lab was 2.5 hours remote from the office that I was in, I would get super frustrated when the disconnection would occur. Hopefully, FreeNAS fixed this bug - if it was a FreeNAS bug. (I left that job before discovering why the disconnections occurred.) One other thing to point out in this Article; is that while the RAID card listed is a good solid HW RAID card, it has a limit of 2TB for the max volume size. Using (4) 500GB SATA II Drives in a RAID 5, would get you 1.5TB RAW/1.3TB useable space. So forget about popping (4) 1TB drives onto that array.

[**How to configure OpenFiler v2.3 iSCSI Storage for use with VMware ESX.**](http://www.techhead.co.uk/how-to-configure-openfiler-v23-iscsi-storage-for-use-with-vmware-esx){:target="_blank"} **The Likes:** This is the Article I referenced most during my build. Even though I was using OpenFiler v2.99, many of the directions were the same. Simon really covered all the bases well in this article, from setting up the OpenFiler and presenting the LUNs, to configuring the iSCSI connection within the ESX hosts.

**The Dislikes:** I will add, that if you do not follow the OpenFiler part of the instructions correctly (in otherwords, you try and jump ahead), you may wind up having issues. One of the biggest issues you may run into would be that the LUNs aren't mapped for the ESX host. You won't find this out until you go to scan the HBAs for the LUNs and nothing is there to add within the ESX host. Also, several people commented their concerns that OpenFiler couldn't do VMotion as it could not support simultaneous connections. I'm here to tell you, this is not the case. My lab environment consists of three ESX hosts, and I have no problems vmotioning VMs, and shuffling files if needed in this array. (So overall, there really isn't any dislikes.)

[**Build Your Own Fibre Channel SAN For Less Than $1000**](http://www.smallnetbuilder.com/nas/nas-howto/31485-build-your-own-fibre-channel-san-for-less-than-1000-part-1?start=3){:target="_blank"} I have no likes or dislikes to this article, rather I found this article to be an awesome read. Discovering that OpenFiler v2.99 supports FC now is amazing. This makes the possibility of building a cheap-ish fiber-SAN for your home or small business a little more feasible. As I dug deeper and deeper into this article, I decided to hit a few more links on this idea.

[Openfiler 2.99 FiberChannel Setup](http://tomlecluse.be/blog/20110619/openfiler-299-fiber-channel-setup){:target="_blank"}

Fiber Channel Setup Step-by-Step (From the OpenFiler Forums) [https://forums.openfiler.com/viewtopic.php?pid=24056](https://forums.openfiler.com/viewtopic.php?pid=24056){:target="_blank"} 
[https://lists.openfiler.com/viewtopic.php?id=3422](https://lists.openfiler.com/viewtopic.php?id=3422){:target="_blank"}

[Fiber-Channel Cables](http://www.fibercables.com/product_info.php?products_id=2933){:target="_blank"}

In addition to all of this goodness, I started to ponder the idea of cutting down my servers to just one power supply - as really, I have no need for the redundant power in my home lab. Which led me to this interesting article.

[**Cost savings through removing server power supplies**](http://www.delltechcenter.com/page/Cost+savings+through+removing+server+power+supplies){:target="_blank"}

I wound up leaving the redundant power supplies in the server. I couldn't find a way to turn off any settings for it, and got tired of looking at amber lights informing me of the missing PS. In addition, there isn't too much heat or power saved from removing them.
