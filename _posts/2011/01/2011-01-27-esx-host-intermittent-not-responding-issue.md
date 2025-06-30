---
layout: post
title: "ESX Host - Intermittent Not Responding Issue"
summary: "Oops... Hosts keeps dropping into 'Not Responding'. How do I deal with this?"
author: samui
thumbnail: /images/2011/01/esx-not-respon.png
permalink: /blog/esx-host-intermittent-not-responding-issue/
date: "2011-01-27"
category: [esx, troubleshooting, vmware]
published: true
---

I've been a witness to a weird phenomenon over the last couple of weeks. I've worked with three different VMware techs on the same issue and it still is not resolved. This is not saying that I'm not getting good help, each of the techs I've spoken with have been very courteous and have a desire to see this through to resolution. Unfortunately, I've stumped them.

I have 6 Dell PowerEdge R710 set up in a cluster, each host with ESX 3.5.0U5 b259926 managed by VCenter 4.0.1 b258672 on a physical machine. This system has been running OK for over several years.

Intermittently over the last several weeks, I've been getting "host not responding" issues, from an ESX Host (not always the same one) within the same Cluster followed by a "disconnected" event from all the VM's on that host. About 8-10 seconds later, I see "Virtual machine .... Connected" for all the VM's, then "host connected state" back to green.

During this "episode", no virtual machines have any issues, no reboots, nothing shows up in their Windows logs, no network interruptions show in the server logs. Pings still work.

It's not always the same host, however the "outages" affect the same cluster each time - other hosts in other clusters are fine. I've rebooted the host that shows the error (after vmotioning the VM's out), but that doesn't seem to help.

I've verified the DNS entries for all the hosts and the VCenter server, everything looks OK. I've done some general internet searching, as well as on the VMWare communities and haven't found much.

I've worked with our security team to try and troubleshoot possible issues on the firewall. We've noticed when the "outage" occurs, it's right after a connection from the VC to the ESX host on 443. The connection switches to port 5989 so that CIM polling can take place, and that's when the "outage" happens.

During this time, ESXTop shows kipmi0 as the active service during the outage, and when the host resumes CIMSERVER is the active service. I've tried disabling CIM within the ESX host. I've also stopped the Pegasus service as well. Neither of these made a difference.

From a wild tangent that was suggested, I even tried replacing the SSL cert on the ESX host - this too had no effect to the intermittent occurrences.

I'll try tech support again in the morning.

There's a plan to upgrade to ESX 4.0.1U2 in the next few weeks. I'm not sure that I may be able to live with this issue. So far, I've had a fileserver VM restarted via HA because the host was not responding for too great of a period of time. This caused some issues because the fileserver syncs to a DR facility.

## UPDATE:

The ESX hosts were all updated to ESX 4.0.1 U2 (build 261974), however this made no difference to the connectivity issue.

I thought that maybe the issue revolved around the fact that we added an additional SAN to the cluster. We migrated all VMs from an older EMC CX700 to a new NetApp FAS3140. So I pulled one of the hosts out of the cluster, and had the Storage Administrators remove the Host from the CX700 storage group. This removed the CX700 LUNs from this one host without deleting them from the rest of the hosts in the cluster. The host still had the NetApp LUNs attached.

At first, the connectivity issues stopped. I double-checked the LUNs and the host, and found that the host was missing 6 NetApp LUNs. I did a rescan and a refresh of the HBAs and the LUNs were reattached to the Host. After a couple of hours of monitoring, the connection error returned. Instead of receiving the error every 30 seconds or so, I now am getting the error every 2 to 4 hours. I then had the ESX host pulled out of the igroup for the NetApp.

At this point, there should have been no LUNs presented to the ESX host. I did a rescan of the HBAs and found a single CX700 LUN (ID 0, 0MB in size) remaining attached to the ESX host. I had storage look to see if something was still being presented from the CX700 side. My Storage Admin couldn't find anything. He then blocked the ports on the brocade between the ESX host and the CX700. Now, all LUNs were removed from the ESX host.

The host still consistently looses connectivity to the VC every three hours and presents an error within the VC. Monitoring is currently underway. If the error appears by lunch, then i'll get vmware on the horn and troubleshoot this afternoon.

## UPDATE:

We recently upgraded the virtual center from 4.0 to 4.1. During the upgrade, it upgraded the vpxuser and vcagents on each of the ESX hosts. Once this completed on the ESX hosts, the connectivity issues stopped. We're thinking that somehow the vcagent for the ESX hosts were corrupted and that was why we were getting intermittent connectivity issues.
