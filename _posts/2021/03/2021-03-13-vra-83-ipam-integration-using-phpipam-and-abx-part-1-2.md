---
layout: post
title: "vRA 8.3 IPAM integration using phpipam and ABX - Part 1"
summary: "Reserving an IP Address from phpipam"
author: samui
date: "2021-03-13"
category: ['lab', 'howto', 'project', 'script', vra, automation]
thumbnail: /images/2021/03/phpipam-vra-logo-8.png
permalink: /blog/vra-83-ipam-integration-using-phpipam-and-abx-part-1
published: true
---

This article will cover my process of providing an IPAM integration to my vRA 8.3 homelab deployment.

For most of my customers, they have commercial grade IPAM solutions, like Infoblox, Bluecat, etc, that provides DDI services. (I will say that while I have seen the acronym DDI repeatedly, I took the time to look it up to see what it means. For those curious it is DHCP, DNS, IPAM.) Through a partnership with my work, I could have gotten an extended license to setup Infoblox for my lab experimentation. However, I wanted to try my hand at something interesting. I wanted to challenge myself to see if I could create an integration between vRA and an open source solution called “[phpipam](https://phpipam.net/){:target="_blank"}”.

I got the inspiration from a conversation I had with my manager and an idea from a previous customer that utilized phpipam. From that conversation, I wanted to use Action Based eXtensibility (ABX) and python to create the integration.

## Use Case

The use case I wanted to address was automatically allocating and assigning the next free IP address in a subnet and releasing that IP address during the decommissioning of a workload.

Knowing that almost every vRA customer uses some form of IPAM integration in their automation engines, I figured that someone had to have done this before. (Yes, I didn’t want to reinvent the wheel.) Therefore, I consulted with Professor Google and that didn’t go very well. It seems like hardly anyone uses phpipam. But I did find one entry buried deep into my search results (it was on page 3). Robert Jensen, a VMware Lead Cloud Management Specialist (Yay! A Co-Worker!) has added to the [VMware code site](https://code.vmware.com/en/samples/7270/phpipam-integration-with-vra-8-x#){:target="_blank"} his experiment work regarding a similar use case. His use case is more aligned with updating phpipam with the IP address after the VM has been deployed. In his words, it acts more like a CMDB. But this gave me a good starting point.

I grabbed Robert’s code and gave it a code looky-look. I liked the format of it, but I found that I wanted something more IPAM like. However, I am going to use his code as my base. I wanted the ABX action to look at phpipam first, grab the next available IP address, and pass it back to vRA to use during provisioning. That was the use case I was after. In addition, I knew that I needed to update Robert’s code.

## Let’s start with allocation and assignment first

Robert’s ABX action ran in the compute.provision.pre event of provisioning. But during this event, any IP address that is to be used has already been identified for a vm’s consumption within vRA, whether through DHCP, IP Pools, etc. I needed my code to run before this event.

If you’re new to the concept of subscription and events it’s probably good to understand the execution flow or order when an instance is provisioned. Here is a diagram to help visualize this:

![](/images/2021/03/cas-extensibility-event-flow-7.png)

In vRA, outputs are used relating to the items under the payload. So to set the IP address, I would have to have an output called “addresses” of the appropriate data type. An additional consideration is not all the fields can be altered, nor are the fields consistent across Lifecycle Event Topics. For example, the compute.provision event has the field for IP addresses, but it’s read-only. The network.configure event also has a field for IP Addresses that can be altered. However it’s missing the resourceNames (aka hostnames) field, which could be useful.

This was a hurdle that I would indeed have two overcome. After several dumps of the payload during the network.configure event, I found that the payload does not provide an “addresses” field - even though it is listed as a parameter of the event.

![](/images/2021/03/Screen-Shot-2021-03-10-at-4.40.54-PM-7.png)

Compared to the compute.provision event, which does include a Read-Only “addresses” field in the payload.

![](/images/2021/03/Screen-Shot-2021-03-10-at-4.40.41-PM-7.png)

In that same google search, I also found an article from vnugget himself, Andy, a Consulting Architect at VMware (another co-worker!). His article, [Customizing IP Assignments with vRO](https://vnuggets.com/2020/01/24/vrealize-automation-customizing-ip-assignments-with-vro/){:target="_blank"} revolved around utilizing, of course, vRO. Andy’s article confirmed that I would need to run the action during the network.configure event, however, I would have to construct the “addresses” field.

That sounds easy enough, but actually wasn’t. During my experimentation, I could easily assign an IP address as a string to the field, but that was creating the addresses field in the wrong format. The addresses field is a 2-dimensional string array. All input, must reflect this. The unfortunate reality here is that, while we point out that the field is a 2D array, VMware doesn’t provide documentation to indicate the relationship of the elements in the array and what or how they relate to other elements.

Andy provides some detail to this:  
_The order that the addresses array should be populated in would be the same order as the machine names in the externalIds array are provided. If the input array_(externalIds - aka hostnames)_contains VMs in the order “VM1, VM2, VM3” then the order you populate the addresses array would be “VM1, VM2, VM3”._

But I will admit, I was still lost. I reached out to a developer friend of mine for more insight. He stated that the first level of the array is the virtual machine, and the second level is the IP address of the NIC.

While this provided some clarity, it gave no insight into how to build the array. I tried numerous different methods to try and build the array properly. My build experimentations were using just one VM and a single IP address.  

```
outputs[“addresses”]=[externalId,ipaddress]
outputs[“addresses”]=[ipaddress,""]
```

These all failed with various array format errors during testing. But try, try again and eventually you get the format. Part of my problem was that I thought I needed to add the hostname of the VM into the array. However, this wasn’t needed. Because of that, I was thrown on the formatting of the array. I needed to build the array like so:  

```
addesses = [ [ipForVm1], [ipForVm2] ]
```

I applied this to the code and retested.

This:

```
outputs["addresses"] = [[ipaddress]]
```

Produced this:

```
{'addresses': [['192.168.18.11']]}
```

Which was then passed back to vRA and seen properly in the compute.provision event.

![](/images/2021/03/Screen-Shot-2021-03-11-at-11.57.58-AM-7.png)

This IP was then assigned to the VM in vRA.

![](/images/2021/03/Screen-Shot-2021-03-11-at-11.58.43-AM-7.png)

vCenter view of the VM.

![](/images/2021/03/Screen-Shot-2021-03-11-at-11.59.22-AM-7.png)

This worked for a single VM, but I needed to figure out how to use this with multiple VMs. Further experimentation was required. To add to the 2D array, I needed to perform an “append” to the array. By default, the Python PIP code library built into vRA’s engine does not contain the “append” option. I added “numpy” to the python script to be able to utilize “append”. Unfortunately, using just “append” did not apply the next IP for the next VM. I needed to add a loop. Additionally, I also needed to alter the input for the array field.

```
addresses.append([ipadress,])
outputs["addresses"] = addresses
```

This created the addresses array like so:

```
{'addresses': [['192.168.18.11'], ['192.168.18.12']]}
```

It seems like this worked for multiple machines in a single blueprint — and I am able to repeat the process.

## Let’s review

Here’s what I have so far.  

### Blueprint

![](/images/2021/03/Screen-Shot-2021-03-12-at-2.15.08-PM-7.png)

The blueprint is a simple Ubuntu 16 blueprint that utilizes cloud-init. There’s nothing special about the blueprint.

### ABX Action/Python Script

The ABX Action is a python script with the dependencies of “requests” and “numpy”. Again, numpy is used to obtain the “append” instruction.

```
import requests
import json
import numpy as np

#Global Variables
phpipamhost   = "<phpipam FQDN>"                          #PHPIpam Host
ipadress      = ""                                          #Cleaned up IP
subnetid      = ""                                          #Subnetid Calculated later from IPadress
hostname      = ""                                          #Cleaned up Hostname
description   = ""                                          #Application Description - Used for Project name
owner         = ""                                          #The requester of the resource
note          = "Created by VRA"                            #Note text
app           = "vmware"                                    #PHPmyipam API App Name
app_token     = "<enter your phpipam api code>"          #PHPmyipam API App Token
token         = ""                                          #Generated auth token
response      = ""                                          #Response - not used yet.

#Get subnet identifier
def get_subnetid(context, inputs):
    global subnetid
    
	# =======(Remove this comment block)===========
	# Robert's code attempted to distinguish which subnet 
	# to use based on the first three octets of the IP address.
	# Since we don't have an IP yet, I needed a way to
	# determine the php subnetid. Since the "network.configure"
	# event supplies the 'networkSelectionId' which corresponds 
	# to the subnets in my lab, I made simple 'IF' statements
	# to set the php subnetid. At some point, I may automate
	# automate this in the future.
	# =======(Remove this comment block)===========

    #Network Key
    # 3f3d5f2a-6f4b-4c6f-a3ae-fcbfdd0d485b -- 192.168.14.X
    # 8eb1a3e9-2840-48b4-8be0-e0071f26357a -- 192.168.16.X
    # ac8db1e3-eb91-4ad2-90c0-28535bb929dd -- 192.168.18.X
    
    network_name = str(inputs["networkSelectionIds"]) 
    
    if '3f3d5f2a-6f4b-4c6f-a3ae-fcbfdd0d485b' in network_name :
      subnetid = "17"
    
    if '8eb1a3e9-2840-48b4-8be0-e0071f26357a' in network_name :
      subnetid = "18"
      
    if 'ac8db1e3-eb91-4ad2-90c0-28535bb929dd' in network_name :
      subnetid = "19"

    print ("     - SUBNET ID OBTAINED......"+ subnetid)

#Create VM info
def create_vm(context, inputs):
    #Fetch IP
    global ipadress
    global hostname
    global owner
    global description
    
	# =======(Remove this comment block)===========
	# I extracted the requester to assign them as an owner
	# within php. 
	# =======(Remove this comment block)===========

    owner = str(inputs["__metadata"]["userName"])
    url = "https://"+phpipamhost+"/api/"+app+"/addresses/first_free/"+subnetid
    payload = "{\"hostname\":\""+hostname+"\",\"description\":\""+description+"\",\"owner\":\""+owner+"\",\"note\":\""+note+"\"}"
    headers = {
     'Content-Type': 'application/json',
     'token': ''+app_token+''
    }
    response = requests.request("POST", url, headers=headers, verify=False, data=payload) 
    jsonResponse = response.json()
    ipadress = response.json().get('data')
    return ipadress
    
   
#Main Function
def handler(context, inputs):
    global hostname
    global description
    print (" IPAM FUNCTION RUNNING.........")
    addresses = []
    outputs = {}

	# =======(Remove this comment block)===========
	# Notice above that the addresses and outputs array 
	# are declared. Also, notice below that I have another
	# key that has been setup to identify the name of the 
	# Project using another 'IF' statement to match the 
	# projectId to a human readable name. This is optional,
	# but is being passed into the description for phpipam.
	# =======(Remove this comment block)===========
     
    #Convert ProjectId to readable Name
    # Project Key
    # '834d5958-715e-4c12-8c01-1fa13b3e8c98' == Research
    # TBD == Production
    projid = str(inputs["projectId"])
    if '834d5958-715e-4c12-8c01-1fa13b3e8c98' in projid :
      description = "Research"

	# =======(Remove this comment block)===========
	# Below, this 'FOR' loop will cycle through each vm
	# and perform the following:
	#      - fetch the subnetId 
	#      - allocate and assign the IP Address in phpipam
	#	   - append the IP address to the array
	# =======(Remove this comment block)===========

    names =  (inputs["externalIds"])
    for hostname in names:
        print ("\n PROCCESSING VM................")
        print ("     - "+hostname)

        print (" GET SUBNET ID.................")
        get_subnetid(context, inputs)
 
        ipadress = create_vm(context, inputs)
        print (" ALLOCATED IP ADDRESS..........")
        print ("     - "+ipadress)
        
        addresses.append([ipadress,])
        outputs["addresses"] = addresses

    print ("\n\n RETURNING IP ASSIGNMENTS......")
    print ("     - ", end = " ") 
    print (outputs)
    
    return outputs
```

During the execution of the ABX action, the console log for the action will display the following:

```
 IPAM FUNCTION RUNNING.........

 PROCCESSING VM................
     - Research-webserver-VM-654
 GET SUBNET ID.................
     - SUBNET ID OBTAINED......19
 ALLOCATED IP ADDRESS..........
     - 192.168.18.11

 PROCCESSING VM................
     - Research-webserver-VM-655
 GET SUBNET ID.................
     - SUBNET ID OBTAINED......19
 ALLOCATED IP ADDRESS..........
     - 192.168.18.12

 RETURNING IP ASSIGNMENTS......
     - {'addresses': [['192.168.18.11'], ['192.168.18.12']]}
```

### Event Subscription

I created an event subscription for a Network Configure event assigning the above ABX action as a blocking task.

![](/images/2021/03/Screen-Shot-2021-03-12-at-4.07.28-PM-7.png)

### phpipam Results

Each virtual machine will be registered like with phpipam with its hostname, description (mapped to the Project Name), and the owner (username of the vRA requestor). As you can see from the example of this blog, two virtual machines were deployed and registered with its own IP address.

![](/images/2021/03/Screen-Shot-2021-03-12-at-3.53.35-PM-1.png)

Details of a vm registration will show further details (if any were added). In my case, I added a note indicating the virtual machine was created by vRA. Other items that could be added could include the MAC address, Location, etc. For my purposes, I kept it simple.

![](/images/2021/03/Screen-Shot-2021-03-12-at-4.00.18-PM-7.png)

## Wrap up

To summarize. I was able to take some code that was provided by Robert Jensen and update it to function as my IPAM allocation with phpipam.

I always preach to my customers that they need to incorporate the full lifecycle for any integration. I say if you can “wind” something up, you must be able to “unwind” it as well. For this integration, I am not done as I must be able to show how I release the IP address during a virtual machine retirement — the “unwind” if you will.

For now, here’s the “wind up”.

#### Reference Links

phpipam API Reference  
[https://phpipam.net/api-documentation/](https://phpipam.net/api-documentation/){:target="_blank"}

PHPipam integration with vRA 8.x

[https://code.vmware.com/en/samples/7270/phpipam-integration-with-vra-8-x](https://code.vmware.com/en/samples/7270/phpipam-integration-with-vra-8-x#){:target="_blank"}

vNuggets - Customizing IP Assignments with vRO  
[https://vnuggets.com/2020/01/24/vrealize-automation-customizing-ip-assignments-with-vro/](https://vnuggets.com/2020/01/24/vrealize-automation-customizing-ip-assignments-with-vro/){:target="_blank"}

Jesse Boyce - vRA 8 and phpIPAM

[https://blog.jpboyce.org/2020/12/07/vra-8-and-phpipam/](https://blog.jpboyce.org/2020/12/07/vra-8-and-phpipam/){:target="_blank"}

Two - Dimensional Array in Python

[https://www.askpython.com/python/two-dimensional-array-in-python](https://www.askpython.com/python/two-dimensional-array-in-python){:target="_blank"}

phpIPAM PowerShell get next IP API

[https://esxweb.wordpress.com/phpipam-powershell-get-next-ip/](https://esxweb.wordpress.com/phpipam-powershell-get-next-ip/){:target="_blank"}

My first vRA 8 extensibility workflow ABX Edition

[https://vbombarded.wordpress.com/2019/11/19/my-first-vrealize-automation-8-extensibility-workflow-action-based-extensibility-abx-edition/](https://vbombarded.wordpress.com/2019/11/19/my-first-vrealize-automation-8-extensibility-workflow-action-based-extensibility-abx-edition/){:target="_blank"}
