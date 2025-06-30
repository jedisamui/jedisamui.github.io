---
layout: post
title: "Windows 7 & Telnet"
summary: "Where is the telnet client? How can I get it?"
author: samui
thumbnail: /images/2011/03/telnet.png
permalink: /blog/windows-7-telnet/
date: "2011-03-14"
category: [windows]
published: true
---

Windows 7 – Enable Telnet

It’s very rare that I use Telnet these days, so it took a long time for me to notice that by default it was not packaged with Windows 7. I did some research and found out that even hyperterminal was removed from Windows 7. More than likely this was an attempt to make Windows more secure by default, as Telnet is very insecure and whenever you have the choice you should always use SSH. However, with that being said, you can quickly re-enable Telnet by following these steps:

- Start
- Control Panel
- Programs And Features
- Turn Windows features on or off
- Check Telnet Client
- Hit OK

After that you can start Telnet via Command Prompt.
