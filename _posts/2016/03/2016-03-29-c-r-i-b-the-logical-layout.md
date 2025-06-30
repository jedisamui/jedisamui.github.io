---
layout: post
title: "C.R.I.B. The Logical Layout of My HomeLab"
summary: "A review of the lab's logical layout." 
author: samui
date: "2016-03-29"
category: [lab, project, vmware, vra, automation]
thumbnail: /images/2016/03/HomeLab.png
permalink: /blog/crib-the-logical-layout/
published: true
---

C.R.I.B. - Stands for **C**omputer **R**oom **I**n a **B**ox. This is the name I have given my homelab. I've used my C.R.I.B. to educate myself, experiment with things, and demo products to my customers.

As you've read in previous posts, my homelab has evolved over the years. Currently, I run one physical Supermicro Super Server attached to a Synology DS 1512+ Array, connected to a gigabit switch. Many of my friends and co-workers have asked how I run everything on one physical box -- which I am going to call pESX. I've tried to explain it and draw it out on a whiteboard. However, until you've seen it drawn out, the explanation gets confusing -- unless you're familiar with nested virtualization.

I utilize nested virtualization to expand the capabilities of the C.R.I.B. without additional physical hardware. If you are unfamiliar with nested virtualization, it is the ability to run ESX as a virtual machine -- which I will call vESX. (William Lam has written numerous articles on how to do it. Just google "William Lam" and "nested virtualization" if you want more info.) The entire CRIB is accessible from my home network -- which is a lifesaver, as I do not have to work in my office. I can access it via VPN or the couch.


A pfSense virtual machine (GATEWAY) was implemented to act as firewall and router to the entire virtualized environment, including the nested layer. The pESX is on the base network with a vmkernel attached to the pfSense Virtual Machine to allow for manipulation and modification of the firewall rules and network configuration. All traffic in and out of the virtual environment will pass through the pfSense VM. The firewall in the pfSense provides isolation as well as communication between the various networks.

All of the infrastructure virtual machines sit on the first virtualization layer - this is considered as the "Management Cluster". However, this cluster, is only made up of the one physical ESX Host (pESX). Normally, we would want multiple hosts for HA redundancy. (But this is a lab, and I'm on a budget.) The vESX Virtual machines sit in this layer as well. The vESX VMs have a direct connection to the base network for access to the iSCSI storage Array. vESX make up the ESX Compute Resources of the "Payload Cluster". These clusters; the Management Cluster and the Payload Cluster, are a VMware architectural design best practice. The infrastructure is made up of your basic VMs; a Domain Controller (DC), a SQL Server (SQL), the Management vCenter (vCSA6), a Log Insight Server (LOG), and a VMware Data Protection VM (VDP) for backups. In addition to these VMs, the vRealize Automation (vRA) VMs and Payload vCenter (PAYVC01) also sit in the management cluster. This self-service portal deploys to a vCenter (PAYVC01) endpoint controlling the compute resources of the Payload Cluster.

The Payload Cluster is made of three virtual ESX Hosts (vESX) and provide the various resources; network, CPU, RAM, & Storage, for consumption by vRA or other platform products. There is an Ultimate Deployment Appliance (UDA) VM providing the ability to deploy scripted ESX images. This provides the ability to quickly rebuild the hosts, if needed.

This is just the base. I am in process of deploying NSX into the environment to provide the ability to deploy multi-machine blueprints within vRA. In addition, I intend on exploring SRM integration with vRA.
