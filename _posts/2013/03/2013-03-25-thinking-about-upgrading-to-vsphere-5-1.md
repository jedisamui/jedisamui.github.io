---
layout: post
title: "Thinking About Upgrading To vSphere 5.1?"
summary: "Environment assessment tool for vSphere 5.1 readiness"
author: samui
thumbnail: /images/2013/03/vmware_vsphere.png
permalink: /blog/thinking-about-upgrading-to-vsphere-5-1/
date: "2013-03-25"
category: [vmware, powershell, script, tools]
published: true
---

If you haven't made the jump to vSphere 5.1, then you may want to try this new fling out (vCenter 5.1 Pre-Install Check Script) from Alan Renouf before upgrading.

(*description borrowed from [the Fling's website](http://labs.vmware.com/flings/vcenter-5-1-pre-install-check-script){:target="_blank"}) This fling is a PowerShell script that when run will help customers validate their environment and assess if it is ready for a 5.1.x upgrade. The script checks against known misconfigurations and issues raised with VMware Support. This script checks the Windows Server and Active Directory configuration and provides an on screen report of known issues or configuration issues, the script also provides a text report which can help with further trouble shooting.

For more information, read New Fling: [vCenter 5.1 Pre-Install Check Script](http://blogs.vmware.com/vsphere/2013/03/new-fling-vcenter-5-1-pre-install-check-script.html){:target="_blank"}.

As a reminder, all checks are read-only and no changes are made to the system.

I use many of Alan's scripts during my customer engagements and this is one more that I have thrown into my bag for future health checks and upgrade validations.
