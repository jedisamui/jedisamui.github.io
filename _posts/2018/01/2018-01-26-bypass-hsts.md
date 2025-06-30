---
layout: post
title: "Bypass HSTS"
summary: "annoying security.... grumble, grumble, grumble"
author: samui
date: "2018-01-26"
category: [howto, tips]
thumbnail: /images/2018/01/Screen-Shot-2018-01-26-at-5.38.02-PM-300x203.png 
permalink: /blog/bypass-hsts/
published: true
---

While I am an avid supporter of encryption and security, I do have to admit... getting the dreaded HSTS error within the Chrome browser sucks. When this happens, one of the quickest easiest "hacks" (tips!) to bypass it is to just type "badidea" on your keyboard. And voila! HSTS is bypassed.

I learned of this trick from the [following blog post](https://scotthelme.co.uk/bypassing-hsts-or-hpkp-in-chrome-is-a-badidea/){:target="_blank"} by Scott Helme. He details how to do this, but more importantly why it's not really a good idea. If you are getting an HSTS error, then there is obviously something wrong with your SSL Certificates and you should investigate. 

![Bypass HSTS](/images/2018/01/Screen-Shot-2018-01-26-at-5.38.02-PM.png)
