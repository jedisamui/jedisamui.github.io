---
layout: post
title: "Project: HomeLab"
summary: "The jounrey continues"
author: samui
thumbnail: /images/2010/09/Home_Network.jpg 
permalink: /blog/project-homelab-2/
date: "2010-09-27"
category: [lab, network, project]
published: true
---

Part 2: Network Planogram

Up until now, my home network has been pretty simple and fairly common-place. It consists of the following: Internet > Cable Modem > Linksys Wireless Router > LAN/WLAN. The service has been consistent over the years, but for this HomeLab Project - it too needs an overhaul. The new network needs to be separated between the home LAN, and the LAB. The two subnets will also need to be trusted so that machines can communicate between the two networks.

My Linksys Router (a WRT54GS) has been running dd-WRT on it for a few years now. Unfortunately, the firmware version (v6) of the Linksys will only allow me to run the "micro" version of dd-WRT. This prevents me from being able to set up VLANs and split the home LAN from the Lab LAN.

I plan to implement a Cisco 2600 Router that has been lying around collecting dust into this new design. This should allow me to split the two networks using VLANs.

Since I have very little background in configuring and setting up routers, this implementation will be a good challenge for me. However, I will need help (Google can only do so much). I have contacted a few of my Cisco friends to bounce ideas and designs off them. With their help, I have a rough outline and a generic/basic router config to get me started.
