---
title: "Building a Raspberry Pi High Availability cluster, part 1"
---


In this blog post series, we will explore the Raspberry Pi platform and
attempt to get the
[Pacemaker](https://clusterlabs.org/pacemaker/)
High Availability cluster stack running on it. The operating system of
choice will be Fedora, in order to make it possible for readers to easily
replicate the setup.

This is part 1 of the series, where we discuss the motivation behing this
project, the background and required hardware, complete with the whys and
hows.


Motivation
==========

Desire to have a cheap, small and portable high availability cluster setup
for demos, with which people could physically interact and examine its
behavior in an intuitive, easily understandable and observable form.


Background
==========

To understand why certain architectural decisions for this build were made,
it is required to have at least a basic knowledge of
[High Availability (HA)](https://en.wikipedia.org/wiki/High_availability)
clustering concepts.

Essentially, the goal of HA clusters is to eliminate
[single points of failure](https://en.wikipedia.org/wiki/Single_point_of_failure)
through
[redundancy](https://en.wikipedia.org/wiki/Redundancy_%28engineering%29),
while automatically detecting and attempting
to remedy failures as they occur (through a healthy mix of
[fault tolerance](https://en.wikipedia.org/wiki/Fault-tolerant_computer_system),
[resilience](https://en.wikipedia.org/wiki/Resilience_%28network%29),
[failover](https://en.wikipedia.org/wiki/Failover)
and
[self-healing](https://en.wikipedia.org/wiki/Self-management_%28computer_science%29)
properties).

In practice, this is achieved by:
  * having multiple cluster nodes (2+)
  * giving cluster nodes the ability to cut off access to shared resources
    (such as storage) for misbehaving nodes in order to prevent them from
    doing damage (for example corrupting a shared file system) -- this is
    usually referred to as
    [fencing](https://en.wikipedia.org/wiki/Fencing_%28computing%29)
    and generally is implemented as either:
    * [Shoot The Other Node In The Head (STONITH)](https://en.wikipedia.org/wiki/STONITH),
      when the power is turned off for nodes to be fenced (this is the most reliable
      option), or
    * a method specific to the shared resource(s) used, such as
      [SCSI persistent reservations](https://www.kernel.org/doc/Documentation/block/pr.txt),
      referred to as I/O fencing
  * [multipathed](https://en.wikipedia.org/wiki/Multipath_I/O)
    deployment of
    [Storage Area Network (SAN)](https://en.wikipedia.org/wiki/Storage_area_network)
    to provide shared storage to the cluster, to ensure failure of a single
    network link does not render the SAN inaccessible
  * similarly, deploying multiple switches and other network equipment, so
    that a failure of any network component does not result in an outage
  * employing
    [Uninterruptible Power Supplies](https://en.wikipedia.org/wiki/Uninterruptible_power_supply)
    to keep the electrons moving


Hardware
========


Cluster nodes
-------------

The
[Raspberry Pi 3 Model B+](https://www.raspberrypi.org/products/raspberry-pi-3-model-b-plus/)
was chosen as the ideal
[single-board computer](https://en.wikipedia.org/wiki/Single-board_computer)
for this build. This was mostly due to the fact that it supports an optional
[Power-over-Ethernet (PoE)](https://en.wikipedia.org/wiki/Power_over_Ethernet)
add-on board, the
[PoE HAT](https://www.raspberrypi.org/products/poe-hat/),
which we will use to eliminate the need for separate Micro-USB power supplies.

As a side note, HAT stands for
[Hardware Attached on Top](https://www.raspberrypi.org/blog/introducing-raspberry-pi-hats/)
and is a Raspberry
Pi specific lingo for add-on boards. Other vendors have similarly bizarre
names for them, for example
[BeagleBoard](https://beagleboard.org/)
refers to them as
[capes](https://beagleboard.org/capes).

In order to greatly reduce both cost and complexity (and keep the whole setup
portable), the network and shared storage redundancy will be out of scope.
We also decided not to use any
[UPS HATs](http://www.raspberrypiwiki.com/index.php/Raspi_UPS_HAT_Board)
either, mostly due to safety
concerns (the PoE HAT tends to get very hot and could damage or quickly
degrade the Li-ion cells).


Network
-------

Next up is the fencing mechanism. Since we already employ PoE to provide power
to the cluster nodes, STONITH is the obvious choice here. For this to work,
however, we will require a PoE-capable network switch.

Plenty of cheap "smart" "managed" switches are available on the market. None
of those seem support programmatic access (via
[Simple Network Management Protocol (SNMP)](https://en.wikipedia.org/wiki/Simple_Network_Management_Protocol),
for example) to turn power on and off for individual Ethernet ports.

The only product supporting SNMP
[Interfaces Management Information Base (IF-MIB)](https://en.wikipedia.org/wiki/Management_information_base)
small enough to be viable for this build is the
[Mikrotik hEX PoE](https://mikrotik.com/product/RB960PGS).

SNMP IF-MIB is a solid choice for our power fencing solution, as it is already
implemented as the
[fence_ifmib](https://github.com/ClusterLabs/fence-agents/tree/master/agents/ifmib)
I/O fence agent and widely used. Notice that
since we power our nodes via PoE and disabling an ethernet port on our switch
also cuts the power to that port, we have effectively turned an I/O fencing
agent into a power fencing one. Neat, isn't it?


Storage
-------

We will not be using any extra hardware to provide a SAN for the cluster to
use as shared storage. Instead, an
[iSCSI](https://en.wikipedia.org/wiki/ISCSI)
target will be hosted on an extra
Pi. This machine will not be part of the cluster, it's reliability and
redundancy is out of scope for the same reasons mentioned earlier.


Human interfaces
----------------

As we want this demo cluster to be human-interactive, some kind of input and
output devices are needed to affect and monitor the state of the cluster.

Ethernet cables can be considered an input device in our case, since manually
unplugging one of the cluster nodes from network will immediately power off
that node and the rest of the cluster should notice this and proceed to act
upon it (recovering any services that were running on the killed node, for
one).

Physical buttons could be added to the individual cluster nodes, connected to
their
[General-Purpose Input/Output (GPIO)](https://en.wikipedia.org/wiki/General-purpose_input/output)
pins and configured in the OS to trigger a
[kernel panic](https://en.wikipedia.org/wiki/Kernel_panic).
Panicking a node will effectively kill it, without having to cut the power.
We could then observe the cluster as it fences the node, turning its power
off and on again -- to reboot it in hope that it will come back and re-join
the cluster.

As for the output devices, screens would be the obvious choice. We decided to
go with a small [E-Ink] display for each Pi, specifically a
[tri-color (red/black/white) 2.13" Waveshare e-Paper HAT](https://www.waveshare.com/wiki/2.13inch_e-Paper_HAT_%28B%29).

To spice things up a bit (and showcase more Pacemaker features :)), we also
purchased a small thermal receipt printer, the
[PTP-II](https://www.cashinotech.com/58mm-portable-mobile-bluetooth-thermal-printer-ptp-ii_p8.html).
The plan is to leverage
[Pacemaker alerts](https://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch07.html)
and have a log of all actions taken by the cluster, as a hard copy on a piece
of paper.


Stay tuned
==========

This concludes the first part of this blog post series on building an
inexpensive and small high availability cluster with the end goal of assisting
in explaining the general concepts of HA clustering.

Stay tuned for part 2, in which we will show how the hardware physically fits
together, how to install Fedora Linux OS on the Raspberry Pi 3 Model B+ and
the first bumps in the road to having the perfect setup.


Full disclosure
===============

At time of writing the author works as a Quality Engineer at Red Hat, making
sure the High Availability and Replicated Storage add-ons for Red Hat
Enterprise Linux work as advertised.

Any and all opinions are personal.
