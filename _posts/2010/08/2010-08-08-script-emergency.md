---
layout: post
title: "Script: Emergency"
summary: "Reference script to power everything off"
author: samuui
thumbnail: /images/2011/01/emergency-shutoff.jpg
permalink: /blog/script-emergency/
date: "2010-08-08"
category: [esx, script, vmware]
published: true
---

I was asked to write a script that I could run locally on an ESX server that would scan the state of the VMs and then try to gracefully power off the VMs that were running. After the VMs are powered off, it would then power off the ESX server.

The reason the script was needed was because I had to do maintenance on the ESX Server and I needed a more automated way to bring down the environment. In this particular environment, each ESX server had locally stored VMs.

This script was written for an ESX 3.5 environment. Graceful shutdowns will require VMware Tools installed on each VM.

```
#!/bin/bash
# emergencyshutdown.sh
# Written By Sam Aaron 

while :
do
     clear
     echo  "****************************************************";
     echo "*** This is an EMERGENCY STOP of the ESX System. ***";
     echo  "****************************************************";
     echo "1. Power off all VMs and the ESX System";
     echo "2. Exit";
     echo -n "Please enter option [1 or 2]";
     read opt
     case $opt in
     1) echo "************************************";
     echo "*** All VMs will be powered off. ***";
     echo "************************************";

     for vm in `vmware-cmd -l`;
          do
               vmx="`basename ${vm}`";
               vmware-cmd "$vm" stop trysoft;
               echo "This ${vm} has been powered off.";
          done;

     echo  "************************************************";
     echo "*** Verifying all VMs have been powered off. ***";
     echo  "************************************************";

     for vm in `vmware-cmd -l`;
          do
               vmx="`basename ${vm}`";
               state="`vmware-cmd -q ${vm} getstate`";
               echo "VM: ${vmx} is currently ${state}.";
          done;

     echo "************************************";
     echo "*** Powering off the ESX Server. ***";
     echo "************************************";

     shutdown -h now;
     exit $?;;

     2) echo "$USER has terminated the EMERGENCY STOP program.";
     exit 1;;
     *) echo "$opt is an invalid option. Please select option 1 or 2 only.";
     echo "Press [enter] key to continue. . .";
     read enterKey;;

     esac
done

```

I have revamped and rewritten this script.
