---
layout: post
title: "Enable EVC when vCenter is running as a VM"
summary: "How to enable EVC when vCenter is on the cluster"
author: samui
thumbnail: /images/2011/01/evc.png
permalink: /blog/enable-evc-when-vcenter-is-running-as-a-vm/
date: "2011-01-26"
category: [esx, vmware]
published: true
---

I wanted to enable EVC in my lab environment, but found that I couldn't because my VC was a VM within that same cluster. VC refused to enable it because there was still at least one VM powered on within the cluster. Duh.

Of course, I broke out my handy-dandy do-it-all tool (yep, google) and proceeded to locate a workaround. Lo and behold, I found a useful link. Thanks go out to Bert at [http://virtwo.blogspot.com](http://virtwo.blogspot.com).

He had instructions for how to do this within an ESX 3.5U2 environment. My environment is a complete ESX 4.1 environment consisting of two ESX 4.1 Hosts. Bert's instructions work fairly well in my environment.

![41cluster](/images/2011/01/41cluster.jpg)
41cluster
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}

Here's what I did for my environment:

1) Evacuate one ESX host in your cluster and place it in maintenance mode. 
2) Move it out of the cluster, right under your datacenter object. 
3) Exit the host out of maintenance mode. 
4) Manually migrate (VMotion via drag-and-drop for example) your VirtualCenter VM (running on an ESX host in the cluster) to the ESX host that is now outside the cluster. Do this with all other VMs that were still running in the cluster. 
5) Enable EVC on your cluster. 
6) Migrate the VMs back to the ESX host within the cluster (again, drag and drop). 
7) Place the ESX host back into the cluster.

Now, I have a happy cluster running EVC.

Here's the link for [Bert's blog entry](http://virtwo.blogspot.com/2008/08/how-do-i-enable-ecv-when-vc-is-running.html). 
There's also a KB Article (#1013111) on this subject as well. [You can find it here.](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1013111)
