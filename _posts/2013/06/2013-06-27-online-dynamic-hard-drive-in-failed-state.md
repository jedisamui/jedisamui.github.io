---
layout: post
title: "Online Dynamic Hard Drive in Failed State"
summary: "RDMs and Dynamic Hard Drives"
author: samui
thumbnail: /images/2013/06/hard-disk-become-dynamic.png
permalink: /blog/online-dynamic-hard-drive-in-failed-state/
date: "2013-06-27"
category: [howto, p2v, windows, svmotion, troubleshooting]
published: true
---

I ran into a new situation. Here's the setup, I'm at a customer's location performing a migration of VMs from one SAN array to another (svmotion). Ok, Not that hard - not really. As soon as you let your guard down -- even on the simplest of things, that's when you get bit by something or you discover a new and challenging puzzle.

The migration consisted of converting VMs with Physical mode RDMs to vmdks. (Again, not that hard). The key thing here is that they were physical and so I needed to take the VM offline for the migration. Yeah, I know I could have converted it to virtual mode and did it online -- but trust me when I say, it was better to do it offline. After a grueling 24 hour move, the RDMs were now vmdks. YAY! Hoorah!

-- Not so fast....

I had the customer boot up the VM for verification. Did the VM survive the move, and does your app perform normally? After a few minutes, his answer was 'no'. uh-oh. What is wrong?

Apparently, when the RDMs were created, they were added to Windows as a 'Dynamic Hard Drive'. Who knows if this was something another vendor did, or whether or not it was done originally. But when we opened "Disk Management", the Dynamic Hard Drives were showing "Online" but in a failed state. What the heck does this mean? In this case, it just meant the drive had become unavailable and needed to be reactivated.

The fix? Believe it or not, right click the drive and select "Reactivate Disk". The 2TB drive took all of about 2 seconds to reactivate and everything started working EXACTLY as it should have.

Here's the link that I followed to resolve the issue: [http://technet.microsoft.com/en-us/library/cc732026.aspx](http://technet.microsoft.com/en-us/library/cc732026.aspx){:target="_blank"}
