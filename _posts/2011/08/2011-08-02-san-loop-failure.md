---
layout: post
title: "SAN Loop Failure"
summary: "Oops....fiber loop fiasco"
author: samui
thumbnail: /images/2011/08/100_0595-Small.jpg 
permalink: /blog/san-loop-failure/
date: "2011-08-02"
category: [esx, vmware, netapp, storage, troubleshooting, vmware-kb]
published: true
---

Here's an example of how important it is for your VMware Engineers to either have visibility into the SAN infrastructure, or work very intimately with the SAN Admins. We lost a fiber loop on one of our NetApp FAS 3160 SANs yesterday. If my team did not receive the critical fail-over emails, we would not have known that there were storage issues for a couple of vital hours.

When your customers are complaining that there is something wrong with their VMs, or they have lost access to them; it is imperative to investigate and start troubleshooting. If we would have started troubleshooting without the knowledge of the SAN issue, then we would have started working on the VMs, which would have quickly led back to the ESX hosts they resided on. Troubleshooting the ESX hosts could have potentially made our outage A LOT worse.

This particular environment consists of a virtual center 4.1 and eight ESX 4.01 U2 hosts. There happens to be a bug in ESX 4.XX that occurs when a rescan is issued while an all-paths-down state exists for any LUN in the vCenter Server cluster. Therefore, a virtual machine on one LUN stops responding (temporarily or permanently) because a different LUN in the vCenter Server cluster is in an all-paths-down state. ([Yeah, I cut and paste that from the VM KB, that you can read here](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1016626){:target="_blank"}.) _The VM KB also mentions that the bug was fixed in ESX 4.1 U1._

Since we were receiving the outage emails, we knew that something was up with our storage. This allowed us to work closely with our storage admins to understand the full extent of the outage.

The details of our recovery goes something like this: A fail-over from Node A to Node B was made during the outage, however, Node B did not have access to the failed loop, therefore the aggregate on that loop was down. Node B carried the load for all other working aggregates giving our storage guys and the NetApp technician time to work on the loop. When repairs were completed, a fail-back (give-back) was done to allow Node A to take back over its share of the load. We confirmed with the NetApp tech that all paths and LUNs were represented to the ESX hosts. We then went in and rescanned each ESX host in the cluster for the ESX host to recognize the downed LUNs once more. After the scan, we viewed the properties of the LUN to ensure all paths were up. Once that was verified, we QA'd our VMs. Of 45 affected VMs, we had one casualty. The VMX file of one VM got corrupted.

The situation could have been worse, much worse. But I'm very glad that we stayed smart and calm and worked closely with our storage admins.
