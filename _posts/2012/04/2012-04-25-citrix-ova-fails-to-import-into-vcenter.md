---
layout: post
title: "Citrix OVA fails to import into vCenter"
summary: "Oops. Citrix ova fails to deploy"
author: samui
thumbnail: /images/2012/04/citrix-logo.png
permalink: /blog/citrix-ova-fails-to-import-into-vcenter/
date: "2012-04-25"
category: [howto, vmware, troubleshooting, vmware]
published: true
---

I wanted to pass along this tidbit of info that I learned today.

I was asked to deploy Citrix's Merchandising Server 2.2 OVA (citrix-merchandising-server-2.2.0-2852\_VMware.ova) and while trying to import it into vcenter, I ran into a problem. The import would seem to move along fine -- until it got to about 96%, then it would fail with the error:

**"Failed to deploy OVF package: The request was aborted: The request was canceled." Recent Tasks shows "The task was canceled by a user."**

A quick google search landed me on this [Citrix Knowledgebase page](http://forums.citrix.com/thread.jspa?threadID=296091){:target="_blank"}. This was exactly my issue -- down to the "Invalid Certificate" message in the OVF import wizard.

The post definitely seemed hopeful, and ultimately, I tried "besemann's" method. This involved me opening the OVA with 7zip and removing the certificate.

Here's the steps I took. 
1) Opened Windows Explorer, browsed to the location of the OVA file. 
2) Right-Clicked the OVA and chose to open the OVA file in 7Zip. 
3) I browsed to the ".cert" file and then deleted it. 
4) Reattempt the import.

While this method did not seem to help the last commentor on the post, it resolved my issue on the first try. I got the OVA imported and all is rainbows and unicorns.
