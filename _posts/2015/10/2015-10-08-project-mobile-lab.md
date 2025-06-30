---
layout: post
title: "Project: Home Lab v4.0 - The Home Lab goes MOBILE!!"
summary: "New lab hardware!"
author: samui
thumbnail: /images/2015/10/IMG_61542.jpg
permalink: /blog/project-mobile-lab/
date: "2015-10-08"
category: [lab, project]
published: true
---

_Woot!_ Ok, that was cheesy, I know. However, my home lab has undergone significant improvement, redirection, and an overall evolution. And that calls for celebration! So Woot it up!

Over the last two years, I went from individual rack mounted Dell PowerEdge 1950s and Synology storage, to a four blade [Dell c6100 Cloud Server chassis](http://www.ebay.com/sch/i.html?_from=R40&_trksid=p2050601.m570.l1311.R4.TR6.TRC2.A0.H0.Xdell+c6100+.TRS0&_nkw=dell+c6100+xs23-ty3&_sacat=0){:target="_blank"}, to rented COLO Space from [OVH](http://ovh.com){:target="_blank"}, and now to the all new mobile lab. You are probably asking yourself "why all the changes?" Well, it grew from a need to be more energy efficient, to a random act of god (lightning strike), and eventually to money.

While the Dell Cloud Server averaged approx 700Watts fully loaded keeping my electric bill down, my new Mobile Lab will average less than 120Watts. (I'm not exactly sure how much lower as I haven't put the meter on it yet.) OVH was a very good colo - especially for the price. I just don't want to pay them rent any longer. The Synology will remain the in-home media server/backup location for the family and no longer used by the lab.

Why mobile? It doesn't have to be. I have a VPN to my home, so I can connect to it remotely. Plus, the new server has the ability for remote control, so I can power it on/off - if needed, without the need of the "break/fix" wife. But with the size of the server, I can take this bad boy on the road. The downside to that is TSA, theft, and potential travel damage. The upside to it is being able to plug it up and demo on the spot. In addition, when there are power or internet outages preventing me from connecting to my home from the hotel or the customer's workplace; I could still work in the lab -- without that dependency.

What makes up this mobile lab? Over the last year, several co-workers and I have been discussing building small labs. We had several ideas for what we wanted to use, but we were also very particular in what we wanted. We pinned requirements and started searching. One friend decided to run an Intel NUC, another is utilizing a Gigabit server, and one more suggested the hardware I am using now.

Our requirements (or I should say, my requirements) were the following:- It had to have a small footprint.
- It had to be energy efficient, low power, and put out little to no heat and noise.
- It had to be able to run 64GB of RAM or more.
- It had to be inexpensive.

  
With that list, you would think oh - and Intel NUC or a MAC MINI. But neither of those can use more than 16GB of RAM. Once you install vCenter, a nested ESX host, and a couple of VMs, you are out of RAM. In addition, the NUC is the only one that is inexpensive. However, you will have to purchase a few of them to be able to do anything worthwhile within a lab environment.

This brings us to the setup of the mobile lab. All of my storage, I salvaged from previous equipment that I have had lying around from my earlier versions of my lab.

![Measurements: 7.5"x7.5"x3"](/images/2015/10/IMG_6161.jpg)
**HOME LAB v4.0 (Mobile Lab)** 
Motherboard:       
[SuperMicro Super Server Motherboard X10SDV-TLN4F](http://www.supermicro.com/products/motherboard/xeon/d/x10sdv-tln4f.cfm){:target="_blank"}      
(1) Intel Xeon D-1540 8-Core 2Ghz CPU      
**UPDATE:** 128GB RAM (Max)      
(2) 10Gb NICs      
(2) 1Gb NICs      
(1) IPMI Port Case:      
[Travla C299 Mni-ITX Case](http://www.itxdepot.com/shop/index.php?id_product=46&controller=product#/external_ac_adapter_and_internal_atx-120w_ac_adapter){:target="_blank"} 

Internal Storage:      
[(1) 250GB Samsung 840 EVO SSD](http://www.newegg.com/Product/Product.aspx?Item=N82E16820147248){:target="_blank"}      
[(1) 1TB Western Digital 5400 SATA III HD Blue](http://www.amazon.com/gp/product/B00C9TEBJQ/ref=pd_lpo_sbs_dp_ss_1?pf_rd_p=1944687562&pf_rd_s=lpo-top-stripe-1&pf_rd_t=201&pf_rd_i=B005WKGTBW&pf_rd_m=ATVPDKIKX0DER&pf_rd_r=103V9QADYTZQKD92B1VS){:target="_blank"} 

External Storage:      
[(1) 2TB Seagate Backup Plus Slim External USB 3 Drive](http://www.amazon.com/Seagate-Portable-External-Storage-STDR2000100/dp/B00FRHTSK4/ref=sr_1_1?ie=UTF8&qid=1444245866&sr=8-1&keywords=seagate+backup+plus+2TB){:target="_blank"}     
[(1) SanDisk Cruzer Fit CZ33 32GB USB 2.0 Low-Profile Flash Drive](http://www.amazon.com/SanDisk-Cruzer-Low-Profile-Drive--SDCZ33-032G-B35/dp/B00812F7O8/ref=sr_1_12?ie=UTF8&qid=1444245775&sr=8-12&keywords=32gb+sandisk){:target="_blank"} 

Software:      
Hypervisor: vSphere ESXi 6.0 U1b build 3380124      

Backups: VMware Data Protection 6

The unit is tiny compared to many other options with the same feature sets. I can pack it into my backpack along with the power supply, a couple of ethernet cables, a small five port switch and still have room for a bag of Reese's Pieces. :) The mobile lab is based on the SuperMicro Super Server Mini-ITX series of motherboards. It is the foundation of the entire lab. Within this base, is a complete nested vSphere 6 environment running almost a full VMware SDDC stack (I'm not running NSX).

As time allows, I will detail the build-out of this lab in the future.
