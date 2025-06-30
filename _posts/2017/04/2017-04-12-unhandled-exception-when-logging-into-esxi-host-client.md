---
layout: post
title: "Unhandled Exception when logging into ESXi Host client"
summary: "Oops. ESXi host client throws an error."
author: samui
date: "2017-04-12"
category: [esx, vmware, troubleshooting, vcenter]
thumbnail: /images/2017/04/unhandled-exception.png
permalink: /blog/unhandled-exception-when-logging-into-esxi-host-client/
published: true
---

Ran into this weird error after powering up the C.R.I.B. and logging into the ESXi Host to boot up VMs. This error popped up while using Safari, Chrome, and Firefox.

Unhandled Exception (1) Cause: Error: [$rootScope:inprog] http://errors.angularjs.org/1.3.2/$rootScope/inprog?p0=%24digest

The error presents itself with two buttons: _Reload_ and _Details_. The picture displayed shows the details of the error that I got. You can hit reload, attempt to log back in, and rinse and repeat the process. Or you can select Details, where you can only 'close' the error and repeat the process.

A quick check with Professor Google finds this is an issue mentioned in the [VMware Communities](https://communities.vmware.com/thread/560937){:target="_blank"}. Luckily, the entry had an answer attached. (Thanks, "rshell"). It's not a fix, but it is an answer.

I can't explain why the error occurs, but I can explain what causes it to pop up. This error occurs when you land on the logon page, enter credentials, and then hit on the keyboard instead of clicking on the button. If you click the login button, you have no errors and vSphere loads normally.
