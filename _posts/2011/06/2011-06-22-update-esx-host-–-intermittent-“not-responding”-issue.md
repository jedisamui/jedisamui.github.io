---
layout: post
title: "Update: ESX Host – Intermittent “Not Responding” Issue"
summary: "Oops...ESX host throws an error during update"
author: samui
thumbnail: /images/2011/06/esxi-not-res.png
permalink: /blog/update-esx-host-intermittent-not-responding-issue/
date: "2011-06-22"
category: [daily]
published: true
---

There is an update and a resolution for the "ESX Host – Intermittent “Not Responding” Issue".

We recently upgraded the virtual center server from 4.0 to 4.1. When the upgrade completed, the VC pushed out updated vcagents and upgraded the vpxuser on each host within the environment. Afterwards, the connectivity issues vanished. It's been about two weeks now, and I haven't see the error not even once.

I'm thinking that somehow the vcagent got corrupted. But if this were the case, then removing and readding the host to the vc should have resolved this issue a while back. It unfortunately did not. I suspect that the underlying issue was a combination between the vcagent and a DB entry. That could explain why removing and readding the host did not resolve the issue.

Unfortunately, I doubt that I will ever find the true cause as the environment has now changed. Hopefully this information will help someone if they are having the same issue.
