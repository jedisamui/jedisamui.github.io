---
layout: post
title: "How to upgrade a simple vRA 7.0 instance to vRA 7.0.1"
summary: "A walkthrough of a vRA upgrade."
author: samui
date: "2016-03-18"
category: [lab, howto, project, vmware, vra, automation]
thumbnail: /images/2016/03/vRA7-Updates-logo-260x250.png
permalink: /blog/how-to-upgrade-a-simple-vra-7-instance-to-vra-next/
published: true
---

Just this week, VMware released vRealize Automation 7.0.1 (vRA). It contains many bug fixes and some enhancements to the vRA platform. I was excited for it to come out and was anxious to perform an upgrade in my home lab.

I will advise caution and planning in any upgrade of your environment. But I would stress heavily on the planning. You should know your dependencies before you attempt an upgrade, and always. ALWAYS, read the [Release notes](http://pubs.vmware.com/Release_Notes/en/vra/vrealize-automation-701-release-notes.html){:target="_blank"} before you start the upgrade process.

The following process is for a simple vRA instance. This is the Proof Of Concept build, sometimes referred to as a "Lab" or "Sandbox" build. However, these steps can be modified for a fully distributed vRA instance.

Here is how I upgraded my lab.

1) Take snapshots of the vRA Cafe Appliance, IaaS VM, and SQL VM.

2) Shutdown the vRA Services      SSH into the vRA Cafe Appliance and shutdown the vco-server, vcac-server, apache2, and the rabbitmq-server services.

1. Run the below commands to stop the above listed services:
    
    - #service vcac-server stop
    - #service apache2 stop
    - #service rabbitmq-server stop
    - #service vco-server stop
    
      
    You can check that the services have stopped using the status syntax: _#service vco-server status_
2. Log into the IaaS Virtual Machine and stop the below listed vRA services.
    - All VMware vCloud Automation Center agents
    - All VMware DEM Workers
    - VMware DEM Orchestrator
    - VMware vCloud Automation Center Manager Service

3) Download the [vRealize Automation Appliance 7.0.1 Update Repository Archive ISO](https://my.vmware.com/web/vmware/details?downloadGroup=VRA-701&productId=561&rPId=10426){:target="_blank"}.

4)Upload the ISO to a datastore, and mount the iso to the vRA Cafe Appliance's CDRom.

![VM Settings](/images/2016/03/Screen-Shot-2016-03-09-at-12.36.09-PM.png)
5) Open a browser and log into the vRA Cafe. Then Navigate to the _"Update Tab"_ -->> _"Settings"_.

6) Change the Update Repository to _"Use CDRom Updates"_. Click on _"Save Settings"_. 
![Use CDRom Updates](/images/2016/03/Screen-Shot-2016-03-09-at-12.36.58-PM-300x194.png)

7) Select the _"Status Tab"_.

8) Click on _"Check For Updates"_. 
![Check For Updates](/images/2016/03/Screen-Shot-2016-03-09-at-12.38.28-PM-300x175.png)

9) An update should be found (as shown in the photo above). Click on _"Install Updates"_.

10) Wait for the update to complete. This took approx 30 minutes for my lab. 
![Install Updates](/images/2016/03/Screen-Shot-2016-03-09-at-12.40.26-PM-300x66.png)

11) Once the updates complete, you will be notified to reboot the vRA Cafe Appliance. 
![Reboot Notice](/images/2016/03/Screen-Shot-2016-03-09-at-1.11.05-PM-300x291.png)

12) Once the vRA Cafe Appliance has completed the reboot, log back into the vRA VAMI and verify the version. 
![Updated Version](/images/2016/03/Screen-Shot-2016-03-09-at-1.16.52-PM-300x95.png)

This completes the vRA Cafe Appliance upgrade. Now it is time to focus on the IaaS Server.

13) Open a console or RDP session into the IaaS Server and log into the machine with the vRA Administrator Service Account.

14) Open a web browser and browse to the vRA Cafe Installer page. _"https://\[vRA Appliance FQDN\]:5480/installer"_

15) Download the _"DBUPGRADE SCRIPTS"_.

16) Verify the Java Path in the Environmental variables. 
![Java Path](/images/2016/03/Screen-Shot-2016-03-09-at-1.23.01-PM-268x300.png)

17) Open the File Explorer and browse to the folder where you downloaded the _"DBUPGRADE.zip"_ scripts file. Extract the DBUpgrade.zip file.

18) Open an elevated Command Prompt.

19) Change the directory to the location of the DBUpgrade Extraction Folder.

20) **NOTE:** Verify that the vRA Administrator Service Account has the SQL sysadmin role enabled.

21) Run the following command to update the SQL Database:       # dbupgrade -S sql.dwarf.lab -d vra -E -upgrade

Replace sql.dwarf.lab with the FQDN of your SQL server. 
![DBUpgrade Script](/images/2016/03/Screen-Shot-2016-03-09-at-1.28.59-PM-300x64.png)
The process may take a few minutes to complete.

22) Return to the vRA Cafe Installer page. _"https://\[vRA Appliance FQDN\]:5480/installer"_. Download _"IaaS\_Setup"_.

23) Browse to the downloaded file in File Explorer. Right-Click the file, and "Run as Administrator".

![vRA 7.0.1 IaaS Installation - 1](/images/2016/03/Screen-Shot-2016-03-09-at-1.32.07-PM.png)

![vRA 7.0.1 IaaS Installation – 2](/images/2016/03/Screen-Shot-2016-03-09-at-1.32.17-PM.png)

![vRA 7.0.1 IaaS Installation – 3](/images/2016/03/Screen-Shot-2016-03-09-at-1.32.33-PM.png)

24) Select _"Upgrade"_ 
![vRA 7.0.1 IaaS Installation – 4](/images/2016/03/Screen-Shot-2016-03-09-at-1.32.42-PM.png)

![vRA 7.0.1 IaaS Installation – 5](/images/2016/03/Screen-Shot-2016-03-09-at-1.33.38-PM.png)

25) Fill in the Blanks. 

![vRA 7.0.1 IaaS Installation – 6](/images/2016/03/Screen-Shot-2016-03-09-at-1.35.31-PM.png)

**NOTE:** For the SQL Connection. If you are not using SSL, uncheck the option to _"Use SSL for Database Connection"_; else you will experience the following error. 

![vRA 7.0.1 IaaS Installation – 7](/images/2016/03/Screen-Shot-2016-03-09-at-1.38.04-PM-300x123.png)

26) For my lab, I had to remove the SSL connection between the IaaS Server and the SQL Database Server. 

![vRA 7.0.1 IaaS Installation – 8](/images/2016/03/Screen-Shot-2016-03-09-at-1.38.33-PM.png)

![vRA 7.0.1 IaaS Installation – 9](/images/2016/03/Screen-Shot-2016-03-09-at-1.39.31-PM.png)

![vRA 7.0.1 IaaS Installation – 10](/images/2016/03/Screen-Shot-2016-03-09-at-1.39.44-PM.png)

![vRA 7.0.1 IaaS Installation – 11](/images/2016/03/Screen-Shot-2016-03-09-at-1.39.54-PM.png)

27) The upgrade installation will take some time to complete. I recommend going and grabbing a drink. The process took approx 30 mins for me for it to complete. 

![vRA 7.0.1 IaaS Installation – 12](/images/2016/03/Screen-Shot-2016-03-09-at-1.55.57-PM.png)

28) The upgrade finishes. 

![vRA 7.0.1 IaaS Installation – 13](/images/2016/03/Screen-Shot-2016-03-09-at-1.56.08-PM.png)

29) Click finish and reboot the IaaS Server.

30) When the server comes back online. Log back in and verify that all vRA services have restarted. 

![vRA Services](/images/2016/03/Screen-Shot-2016-03-09-at-2.00.59-PM.png)

31) Log back into the vRA Cafe Appliance and check all Services are returned to _"Registered"_. 

![Cafe Services](/images/2016/03/Screen-Shot-2016-03-09-at-2.02.36-PM-267x300.png)


If everything happened without any issues, then you have successfully upgraded vRA from 7.0 to 7.0.1. Go log into your portal and check it out! 

![vRA Portal Login](/images/2016/03/Screen-Shot-2016-03-09-at-2.03.30-PM-291x300.png)
