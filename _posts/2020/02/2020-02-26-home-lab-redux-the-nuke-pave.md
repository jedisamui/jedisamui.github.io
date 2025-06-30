---
layout: post
title: "Home Lab Redux: The Nuke & Pave"
summary: "Rebuild #487"
author: samui
date: "2020-02-26"
category: [lab, project, vmware, vcloud, vcloud-director, vmware]
thumbnail: /images/2020/02/D69DE98A-7413-4057-9F53-D95A84D178A6.jpeg
permalink: /blog/home-lab-redux-nuke-and-pave
published: true
---

My homelab is undergoing yet another rebuild - partially my fault, partially time to just start fresh, partially because half-way through an experiment I said ‘screw it!’ and then commenced to nuke and pave!

![](/images/2020/02/D69DE98A-7413-4057-9F53-D95A84D178A6.jpeg)

I believe that’s the circle of life for a home lab (or any lab) though. I mean, we build a “lab” (or test environment) for experimentation and exploration. So when things do go bad, it’s not production. The environment is a safe way to try something without attempting experimentation within a production environment.

The experiment that I was attempting was trying to replace the Windows Domain Controller with a Ubuntu based product called Zentyal. Zentyal 6.0 to be exact (even though 6.1 is out). What I found was that while it was a super simple setup for the domain controller, DNS, and CA Authority. Even issuing certificates was stupid simple. However, the downfall of this particular experiment was the fact that the BIND DNS that backed the DNS Server part of the build just would not do reverse DNS lookup. This caused almost every one of the VMware products to have problems. Even when I went into the command line and editing the appropriate BIND files, it still would not process reverse lookups. NSLOOKUP kept returning that the domain just did not exist. So scrap this.

This meant rebuilding the domain controller again. Ugh. I returned to an All-In-One Windows AD Domain Controller, DNS, CA server. But I went ahead and updated to Windows Server 2019. This seemed pretty straight forward (if you’ve built this on 2016, the process is basically the same.

Once the domain controller was rebuilt, I went ahead and redeployed vCenter 6.7 U3, ESXi 6.7 U3, vCloud Director 10.0.1., and NSX-V 6.4.6. Yeah, yeah... I know. The VMs that sit on the physical hardware that present the resources to be consumed by the nested environment.

![](/images/2020/02/homelab.png)

The vCloud Director build, I decided to go with the vCD Appliance versus building out a CentOS VM and manually building out Postgres. The appliance was a snap and I had vCD stood up and running within minutes. Configuring it was a bit slower..... as the HTML 5 interface and it’s desire to run the networking based on NSX-T created confusion on how to build things out. However, (going back to “V” for a moment) while vCD can utilize NSX-T, many of the features that just work in T, do not ‘just work’ in V. Where vCD and NSX-V will auto-build edges and such, much of this work would be manual in T. Eventually, it’ll be a like-for-like parity, but in the mean time, you have to do some work manually. For this, I elected to stay with NSX-V. Maybe, in the next build, I will go for T. (Or if I find idle time, maybe I will build out a vRO workflow or two to manipulate the NSX-T objects until there is like for like parity.)

As depicted in the above diagram, the Layer 1 Virtualization Layer is where the nested virtual machines reside that will be used for most of my lab testing and experimentation. This is also the vApp created from my vCloud Director’s tenant construct.

My Home Lab’s Logical Architectural Concept explained — or “Nested Virtualization 101”. The base virtual management infrastructure of Layer 0 will manage the physical infrastructure. The virtual management infrastructure in Layer 1 was built similarly to that of the base management infrastructure in Layer 0. Of course, with a different Active Directory Domain, subnets, vCenter, etc. The newly released vRealize Automation 8.0.1 has also been deployed. (This deployment will be covered in a future blog post.) The Layer 2 Virtualization Layer is the logical construct where VMs that are requested will be provisioned.

With the base rebuilt, and the first vCloud Director Tenants’ vApp constructed, I can start to work on getting the rest of the environment set up, configured, and various experiments underway. Stay tuned for more.
