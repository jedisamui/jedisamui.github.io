---
layout: post
title: "Script: Drive Letter Order"
summary: "script to place windows drive letters in specific order"
author: samui
thumbnail: /images/2011/05/poweroff_bat.jpg 
permalink: /blog/script-drive-letter-order/
date: "2011-05-12"
category: [project, vmware, howto, powercli, powershell, script]
published: true
---

I'm currently writing an application (it's a huge script - 2100 lines of code) in powershell to automate a weekly VM refresh operation that my team does every Saturday. At the end of the script, I wanted to check the newly deployed virtual machines to verify that the automated scripts did what they were supposed to. One of the items on this checklist is to check the drive letters. For the business owners, the VM has to be setup similar to the source machine. The drive letters have to be setup as follows: C Drive (OS), D Drive (CD Rom), E Drive (Data).

I use VMware's clone ability to copy the source VM into a template. I use a "CustomizationSpec" to reset the VM's identifiers and "sysprep" the virtual machine. During the "sysprep" process, the drive letters are reset to: C Drive (OS), D Drive (Data), E Drive (CD Rom). Since this doesn't meet the customer's requirement, I created a batch script that is called during the runonce section to change the drive letters. This batch script was placed on the source machine with its's support files, and is thus carried forward in the clone.

There are three files that are placed on the source VM - poweroff.bat, drvltrs.bat, and assign.txt. Poweroff.bat is called during runonce and is designed to check for the second batch file, drvltrs.bat. If that file exists, then it is called to run. Once drvltrs.bat has completed, poweroff.bat renames drvltrs.bat so that it can not be called again, and shuts down the virtual machine.

Here's what each batch file looks like. These files can be downloaded from the links below.

**POWEROFF.BAT**

```
@echo off
REM ======================================
REM ==
REM ==     POWEROFF.BAT
REM ==     Script to Shutdown VM
REM ==     by J. Sam Aaron
REM ======================================

@echo on
REM == Change Drive Letter Order
IF EXIST "c:\windows\refreshfiles\drvltrs.bat" CALL "c:\windows\RefreshFiles\drvltrs.bat"

REM == Renames the Drive Letter Batch File
ren "c:\windows\refreshfiles\drvltrs.bat" drvltrs.txt

REM == Shuts down machine
shutdown /s /t 2

```

  
**DRVLTRS.BAT**

```
@echo off
REM == Uses DiskPart to change the Drive Letter Order
"c:\windows\system32\diskpart.exe" /s "c:\windows\refreshfiles\assign.txt"

```

  
**ASSIGN.TXT**

```
select volume D
assign letter F
select volume E
assign letter D
select volume F
assign letter E

```

At the end of the deployment process, the virtual machine should have the following drive letters set: C Drive (OS), D Drive (CD Rom), E Drive (Data). However, you never know when tragedy is going to strike. So I needed a double-check in my Powershell script to occur at the end of the refresh. Since I am still learning Powershell, I turned to google to help me with finding an answer. After a few clicks, I found something I could use.

The following code was written by Tobias and posted on [Powershell.com](http://powershell.com/cs/media/p/7924.aspx){:target="_blank"} on 10-20-2010. It was pretty close to what I was looking for.

**CHECK DRIVE LETTER**

```
function Combine-Object 
{ 
    param($object1,$object2) 
    trap 
	{ 
        $a = 1 
        continue 
    } 
    $propertylistObj1 = @($object1 | Get-Member -ea Stop -memberType *Property | Select-Object -ExpandProperty Name) 
    $propertylistObj2 = @($object2 | Get-Member -memberType *Property | Select-Object -ExpandProperty Name | Where-Object { $_ -notlike '__*'}) 
    $propertylistObj2 | ForEach-Object 
	{ 
        if ($propertyListObj1 -contains $_) 
		{ 
            $name = '_{0}' -f $_ 
        } else 
		{ 
            $name = $_ 
        } 
        $object1 = $object1 | Add-Member NoteProperty $name ($object2.$_) -PassThru 
    } 
    $object1 
}  

function Get-Drives 
{ 
	Get-WmiObject Win32_DiskPartition  -computername "$checklabelvm" -Credential $creds | 
    ForEach-Object 
	{ 
        $partition = $_ 
        $logicaldisk = $partition.psbase.GetRelated('Win32_LogicalDisk') 
        if ($logicaldisk -ne $null) 
		{ 
            Combine-Object $logicaldisk $partition  
        } 
	} | select-Object Name, VolumeName, DiskIndex, Index 
}
	
## CHECK DRIVE CONFIGURATION
write-host "Drive Configuration: "
$drives = get-drives
foreach ($drive in $drives)
{
	if ($drive.name -eq "D:")
	{
		write-host $drive.name "has not been changed." -foregroundcolor red
	}
	else 
	{
		write-host $drive.name "CHECK!" -foregroundcolor green
	}
}

```

So I copied his code and added the check that I wanted. I wanted to check and verify if "D" was a logical disk. If so, then we have a fail and the drive letters would have to be manually changed. If not, then we have a pass ("CHECK!") and we meet this requirement from the customer.
