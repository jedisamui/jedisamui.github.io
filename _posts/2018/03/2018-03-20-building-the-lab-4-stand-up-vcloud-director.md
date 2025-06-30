---
layout: post
title: "Building the lab 4: Stand up vCloud Director"
summary: "Continuing the lab build out. Focusing on vCloud Director (VCD)."
author: samui
date: "2018-03-20"
category: [lab, howto, project, vcloud, vcloud-director, vmware]
thumbnail: /images/2018/03/vcd.png
permalink: /blog/building-the-lab-4-stand-up-vcloud-director/
published: true
---

First, I would like to declare: "vCloud Director is NOT dead!" I can say emphatically, this product did not die, never died, and I don't believe that it is going to die! It is still actively being developed by VMware.

With this clarified, let's move on to getting vCD stood up. Again, I followed along with the [wonderful guide from Sysadmin Tutorial](http://www.sysadmintutorials.com/tutorials/vmware-vcloud-director/vmware-vcloud-director-lab-database-setup-part-1/){:target="_blank"}.

This guide has a very good walk-through for standing up vCloud Director 8.0 for a Proof of Concept (it also works well for 9.0). There are multiple steps that break out each milestone of the installation/deployment. You could follow along each part, as I did. Along the way, I will point out the various things that I did or changed for my environment.

[Part One](http://www.sysadmintutorials.com/tutorials/vmware-vcloud-director/vmware-vcloud-director-lab-database-setup-part-1/){:target="_blank"} is self explanatory. The walkthrough shows you how to set up a SQL database. Yes, MS SQL is still supported with vCD 9.0. While you may want to migrate or move to a PostGreSQL Database, this guide sets you up for MS SQL. (I will cover how to setup PostGreSQL and migrate the database sometime in the future. You may need or want this down the road when you get ready to upgrade.)

[Part Two](http://www.sysadmintutorials.com/tutorials/vmware-vcloud-director/vmware-vcloud-director-lab-rabbitmq-install-part-2/){:target="_blank"} - setting up a RabbitMQ server, I skipped. Why do you ask? Well, the answer is selfish. My environment is small and is designed for one thing - quick deployment and stand up of an SDDC environment for play and discovery. Unlike many vCD environments that can be found in the wild, I will not be interfacing or integrating with any outside services. Nor will I be standing up mulitple cells. So I have no need of a RabbitMQ server at this time. You and your environment may very well need one.

[Part Three](http://www.sysadmintutorials.com/tutorials/vmware-vcloud-director/vmware-vcloud-director-lab-vcloud-director-install-part-3/){:target="_blank"} of this guide is very good. I like how they dig into the certificate creation and the details of what to do with them. This portion of the walkthrough also includes how to create the cert with a Microsoft CA server. These are details that I would like to see VMware include in their documentation. This is one area that plagues many installations as certificates always seem to be problematic and having a good walkthrough would really go a long way.

Once you complete these steps, you are ready to configure vCloud Director for consumption. Like all VMware products, you should have a good idea of how or what you want to do. Setting this up to play with is one thing. But if you are trying to utilize it beyond "how do I install it?", then you need to have an idea of what you are trying to accomplish. If you haven't taken the time to do this, you should.

For me, as I said previously - I want to stand up vCloud Director to be a mechanism where I can quickly deploy full SDDC environments to manipulate and play with. I want to utilize these environments to learn, discover, and grow my skillset. I do not want to destroy and rebuild my lab environment every time I have a different scenario I want to test. My goal is to 'mimic' the Hands On Lab environment. Ambitious? Yes.

I'm going to stop here as the next [Part of the SysAdmin Tutorial walkthrough](http://www.sysadmintutorials.com/tutorials/vmware-vcloud-director/vmware-vcloud-director-lab-nsx-install-and-configure-part-4/){:target="_blank"} was already covered when I stood up NSX in "Building the lab 3: NSX". Before I continue with the SysAdmin Tutorial on and kick off Part 5, I want to set up more storage.
