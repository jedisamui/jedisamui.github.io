---
layout: post
title: "How to P2V linux into VMware ESX Server"
summary: "Instructions to convert a linux machine to a VM"
author: samui
thumbnail: /images/2012/03/VMWare-Converter-01.jpg
permalink: /blog/how-to-p2v-linux-into-vmware-esx-server/
date: "2012-03-01"
category: [howto, p2v, linux]
published: true
---

I stumbled on this article a while back and stored it on a thumb drive. The original article was posted by Vladan Seget. It can be found [here](http://www.vladan.fr/how-to-p2v-linux-into-vmware-esx-server/){:target="_blank"}. If you do a lot of P2V conversions, I would recommend giving this a read over.

  
The steps you need to do: 
01.) [Download Vmware Converter for linux here](http://www.vmware.com/download/converter/){:target="_blank"}. 
02.) Extract the converter package on your linux server:

- **# tar xf VMware-converter-4.0.3-u4.tar.gz**

03.) Start the installation

- **# cd vmware-converter-distrib/ && ./vmware-install.pl**

04.) You can just accept the majority of the options just make sure that you activate the remoteaccess

- **Do you want to enable remote access in Converter Standalone Server? yes**

05.) Usually a linux server has an Apache installed on port 80, so good think to do is to add aditional port for http proxy :

- **What port do you want the HTTP proxy to use? [80] 8080**
- **What port do you want the HTTPS proxy to use? [443] 444**

06.) Also you should go and allow root to login localy

- **#vi /etc/ssh/sshd\_config**

change the line

- **PermitRootLogin yes**

07.) Then you just leave your linux server and go to your Windows XP or Vista Workstation and connect to your linux server remotely via http for example **“http://192.168.0.1:444”** 
08.) Download and install the VMware Converter client. 
09.) Start the Converter Client and go to **Administration > Connect to another server**

![1](/images/2012/03/1-300x148.jpg)

10.) In the credentials window just enter the **IP address** followed by the **port number**, then the **root login** and **password**.

![2](/images/2012/03/2-291x300.jpg)

11.) Enter the credentials once more and choose Linux as an OS family.

![3](/images/2012/03/3-300x223.jpg)

12.) Then just follow the assistant for the rest of the steps. As a destination you choose the ESX server IP address or hostname. There is nothing difficult with that…
