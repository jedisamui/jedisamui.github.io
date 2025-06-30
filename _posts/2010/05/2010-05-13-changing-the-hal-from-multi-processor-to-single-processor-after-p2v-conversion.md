---
layout: post
title: "Changing the HAL from Multi processor to Single processor after P2V conversion"
summary: "Reference on how to adjust CPUs for a VM"
author: samui
thumbnail: /images/2010/05/cpu.png
permalink: /blog/changing-the-hal-from-multi-processor-to-single-processor-after-p2v/
date: "2010-05-13"
category: [esx, p2v, vmware]
published: true
---

1. Before making any changes to the VM, Create a Snapshot in Virtual Center. 
2. RDP or Console into the Server and go to Device Manager 
3. For Windows Server 2003, install the following hotfix. It does not require a reboot. This hotfix allows you to downgrade to Uniprocessor. WindowsServer2003-KB923425-v2-x86-ENU.exe (updated 08.01.08 – you do not need the hotfix if you have an APIC HAL. If you have an APCI HAL the option to downgrade the HAL described in the next steps is not available and you must apply the hotfix before continuing.) 
4. Expand the Computer tab to see what HAL is loaded 
5. Right click on ACPI Multiprocessor PC and choose Properties 
6. Click the Driver tab 
7. Click Update Driver 
8. Click Next 
9. Choose Display driver [in order to] choose and click Next 
10. Choose "Show all hardware of this device class" 
11. Choose ACPI Uniprocessor PC and click Next 
12. You will be warned about changing this driver, but Click Yes to continue 
13. Click Next 
14. Click Finish 
15. Click Close 
16. You will be asked to reboot the computer. 
17. If the Computer reboots cleanly and everything seems to work, clean up the Snapshot you took. 
18. Verify in Device Manager that the HAL has changed.
