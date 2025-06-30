---
layout: post
title: "Disable Flow Control"
summary: "How to disable flow control and make it stick"
author: samui
thumbnail: /images/2013/11/hpchassis.jpg
permalink: /blog/disable-flow-control/
date: "2013-11-25"
category: [esx, howto, vmware, network, ssh, troubleshooting, vmware-kb]
published: true
---

I was onsite this past weekend doing a vSphere 5.1 turnkey install. The turnkey consisted of a C7000 Blade Chassis, and two ESXi 5.1U3 hosts. With this new installation, the customer was adding to a new 10Gbe environment to his network. There were also a few settings that needed to be made, specifically, disabling Flow Control on the ESXi Hosts for Mgmt, Data, and vMotion.  
The Flex10 Virtual Connect Modules in the chassis do not have a method of disabling flow control, each ESXi Host would auto-negotiate with Flow Control On for both transmit and receive. Personally, I've never had to worry about Flow Control within the environments that I've set up. But on this engagement, I was working side by side with a guy who lives, breathes, and retransmits network. Trust me, he knows his stuff. It was his recommendation to disable Flow Control. Realizing I had to figure this out quickly, I turned to the number one tool in my toolbag - Professor Google.  
Flexing my "Google-Fu", I found my answer right away: [VMware's KB Article 1013413](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1013413){:target="_blank"}. I opened an SSH session, and ran this command  

**# ethtool --pause tx off rx off**  

That's Great! One problem. When I reboot, Flow Control is going to reenable because of the auto-negotiate. So how do I make this persistent/permanent? Another well versed Google search brought me to [VMware KB Article 2043564](http://kb.vmware.com/selfservice/microsites/search.do?cmd=displayKC&docType=kc&docTypeID=DT_KB_1_1&externalId=2043564){:target="_blank"}. Once this change was applied, reboots did not reenable Flow Control. My network guy was happy, and hey - I learned something new.  
  
So the next question then, is how do I add this change into a kickstart file. Hmm..... Stay tuned folks. I'll see if there's a way to do that.
