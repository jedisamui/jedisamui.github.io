---
layout: post
title: "Unable to access file since it is locked"
summary: "Oops.... I can't access the drive!"
author: samui
thumbnail: /images/2011/04/lock-file.png
permalink: /blog/unable-to-access-file-since-it-is-locked/
date: "2011-04-29"
category: [esx, troubleshooting, vcenter, vmware, vmware-kb]
published: true
---

I came across this error earlier today and spent the entire day troubleshooting it.

![unable_access_error](/images/2011/04/unable_access_error.jpg)

So I wanted to share with you what my resolution was. I was getting this error on a MS cluster setup within my environment. I was working with a Systems Engineer; we were converting an existing MS cluster from using vmdks for the quorum and msdtc shared drives to RDMs. Once Node A was configured, I attached the drives and powered up Node B. Node B would make it to 95% and then provide the error (shown above).

I checked and double-checked my settings. Second SCSI controller set to LSI Logic Parallel Controller Type with Physical SCSI Bus Sharing selected (VMs were on separate hosts). SCSI IDs set to SCSI 1:1, 1:2, and 1:3 for the new RDM Drives. The RDMs set to virtual compatibility mode.

After numerous attempts of re-configuring as I was doubting that I did it right. I consulted Google - which lead me to an older [white paper](http://www.vmware.com/pdf/vi3_35/esx_3/vi3_35_25_u1_mscs.pdf){:target="_blank"} (one for ESX 3.5). This wasn't too bad, as the steps are pretty much the same for our ESX 4.0 Update 2 environment. Since the white paper verified that I was doing the right thing, I dug deeper into Google for an answer. I came across this [vmware KB Article (#10051)](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=10051){:target="_blank"} that provides an in-depth detail of the error. So now I'm worried. I start troubleshooting according to the KB article, and it did allow me to find something rather odd. Even though the VM was physically sitting in one Datastore, the ESX host believed it to be in another datastore. A quick svmotion corrected this, but it was REALLY odd.

Almost on the verge of giving up, I consult with a co-worker. He starts going down the list of tasks to do that is eerily familiar - they were the same steps from the white paper. Once they were triple-checked, he asks me the one question that I've asked the SE three or four times previously. Has the cluster been completely unconfigured. Turns out the cluster was completely removed on Node A, but Node B had not been evicted. Once the cluster cleanup was done on Node B, the two nodes booted up fine with the RDMs attached without an error. Eight hours later, the cluster is reconfigured and running fine.

The moral of the story. Rule #1 is always right. Do not trust what the user tells you. :)
