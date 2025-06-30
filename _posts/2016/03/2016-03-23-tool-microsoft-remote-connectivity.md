---
layout: post
title: "Tool: Microsoft Remote Connectivity"
summary: "A Microsoft tool that will validate the connection between workstation and office365."
author: samui
thumbnail: /images/2016/03/Screen-Shot-2016-03-23-at-9.02.33-AM-3-300x196.png
permalink: /blog/tool-microsoft-remote-connectivity/
date: "2016-03-23"
category: [tools, troubleshooting, vra, automation]
published: true
---

While onsite with a customer yesterday, I was introduced to a new free tool provided by Microsoft called the "Remote Connectivity Analyzer".

So what does this do? How can it be of use to me?

I'm glad you asked. Our scenario was that the company used Office365 for their emails. I've watched companies make the move to cloud provided email for years now. But where this gets to be problemsome, is when you need to configure applications to send/receive email. Our specific scenario involved configuring vRA to use Office365. Now, I already know that if Office365 is set up to use EWS (Exchange Web Services) this won't work. So boys and girls, if the customer uses Office365 and they are setting up vRA - ask them if O365 is using EWS. If so, stop. Save yourself a headache.

But back to the problem. So when we go to configure the email server (inbound or outbound), we only have the ability to plug in the configuration and click a little "Test Connection" button. If it fails, you get a generic error message and then you have to scour logs to try and decrypt what went wrong. Do you not have enough ports open? Are they blocking subnets? Is the email account enabled? Now you have to try and troubleshoot the problem with multiple teams.

Our first foray was to try and check if we could ping outlook.office365. We SSH'd into the vRA Cafe and pinged. We were good. We attempted telnet by running the command: curl -v telnet://outlook.office365.com:993. We were good. We had the network team verify that traffic was making it to and from. Again, good.

The next step was to validate the email configuration. The email team checked over our settings -- which surprise, we're good. Then they introduced us to the [Remote Connectivity Analyzer](http://testconnectivity.microsoft.com){:target="_blank"}. Using this tool, they were able to see test the connection between the admin's workstation and outlook.office365. What we found was that the email account was set up as a shared email mailbox, and not an individual user box.

But what was fascinating was that the analyzer ran tests to check EVERYTHING. So the next time you run into issues with trying to configure email, give this a shot.

_FINE PRINT: The test machine will need to have internet access to give this a whirl._
