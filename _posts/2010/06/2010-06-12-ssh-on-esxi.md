---
layout: post
title: "SSH on ESXi"
summary: "How do I enable SSH?"
author: samui
thumbnail: /images/2010/06/ssh.png
permalink: /blog/ssh-on-esx/
date: "2010-06-12"
category: [esx, ssh, vmware]
published: true
---

I've had to google this too many times. So I figure if I make it a post in my blog, I might retain the information.

1. Go to the ESXi console and press alt+F1 
2. Type: **unsupported** 
3. Enter the root password(No prompt, typing is blindly) 
4. At the prompt type “**vi /etc/inetd.conf**” 
5. Look for the line that starts with “**#ssh**” (you can search with pressing “/”) 
6. Remove the “**#**” (press the “x” if the cursor is on the character) 
7. Save “**/etc/inetd.conf**” by typing “**:wq!**” 
8. Restart the management service “**/sbin/services.sh restart**”
