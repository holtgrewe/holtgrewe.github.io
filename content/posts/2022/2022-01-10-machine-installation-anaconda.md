---
title: "HPC for Bioinformatics (2/?): Machine Installation"
date: 2022-01-10
categories:
  - HPC for Bioinformatics
tags:
  - hpc
author: Manuel Holtgrewe
draft: true
---

This post describes the two basic options for system installation that we have tried so far.
The wonderful people at [StackHPC](https://www.stackhpc.com) have a [blog post on "zero touch provisioning"](https://www.stackhpc.com/ironic-idrac-ztp.html) that has more technical depth than mine.
This is probably what one aim for but we are not there yet at the moment of writing.
Rather, this post gives some perspective on how to manage the installation 200 computers with "few touches", albeit not zero.

First, let us define the problem taht we need to solve in more detail: installing the operating system.
"What is hard and why is this necessary?" you might ask.
You probably bought your private laptop with an operating system preinstalled and your work place/school might provide you with computers with preinstalled operating systems.

How does the vendor of your laptop install the operating system?
Most probably, they install it once on one machine and configure it such that you can make the inital setup like creating a user on the first start.
Then they copy the whole disk as one **disk image** to a server and copy this disk image to hundreds of disks that they then use for assemling your laptop.
This approach can be very useful if you need to manage a hand full of laptops in a school, for example, that are given out for courses.
If all laptops are the same and you make a copy of the initial image, e.g., with [Clonezilla](https://clonezilla.org/) then you can restore the laptops within a few minutes with very few clicks.
You boot the laptop from a USB stick into Clonezilla, copy the disk image from an external hard drive and the laptop will come up with exactly the same configuration as you created the image.
Pretty neat, eh?

## Introducing PXE

But clearly this does not scale up to two hundred servers.
Also, each server needs to be setup with its own IP address and host name.
For this, the [preboot execution environment](https://en.wikipedia.org/wiki/Preboot_Execution_Environment) (PXE, pronounced as "pixie") has been developed in the 1990s.

{{< figure src="/posts/2022/2022-01-10-pxe-boot-wikipedia.png" width="50%" caption="PXE boot according to Wikipedia [source](https://en.wikipedia.org/wiki/File:PXE_diagram.png)" >}}

So what exactly is PXE?
PXE is an open and vendor-independent standard for a client-server environment for a client-server boot environment.
Shortly, the client computer boots such that the server can control what action the client does next.
This action can be the installation of an operating system with a particular configuration.

But how exactly does this work?
When booting in PXE mode, the client uses [DHCP (dynamic host configuration protocol)](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) to obtain its initial network configuration and other information.
The network configuration includes its IP address and addresses of DNS servers and the ntwork standard gateway.
The other information includes the address of a (minimal) system image to boot and configuration to load.
The server boots into this system image and loads the configuration.
This configuration provides the information what to do next.

In the case of Red Hat based operating systems such as CentOS and Rocky Linux, [Anaconda](https://en.wikipedia.org/wiki/Anaconda_(installer)) is started which installs the operating system from the network and applies certain configuration, including the host name.
The server runs through an automated operating system installation that uses the same installer that you would use for manually installing Rocky Linux, just with all options prefilled.
After the installation the machine reboots from the disks and you have a blank installation of your server that can be configured further.
When configured appropriately, the server could run a script on the first startup to perform additional setup steps with automation tools such as Ansible or Puppet.

## Home-Build PXE

Our first iteration of cluster setup used a manual setup for PXE boot.
You have to get everything "just right" but on the other hand you can understand and control each component.
You will need

- a DHCP server, we used `dnsmasq`,
- a TFTP server, we used the one built into `dnsmasq`,
- proper minimal images for the PXE boot and configuration file served over TFTP (you can find a lot of tutorials for this online),
- a HTTP server that serves the packages to be used during the installation.

When everything is setup properly, your server will boot into the anaconda installer, installation will run and it will come up with a fresh, say, CentOS installation.
Of course, you should distribute SSH keys to your server so you can login once the server boots.

## PXE with xCAT

We used xCAT for the same installation.
xCAT can be used as a "middleware" where you configure the compute servers in a so-called `stanza` format that you pass to commands calls `mkdef`, for example.
The stanza format looks like this:

{{< highlight text >}}
compute01:
    objtype=node
    arch=x86_64
    mgt=ipmi
    cons=ipmi
    bmc=10.1.0.12
    nictypes.etn0=ethernet
    nicips.eth0=11.10.1.3
    nichostnamesuffixes.eth0=-eth0
    nicnetworks.eth0=clstrnet1
{{< / highlight >}}

You can clearly see that xCAT is from the early 2000s where JSON and YAML where not a thing yet.
Also, the definition is limited to two levels and for specifying deeper nesting, you need interesting syntax.
E.g., for setting two IPs, you would use `nicips.eth0=11.10.1.3|12.10.1.3`.
xCAT will then generate the appropriate `dhcpd` configuration and manually allocate IPs in the `dhcpd` leases file (which occasionally broke in our hands and we had to manually clear the file, restart `dhcpd` and generate the configuration fresh).

## The Ironic Way

Ironc is an OpenStack project that allows you to manage bare metal hardware.
One main advantage is that once everything is setup properly, you can schedule bare metal servers using the same way that OpenStack uses for running virtual machines (the component for managing compute in OpenStack is called *nova*).
This is available to you in the Ansible collection [openstack.cloud](https://docs.openstack.org/openstack-ansible/latest/) and you only specify a different "machine flavor" to get a bare metal machine instead of a virtual one.
This makes it very easy to have a test/staging environment, make everything work there and then roll it out on bare metal.
Further plus points is the comprehensive documentation (in particular when compared to xCAT), the ironic community is much larger than xCAT (which appears to be mainly a developer at IBM), and the ironic people are extremely nice and friendly on IRC.
What's not to love?

- ironic python agent
- copy image
