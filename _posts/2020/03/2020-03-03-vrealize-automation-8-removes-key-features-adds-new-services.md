---
layout: post
title: "VRealize Automation 8 Removes Key Features, Adds New Services"
summary: "What's new in vRA 8?"
author: samui
date: "2020-03-03"
category: [vmware, vra, automation]
thumbnail: /images/2020/03/VRealize_Easy_Installer.jpg
permalink: /blog/vrealize-automation-8-removes-key-features-adds-new-services
published: true
---

## VRealize Automation 8 lacks approval policies and some XaaS functionality, but it introduced ABX scripts, Cloud Accounts and Cloud Assembly as part of its cloud optimization.

The following Article was written by Rob Bastiaansen. I found the original article [here.](https://searchvmware.techtarget.com/tip/VRealize-Automation-8-removes-key-features-adds-new-services){:target="_blank"} I wanted to share this, because I believe that Rob was spot on with his review of vRA 8 on-prem. I do believe that this information can benefit the community as a whole. As I continue to build out my homelab, I am sure that I will be discovering some of these changes within the product.

![Rob Bastiaansen](/images/2020/03/rob.jpg)
Rob Bastiaansen
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}

_Rob Bastiaansen is an independent trainer and consultant specializing in VMware and Linux. A VMware Certified Instructor based in the Netherlands, he delivers training around the world. He writes articles for several print and online publications and is founder of VMwarebits.com, a site dedicated to technical content related to VMware. Rob has been working with VMware technology since 2000 and has authored several books, including one of the first books dedicated to VMware entitled Rob's Guide To Using VMware. He has spoken at VMworld, which he has attended since 2004, and other conferences._

At the end of 2019, VMware released the new version 8 of vRealize Automation. Usually, a new version of a product offers more features than previous versions, but that's not the case with vRealize Automation 8. Instead, VMware released vRealize Automation 8 mainly as an on-premises version of its new product, vRealize Automation Cloud, with a new code base but fewer features.

Earlier in 2019, VMware introduced Cloud Automation Services, a cloud-based offering that provided the same functionality as vRealize Automation (vRA) in the cloud. VMware built that software from the ground up, rather than modifying vRealize Automation Services, although it offers similar services. VMware then renamed this product vRealize Automation Cloud.

And that software now has also been [released as the on-premises software stack](https://searchservervirtualization.techtarget.com/news/252460838/VMware-vRealize-suite-updates-focus-on-cloud-management){:target="_blank"} for vRealize Automation 8. This change to the codebase renders the old vRA appliance obsolete and is a primary reason that some of version 7's functionality isn't yet present in version 8.

### Feature backtrack with vRealize Automation 8

Although VMware likely intends to add features back in, it has yet to announce anything officially. Still, you can expect many features missing in 8.0 to return in version 8.1 or 8.2. VRA 8 lacks approval policies as well as endpoints for Hyper-V, kernel-based VM and vCloud Director. It also has limited functionality for anything as a service, with no custom resources and no XaaS blueprint components (software components).

VMware also has yet to release an upgrade path from version 7.x to 8.0. VMware will likely premiere upgrade functionality in a future version. Until then, VMware offers documentation about how to prepare for a future upgrade, and you can use the vRealize Automation 8 Migration Assessment Service to ease the process.

### Installing vRealize Automation 8

To install the new version of vRA, you can use a single ISO file containing the Easy Installer. This software installs Lifecycle Manager first and, based on your input, deploys VMware Identity Manager and vRA. This means that for all deployments you automatically have [an instance of Lifecycle Manager](https://searchvmware.techtarget.com/tip/Get-to-know-VMwares-vRealize-Suite-Lifecycle-Manager){:target="_blank"}.

![vRealize Easy Installer](/images/2020/03/VRealize_Easy_Installer.jpg)
vRealize Easy Installer
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}

A single ISO file contains vRealize Easy Installer, which installs Lifecycle Manager, VMware Identity Manager and vRealize Automation.

If you choose to, you can also use an existing Identity Manager instance or deploy vRA later from Lifecycle Manager. Lifecycle Manager, Identity Manager and vRA each run in their own appliances. However, vRA software now runs entirely in [Kubernetes pods](https://searchitoperations.techtarget.com/definition/Kubernetes-Pod){:target="_blank"}.

You don't have to set up the Kubernetes nodes yourself; Easy Installer creates them automatically. You can also access your entire deployment with the Kubernetes command set. You can deploy additional vRA nodes to form a cluster via Easy Installer, making it easy to create a [highly available deployment](https://searchitoperations.techtarget.com/tip/Ensure-Kubernetes-high-availability-with-master-node-planning){:target="_blank"}.

### New services and features

VRA 8 offers several new and updated services and features, despite the removal of certain familiar features. These include:

**Catalog.** From the perspective of an end user who already has access to a current version of vRealize Automation, the catalog appears much the same. It's part of Service Broker, the catalog management service that enables you to set up access to blueprints and [vRealize Orchestrator workflows](https://searchvmware.techtarget.com/feature/Picking-the-right-design-for-VMware-vRealize-Orchestrator){:target="_blank"}. It still includes custom forms management from version 7.4.


![Catalog](/images/2020/03/catalog.jpg)
Catalog
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}

The vRealize Automation catalog in version 8 closely resembles earlier versions of the same feature.

**Cloud Assembly.** Cloud Assembly helps you to create the setup for your environment and manage blueprints. Although you must use code to define what you want to achieve in Cloud Assembly, [the language it uses -- YAML](https://searchitoperations.techtarget.com/tip/Learn-YAML-through-a-personal-example){:target="_blank"} -- is simple.


![Cloud Assembly](/images/2020/03/cloud-assembly.jpg)
Cloud Assembly
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}

Using Cloud Assembly, you can define a vSphere machine with a configured network adaptor and modify VM properties with code.

Cloud Assembly features a [test-button](https://www.stevenbright.com/2019/11/getting-started-with-vrealize-automation-8-0-cloud-assembly/4/){:target="_blank"} that performs a check of your configurations to verify that you can deploy the blueprint with the given parameters. If you then want to deploy the blueprint, you can do so directly from the design interface -- no need to publish it and place it in the catalog first.

You can also now save versions of your blueprint design. These saved blueprints behave like snapshots in the sense that you can return to a previous version or to newer saved copies. You can also compare versions both graphically -- such as in the image -- or by comparing the YAML and looking for alterations in the code.

![Versioning](/images/2020/03/versioning.jpg)
Versioning
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}

Cloud Assembly enables you to compare blueprints with two side-by-side graphic images.

**Cloud Accounts.** Previous versions of vRA referred to Cloud Accounts as Endpoints. Blueprints can contain components from the various Cloud Accounts with which vRA can connect. Such Cloud Accounts include vCenter, VMware Cloud on AWS, AWS, Microsoft Azure and Google Cloud Platform. VRA can also connect with either [NSX-V or NSX-T](https://searchvmware.techtarget.com/feature/Learn-the-ins-and-outs-of-NSX-T-vs-NSX-V){:target="_blank"} in combination with vCenter instances.

Blueprints can also integrate with Kubernetes. For that, however, you must use another new vRA 8 service called Code Stream. Code Stream enables you to deploy container workloads in Kubernetes clusters, and you maintain full control over versioning and updating.

![Cloud Accounts](/images/2020/03/cloud-accounts.jpg)
Cloud Accounts
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}

Major Cloud Accounts include AWS, Microsoft Azure, Google Cloud Platform, both versions of NSX, VMware Cloud on AWS and vCenter.

**Event Broker.** [VRA 7 introduced Event Broker](https://searchvmware.techtarget.com/tip/How-vRealize-Automation-7-builds-on-previous-versions){:target="_blank"}, which remains in version 8. Although the number of event topics has been reduced, many of the necessary events remain for when you must trigger Orchestrator workflows.

![Event Topics](/images/2020/03/event-topics.jpg)
Event Topics
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}

Major event topics deal with blueprint configuration, compute management and deployment actions, among other events.

To further simplify your use of event subscriptions, you can now script your workflow conditions. Such workflow events can be as simple as checking the username for blueprint requests or as complex as triggering large constructions based on blueprint name or project name.

![Hostname](/images/2020/03/hostname.jpg)
Hostname
{:style="color:gray;font-style:italic;font-size:90%;text-align:center;"}

You can use a modified version of the default template with a Python script to rename a machine in the blueprint and process it as a custom property.

You can also now use both Orchestrator workflows and scripts that you enter directly into vRA, called Action Based eXecutions (ABX). You can write ABXes in Python or JavaScript, and they can manipulate a request at runtime. Formerly, you could only complete tasks such as modifying a computer's host name via an Orchestrator workflow. Now, you can complete such tasks with an ABX action.
