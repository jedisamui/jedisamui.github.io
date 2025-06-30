---
layout: post
title: "Fixing Renumbered vmnics"
summary: "Oops...my vmnics for my host got renumbered. How do I fix?"
author: samui
thumbnail: /images/2010/05/esxi-network.png
permalink: /blog/fixing-renumbered-vmnic/
date: "2010-05-20"
category: [esx, vmnic, commandline, vmware]
published: true
---

I've been in a situation where I've installed a new NIC and it renumbered the vmnics within an ESX host. The reason this became an issue was that the Service Console vmnic was one of the vmnics that was affected. Thus I lost my access to the ESX host. Thankfully, I could just drive over to the Datacenter and fix it via the commandline. It was just inconvenient to do so. A quick google for instructions and I was off.

So apparently there are multiple ways to fix this. For me, Method 1 got me up and running within a few minutes with little to no fuss.

Method 1 – Editing the esx.conf file • Login to the Service Console • Check your existing NIC numbering by typing ‘**esxcfg-nics –l**’ • Type ‘**cd /etc/vmware**’ to change to the correct directory • Type ‘**cp esx.conf esx.con.bak**’ to make a backup of this file as it is a critical configuration file for ESX • Type ‘**nano esx.conf**’ to open the file for editing • Type CTRL-W and then enter ‘**vmnic2**’ to search for the new first NIC • Change ‘**vmnic2**’ to ‘**vmnic0**’ • Change the subsequent NIC’s from ‘**vmnic3**’ to ‘**vmnic1**’, ‘**vmnic4**’ to ‘**vmnic2**’ and ‘**vmnic5**’ to ‘**vmnic3**’ • Type CTRL-O to save the file • Type CTRL-X to exit the Nano editor • Type **esxcfg-boot -b** to rebuild the config files • Shutdown and restart the ESX server, when the server comes back up the NIC’s will be numbered vmnic0 – vmnic3, verify this by typing ‘**esxcfg-nics –l**’

Method 2 – Modify your vswitch configuration • Login to the Service Console • Check your existing NIC numbering by typing ‘**esxcfg-nics –l**’ • Check your current vswitch configuration by typing ‘**esxcfg-vswitch –l**’ , note which NIC’s are assigned to which vswitches (uplink column) • Remove the old NIC’s that have been renamed by typing ‘**esxcfg-vswitch –U vmnic# vswitch\_name**’, ie. **esxcfg-vswitch –U vmnic0 vSwitch1** • Add the new NIC’s with the correct names by typing ‘**esxcfg-vswitch –L vmnic# vswitch\_name**’, ie. **esxcfg-vswitch –L vmnic2 vSwitch1** • Repeat this process for any additional NIC’s. Once you have the vswitch that contains the Service Console corrected you can also log in via the VI Client and correct the other vswitches that way • Your newly renamed NIC’s should now be assigned to the original vswitches and your networking should now work again

This information was found within the VMware Knowledge Base. (Awesome!) 
[How To Configure Networking from the Service Console Command Line](http://kb.vmware.com/kb/1000258){:target="_blank"} 
[VI Client loses connectivity to the ESX Server Host after you add a new network adapter](http://kb.vmware.com/kb/2243){:target="_blank"}
