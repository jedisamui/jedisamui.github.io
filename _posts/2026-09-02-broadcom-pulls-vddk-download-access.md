---
title: "Broadcom Pulls Public Access to the VDDK: What It Is, Why It Mattered, and What Happens Now"
description: A breakdown of the VMware Virtual Disk Development Kit (VDDK) - what it does, how backup and migration tooling has relied on it for years - and what changed when Broadcom quietly restricted public access to it in late August 2026.
author: samui
date: 2026-09-02
categories:
- Virtualization
- Storage
tags:
- vddk
- vmware
- broadcom
- migration
- backup
- disaster-recovery
- storage
- api
image:
  path: /images/2026/09/cubicle404.png
permalink: /blog/broadcom-pulls-vddk-download-access/
---

If you've tried to grab a fresh copy of the VMware Virtual Disk Development Kit (VDDK) recently and hit a dead link, you're not imagining it. In late August 2026, Broadcom quietly pulled public access to the VDDK download pages - no advance notice, no deprecation warning, just a working URL one day and a 404 the next.

For a lot of infrastructure teams this barely registers. For the backup, disaster recovery, and migration ecosystem that has quietly depended on this one SDK for the better part of two decades, it's a genuinely big deal. Here's what the VDDK actually is, how it's been used, and what its restricted availability means going forward.

## What Is the VDDK, Actually?

The Virtual Disk Development Kit is a software development kit that VMware (and now Broadcom) has published for years, giving developers a supported way to read and write VMware virtual disk files (VMDKs) **from outside the guest operating system and outside the hypervisor's own management interfaces.**

In plain terms: normally, if you want to get data off a VM's disk, you either go in through the guest OS (installing an agent, mounting a filesystem) or you go through vCenter/ESXi management APIs to clone or snapshot the whole VM. The VDDK offers a third path - a library that lets a piece of software talk directly to the block-level contents of a virtual disk, without needing a live agent running inside the VM.

Functionally, the VDDK exposes:

- **Disk-level read/write access** to VMDK files, including running VMs via snapshots
- **Multiple transport modes**, most notably:
  - **NBD/NBDSSL** - network block device transport, streaming disk data over the network from the ESXi host
  - **HotAdd** - attaching the target disk directly to a proxy VM for local, high-throughput access
  - **SAN transport** - reading directly off shared storage when the backup/migration host has LUN access
- **Changed Block Tracking (CBT) support** - the mechanism that lets software ask "which blocks have changed since my last backup?" instead of re-reading an entire disk every time

That CBT piece in particular is what turns "backup a VM" from an hours-long full-disk copy into a fast incremental job. It's a big part of why the VDDK became foundational rather than optional.

## How It's Actually Been Used

The VDDK isn't customer-facing software you install and click around in - it's a library that other software links against. If you've ever run an image-level (agentless) backup job against a VMware VM, or used a tool to convert or move a VM's disks to another platform, there's a very good chance the VDDK was doing the heavy lifting underneath, even if you never saw its name.

Typical use cases include:

- **Agentless, image-level backup and recovery** - snapshotting a VM and streaming its disk blocks out to backup storage without touching the guest OS
- **Disaster recovery / replication tooling** - keeping an off-site or secondary copy of a VM's disks in sync using CBT-driven incrementals
- **P2V, V2V, and cross-platform migration and conversion tools** - reading a VMDK's contents in order to write it out in a format another hypervisor or cloud platform can consume
- **Data protection appliances and archival tools** that need to inspect or extract VM disk contents for compliance or e-discovery purposes

Because the VDDK is a redistributable-restricted library rather than an open standard, none of this software could simply "ship" it with their installers. Every vendor and open-source project either signed a redistribution agreement with VMware/Broadcom, or - far more commonly - documented a manual step telling the end user to go download the VDDK themselves from the public developer portal and drop it into a specific folder before the tool would work. That single external download step is what just broke.

## What Actually Changed

On or around August 25, 2026, Broadcom's public VDDK download pages began returning errors. Every version, every release line, gone from the standard public developer portal. No blog post, no release notes entry, no email to registered developers - it simply stopped resolving.

![Broadcom VDDK download page returning an error](/images/2026/09/vddk-download-404.png)
_The public VDDK download page, now returning an error instead of the SDK_

When affected users reached out, Broadcom Customer Care responded with the following:

> *"To ensure the highest standard of security, reliability, and product features, the Virtual Disk Development Kit (VDDK) is no longer available for use or download. Broadcom continues to actively maintain a variety of APIs and SDKs to enable authorized technology alliance partners to build backup and recovery software solutions…"*
>
> -Broadcom's official response provided by Broadcom Customer Care to their affected users

Reading between the lines, the model going forward is gated access: the VDDK is apparently still being maintained and distributed, but only to organizations that hold verified customer entitlements or official Technology Alliance Partner status. The self-serve, "anyone with a free developer account can grab it" model that the ecosystem has relied on since VDDK's early days appears to be over, at least for now.

![Diagram showing third-party backup/migration software linking against the VDDK library to read and write virtual disk blocks directly from vSphere storage](/images/2026/09/vddk-architecture.svg)
_Where the VDDK sits between third-party tooling and the underlying virtual disks_

## The Ramifications

**For established, partnered vendors:** relatively limited disruption. If your backup vendor already has a signed Technology Alliance Partner agreement with Broadcom, they likely have (or can get) an authorized distribution path for the library, and your existing product will probably keep working as before.

**For everyone else, the impact is much sharper:**

- **In-flight migrations get stuck mid-project.** Any migration effort that was relying on a fresh VDDK download - rather than a copy already cached locally - lost its path forward overnight, with no warning to plan around.
- **Unpartnered and open-source tooling is hit hardest.** Because the VDDK's license has never permitted general redistribution, no Linux distribution, container image, or open-source project has ever been able to bundle it directly. They've always depended on the end user fetching their own copy from the public portal. With that portal now gated, those projects lose their only legitimate distribution path for a dependency their entire ecosystem was built around.
- **It reinforces a well-known form of vendor lock-in.** The practical effect of restricting the tooling used to get data *out* of a virtualization platform is that leaving becomes harder, slower, and more dependent on the incumbent vendor's goodwill - regardless of whether that was the primary intent.
- **There is no clean workaround.** Because redistribution has always been restricted, nobody can simply mirror the files without taking on real legal exposure themselves. Alternative approaches exist - exporting full OVF/OVA packages instead of streaming disk blocks, or moving data through the storage layer rather than through the hypervisor API - but they generally trade away either speed, efficiency, or both compared to a VDDK-based path.

## Context: This Follows a Pattern of Change

The VDDK access restriction doesn't exist in a vacuum. It's the latest in a series of changes that have reshaped how VMware is licensed, packaged, and supported since Broadcom's acquisition of VMware closed in November 2023. Presented here purely as a timeline, without judgment on any individual decision:

- **Nov 2023** - Broadcom's acquisition of VMware officially closes.
- **Dec 2023** - Perpetual license sales end; VMware shifts to a subscription-only model.
- **Jan 2024** - The original partner program is retired in favor of a new, invite-only structure.
- **Feb 2024** - The free version of the hypervisor (previously offered at no cost) is discontinued.
- **2024** - Licensing shifts from a metered-capacity model toward per-core subscription bundles, with product editions consolidated into larger suites.
- **2025** - Partner program tiers are further consolidated and restructured, with some existing partners transitioning out and others invited to remain.
- **Aug 2026** - Public download access to the VDDK is restricted, moving to a gated, entitlement-based model.

Whatever one's view of the individual decisions, the throughline is consistent: access to the platform - licensing, partnership, and now some of the underlying tooling - has been steadily moved behind more formal, verified relationships with Broadcom.

## Where This Leaves You

If your organization depends on VMware backup, DR, or migration tooling - whether commercial or open-source - a few practical things are worth doing now rather than later:

1. **Check whether you already have a working local copy of the VDDK.** If a colleague, build server, or old project directory still has a cached copy, treat it as an asset worth preserving under change control, along with its version number and published checksum.
2. **Confirm your entitlement status directly with Broadcom** if you have an active vSphere support agreement - the gated model reportedly still allows verified customers through, just not via the old public portal.
3. **Test your fallback path now, not during an actual migration or DR event.** If your tooling supports a non-VDDK route (such as a full-image export), exercise it once deliberately so it's a known quantity rather than a surprise under pressure.
4. **If you're evaluating a move off VMware entirely**, factor this into your timeline. Whatever tooling you choose - whether it targets an alternative hypervisor platform, a cloud provider, or a container-based re-platform - ask specifically how it sources virtual disk data today, and what its fallback looks like if that source becomes harder to reach.

The VDDK itself hasn't been declared dead, and Broadcom's statement suggests it's still very much alive for partners working through official channels. But for the long tail of smaller vendors, open-source maintainers, and IT teams who relied on a public, no-questions-asked download link, that link is gone - and there's no indication of when, or whether, it's coming back in its old form.
