---
layout: post
title: "vRA 8.3 IPAM integration using phpipam and ABX - Part 2"
summary: "Releasing the IP Address from phpipam"
author: samui
date: "2021-03-18"
category: ['lab', 'howto', 'project', 'script', 'vra', automation]
thumbnail: /images/2021/03/855114-ipam.RZ.png
permalink: /blog/vra-83-ipam-integration-using-phpipam-and-abx-part-2
published: true
---

Continuing the work on the phpipam integration with vRA from Part 1, this post will walk you through what I did to “unwind” the process. Or rather, the retirement and recovery of IP Address(es).

## Use Case Review

Let’s review the use case once more. I wanted to automatically allocate and assign the next free IP address in a subnet and then release the IP address(es) during the decommissioning of a workload.

## ABX Action

Part 1 focused on the efforts to get some borrowed code to fulfill the first portion of the use case: “to automatically allocate and assign the next free IP address in a subnet.” If you remember, I borrowed Robert Jensen’s code from the VMware code site. And for the most part, we are not changing the ‘destroy’ portion of the code.

I created a new ABX Action and again, used [Robert Jensen’s code](https://code.vmware.com/en/samples/7270/phpipam-integration-with-vra-8-x#){:target="_blank"} as my base and adding some cleanup it looks like this:

```
import requests
import json

#Global Variables
phpipamhost   = "<phpipam FQDN>"                          #PHPIpam Host
ipadress      = ""                                          #Cleaned up IP
subnetid      = ""                                          #Subnetid Calculated later from IPadress
app           = "vmware"                                    #PHPmyipam API App Name
app_token     = "<enter your phpipam api code>"          #PHPmyipam API App Token
token         = ""                                          #Generated auth token
response      = ""                                          #Response - not used yet.

# =======(Remove this comment block)===========
# I had to modify the IF/Then comparison so that 
# the line would work in this version. Before the change, 
# Robert's code would not resolve to true and therefore
# would not return the subnetid value. Performing a 'find'
# allowed the statement to resolve to a boolean value.
# =======(Remove this comment block)===========

#Get subnet identifier
def get_subnetid(context, inputs, ipadress):
    global subnetid

    #print (ipadress)

    if ipadress.find('192.168.14') >= 0 :
      subnetid = "17"
    
    if ipadress.find('192.168.16') >= 0 :
      subnetid = "18"
      
    if ipadress.find('192.168.18') >= 0 :
      subnetid = "19"

    print ("     - SUBNET ID OBTAINED......"+ subnetid)

# =======(Remove this comment block)===========
# The only change here was made to add the static token. 
# I no longer need to authenticate to get the token. 
# =======(Remove this comment block)===========

#Delete VM info
def delete_vm(context, inputs, ipadress):
    url = "https://"+phpipamhost+"/api/"+app+"/addresses/"+ipadress+"/"+subnetid+""
    headers = {
     'Content-Type': 'application/json',
     'token': ''+app_token+''
    }
    response = requests.request("DELETE", url, headers=headers, verify=False )
   
#Main Function
def handler(context, inputs):

    print (" IPAM FUNCTION RUNNING.........")
    ip_raw =  (inputs["addresses"])
    ipadress = str(ip_raw[0])[2:-2] 
    
    print ("\n PROCCESSING VM................")
    print ("     - "+ipadress)

    print (" GET SUBNET ID.................")
    get_subnetid(context, inputs, ipadress)
        
    delete_vm(context, inputs, ipadress)
    print (" RELEASING IP ADDRESS..........")

```

The dependencies are only “requests”.

![](/images/2021/03/Screen-Shot-2021-03-18-at-4.05.07-PM.png)

## Subscription

A major change that I did to Robert’s example, is I broke up the code to run during specific Events. If you recall in Part 1, we set up an Event Subscription to perform the IP Allocation during the “compute.provision” state. For the IP Release, I created a new Event Subscription to perform the above action during “compute.removal”.

![](/images/2021/03/Screen-Shot-2021-03-18-at-4.12.29-PM.png)

## Action Runs

When the action runs, the Console Log has a clean report of what is occurring.

![](/images/S2021/03/creen-Shot-2021-03-18-at-4.05.50-PM.png)

You may notice that the action will run for each VM that is being destroyed. If there are multiple virtual machines, then you may notice a different action run for each machine. In the pictured example, notice there are two different action runs for IPAM Release.

![](/images/2021/03/Screen-Shot-2021-03-18-at-4.07.49-PM.png)

## Summary

With this action in place, I have met the requirements of my use case. I now have the ability to allocate and assign the next available free IP address(es) during provisioning - as well, as release the IP addresses when a virtual machine is destroyed/retired/decommissioned.

Granted, this was just a small sampling of what can be done with IPAM integration. But I wanted to get basic IPAM functionality working in my homelab, and I can mark this “completed.”

If you need more for your environment, I would encourage you to research and explore suitable DDI alternatives for your needs.

As with everything, your milage will vary by trying to adapt a blog post to your scenario.
