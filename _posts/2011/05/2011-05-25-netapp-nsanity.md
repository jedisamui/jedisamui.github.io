---
layout: post
title: "NetApp nSANity"
summary: "NetApp tool to collect SAN information"
author: samui
thumbnail: /images/2011/05/nsanity1.jpg 
permalink: /blog/netapp-nsanity/
date: "2011-05-25"
category: [howto, dos, esx, netapp, storage, tools, vmware]
published: true
---

I was asked a few days ago to run a tool against one of my ESX hosts. The tool is called nSANity and was supplied by my NetApp vendor. This application is designed to collect details of all SAN and fibre connectivity for end-to-end diagnosis. It takes this data and makes an XML report out of it. The reason I was requested to run this tool was so that data could be collected about a remote site and how it is currently connected to one of our NetApp SANs in preparation of a SAN data move to be performed by NetApp Professional Services.

The [link](http://now.netapp.com/NOW/download/tools/nsanity/){:target="_blank"} for the software was supplied and I went to go retrieve this tool. I'm providing the [link to the tool](http://now.netapp.com/NOW/download/tools/nsanity/){:target="_blank"}, but you will have to be a NetApp NOW member to download it. I get to the URL and start reading. I notice this right off the bat (see Fig 1). Hmm.... NetApp wants me to run this tool that they provide, yet they can not support you if you're having trouble. Which I read as, "if this poops your machine, you're on your own."

Further reading gave me no indication of what kind of data was to be collected. In addition, the documentation mentions that you have to run this via SSH to the ESX Host as root. My company's environment does not allow root access over SSH. It falls under the "No Fly Zone." It's forbidden, taboo, etc. Unfortunately, nSANity cannot be run locally on the ESX host. So I had to get an approval from Congress to allow root over SSH. (Ok, that's an exaggeradtion, but that's what it feels like.) Once I had the green light, I break out the fire extinguishers and prepare for the rapture of this ESX guinea pig.

The windows version of the tool runs very well on Windows 7. To fill in the blanks on how I ran this, I opened a command prompt (Run as Administrator) and then used the following command:

`c:\> nsanity -d c:\temp vmware://root:*@esxhost.domain.com`

The app runs for approximately ten minutes and collects configuration data on firewall rules, virtual switches, virtual nics, drives, cpu, processes, lspci, hbas, etc. Once the data was collected, root access via SSH had to be replugged. It seems harmless to the environment and the data collected doesn't seem to fall under any security risks.

If you are having to run nSANity and aren't familiar with the tool, I hope my experience with it provides some insight on how it works.
