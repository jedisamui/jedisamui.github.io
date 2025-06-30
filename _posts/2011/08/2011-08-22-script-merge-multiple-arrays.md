---
layout: post
title: "Script: Merge Multiple Arrays"
summary: "script mechanism that will merge array data"
author: samui
thumbnail: /images/2011/08/powercli.png
permalink: /blog/script-merge-multiple-arrays/
date: "2011-08-22"
category: [howto, project, vmware, esx, powershell, script, vmware]
published: true
---

Each Saturday my company does a refresh of a few different machines. This refresh consists of cloning a couple of production machines into our lab environment for development. It's a very tedious process that takes the entire day to do. Until recently, this process has been done manually. I've been working on a script to automate the process as much as possible. Along the way, I've learned a great deal about powershell/powercli - especially from hitting a wall and trying to figure out a way around. This blog entry is about one of those snags along the path of progress.

The refresh consists of two major Acts. Act I is all about cloning the source VMs, stripping them down, and converting them into templates. Act II is about deploying VMs using differing templates. There will be a later entry discussing this entire script in greater detail. This entry focuses on creating merged array to be used in the deployment CSV list for Act II.

Each week, the refresh list changes. One week, the source VMs may be SourceVM01 & SourceVM05. Next week, it may be SourceVM01 & SourceVM03. I created a GUI frontend for this script using PrimalForms 2011. The GUI has a "pick-list" where you can cherry pick what SourceVMs and what DestinationVMs will be needed. Upon your selection, three arrays are built - $a = #VM Names, $b = #Specs, $c = #Template Names.

For example; If I needed to refresh DEV-VM01, TEST-VM01, and QA-VM01; I'd check their boxes, and the following variables would be added to the three arrays. (You can start to see where if you had multiple Dev and Test VMs how these arrays can get quite complex.)

| DEV-VM1 | TST-VM1 | QA-VM1 |
| $avm = "DEV-VM1" | $avm = "TST-VM1" | $avm = "QA-VM1" |
| $bspec = "DEV" | $bspec = "TST" | $bspec = "QA" |
| $ctmp = "TMPL-DEV01" | $ctmp = "TMPL-TST01" | $ctmp = "TMPL-QA01" |

So the question was, how do you take this information and merge it - with one array, and all the information within? That's where I came up with this solution. I replay each of the arrays using a "counter" ($i) and building a new array with the information from the table above. The counter variable matches each record for that row (ie. $a.record1 with $b.record1 and $c.record1).

```
$d=@()
for($i=0;$i -lt $a.count;$i++)
	{
	$row = "" | Select-Object VMName, Spec, Template
	$row.VMName = "$($a[$i])"
	$row.Spec = "$($b[$i])" 
	$row.Template = "$($c[$i])"
	$d+=$row
	} 

```

The array will continue to loop, until it runs out of entries based on the $a array. Once this fourth array ($d) has been created, I can then use it to build my deployment CSV file (SEE THE DEPLOYMENT CSV LIST BUILD SCRIPT - To come).
