---
title: "HPC for Bioinformatics (1/?): Introduction"
date: 2022-01-03
categories:
  - HPC for Bioinformatics
tags:
  - hpc
author: Manuel Holtgrewe
---

This is the first entry in the *HPC for Bioinformatics* series.
In this series, I will outline my experience with building and operating an HPC system for bioinformatics.
I will start by outlining the overall history and then plan to discuss more or less random topics.
The selection will be mostly based on what I find interesting at the time of writing and I will try not to rehash things that have been discussed a lot of times in other places.

## The Origins

In ca. 2017 the Core Unit Bioinformatics (CUBI) at Berlin Institute of Health took over the organisation and administration of the HPC system.
Roughly, the hardware setup was as follows:

- ca. 200 Dell PowerEdge M600 series nodes (16 of these go into one M1000e) enclosure,
- a Dell R630 server with two VT100 GPUs,
- 4 Dell R630 high memory nodes (2x512GB and 2x1TB RAM),
- 2PB of visible storage appliance by DDN based on IBM GPFS (now called SpectrumScale) as "tier 1" storage,
- 10/40GbE interconnect (switches are redundant VLT pairs) with Dell switches,
- about 500TB of visible ZFS storage as NFS shares as "tier 2 storage", in a two 60 disk Dell PowerVault enclosures,
- a couple of miscellaneous servers for utility services.

The user-visible software side of things were:

- CentOS 7 operating system,
- GridEngine scheduler (the open source Son of a Grid Engine variation).

From the perspective of `root` important:

- system configuration was based on Puppet,
- bare metal machines were setup using a home cooked PXE boot environment,
- utility machines ran as KVM virtual machines using libvirt,
- we could use the institute's tape library for backups/archive purposes,
- there was one out of band management network in addition to one flat compute network.

## The Current State (Q1 2022)

The current, modern state of the system is as follows:

- ca. 150 Dell PowerEdge M600 series nodes (because of space issues in the racks, the others now serve as cold spares),
- the same high memory nodes,
- 36 Dell PowerEdge C6420 servers with 25GbE network,
- 7 Dell PowerEdge C4130 servers with 4 NVIDIA Tesla V100 GPUs,
- 10/40GbE interconnect for the older services, 25/100GbE interconnect for the more recent servers,
- the same 2PB of DDN/GPFS storage (sunsetting by the end of 2022),
- two "tier 2" Ceph clusters (HDDs) with a raw capacity of 3.5PB each (one primary and one mirror system),
- 8 servers as OpenStack compute hosts,
- a "tier 1" Ceph all-NVME cluster with a raw capacity of 1.5PB (setup is in progress).

From the user side:

- Rocky Linux 8 operating system,
- Slurm scheduler,
- Open OnDemand Web Portal.

From the `root` perspective:

- system configuration based on Ansible,
- streamlined host setup:
  - deployment of OpenStack hosts and Ceph servers with [Kayobe](https://docs.openstack.org/kayobe/latest/),
  - deployment of baremetal compute hosts with [OpenStack Ironic](https://ironicbaremetal.org/),
- backup and archive go to our Tier 2 Ceph clusters,
- relatively flat network structure with out-of-band, private compute, and public network.

{{< figure src="/posts/2022/2022-01-03-infrastructure-openstack.jpg" width="50%" caption="OpenStack has a module for most of your infrastructure needs!" >}}

## In Between: Proxmox VE

We ran [Proxmox VE (PVE)](https://www.proxmox.com/en/proxmox-ve) for some years for management of virtual machines.
PVE is a good solution for running compute servers based on virtual machines and Linux containers (LXC) and Ceph can be used for storage for several versions.
We have used it both with ZFS based storage and Ceph based storage.
Using ZFS storage with LXC compute provides excellent performance for database based applications such as our [VarFish](https://cubi.bihealth.org/software/varfish/).
Using Ceph has the advantage of distributed storage and virtual machines can be migrated between hosts more easily.

PVE clearly is no "cloud" solution but a virtualization environment.
This is not bad in itself and gives a stronger feeling of control as one can (and must) specify what is run where.
However, managing 200+ bare metal nodes is not within its capabilities.
A very strong point is the easy-to-use user interface.

We had some issues over the time with the python client library that we used from Ansible for creating VMs and containers in Proxmox.
An upgrade of PVE would lead to the library not working until the library itself had been patched (or the Ansible module, I cannot remember for certain).

## In Between: xCAT

We have been using home-cooked PXE environment for several years, based on dnasmasq/tftp and anaconda for system installation.
However, managing the PXE directories was tedious, in particular for system upgrade.
We switched to using [xCAT](https://xcat.org/) ca. 2020.

xCAT (extreme cluster/cloud administration toolkit) is an open source software for managing bare metal infrastructure originally developed by IBM.
This software is used for the deployment of many large HPC systems on the [TOP500](https://www.top500.org/) list.
Our experience with xCAT was generally OK and a great step forward from a self-built PXE solution.

My personal main impression is that xCAT is a relatively opaque system based on a large number of bash/perl scripts.
It is open source, so you can go into the sources and find out where the problems are but a lot of things cannot be configured.
For example, we had an issue with switch network ports not coming up properly (in a cluster not mentioned above that is using arriva switches).
The way to resolve this involved adding `set -x` in a bash script in `/install/...` on the xcat server, looking at the logs and adding a `sleep` in the xcat source code **in place**.
The mixture of shell and awk scripts that I saw in the meantime was ... interesting and is probably best described as "enterprise grade".
To be fair, xCAT did its job very well and my main problem was that I could not control it from Ansible and thus automation was problematic.

## To Summarize...

... I now realize that this is an extremely boring blog entry that is just listing things (at least I did not make any major detour).
What can be learned from this:

- We have now migrated to OpenStack for managing the compute infrastructure.
  This gives us a *single pane of glass* for managing our servers in a modern, cloud-based fashion.
- OpenStack [Kayobe](https://docs.openstack.org/kayobe/latest/) allows to setup the openstack cluster and other infrastructure (aka "undercloud").
- We have migrated to a modern OpenStack cluster scheduler - Slurm.
- We will abandon the commercial storage system and rather than spend money on licenses (and, granted, support) spend it on hardware.
