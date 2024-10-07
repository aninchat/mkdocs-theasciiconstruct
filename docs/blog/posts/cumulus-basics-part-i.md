---
date:
  created: 2021-12-05
  updated: 2021-12-05
authors:
    - aninchat
blog_toc: true
categories:
  - cumulus
tags:
  - cumulus
---

# Cumulus Basics Part I - navigating the OS
This first blog on Cumulus introduces the reader to the basics of the operating system and Cumulus' NCLU.

## Introduction

This is going to be a new mini series that we will do in preparation for the first open networking certification that Cumulus Networks introduced,see [here](https://education.cumulusnetworks.com/certification-exam-registration). 

Cumulus Networks has a great page (both free and paid content) where you can spend some time and learn all things Cumulus and open networking related, see [here](https://education.cumulusnetworks.com/).

So, where do we begin? Every time I learn a new product, I start at the start. Things we learned way back when. Because Cumulus Linux is a native Linux distribution (and it's interface may be unfamiliar to many), we'll start with some very simple aspects of working with the box - basic port bring up/down, port configurations, gathering information about a port and finally an introduction to Cumulus' NCLU! 

## Topology

As a reference, we'll be working on the following topology:

![basic1](/static/images/cumulus/cumulus_part1/cumulus_basic_1.jpg)

<!-- more -->

## Basic interface configuration

With Linux networking, your interface configurations would be found in /etc/network/interfaces (there are several helpful pages that you can google and find to understand the syntax in this file so we're not going to go over that in too much detail). You can quickly view this via the 'cat' option.

On both boxes, with a blank configuration, we see:

```
cumulus@cumulus:~$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*.intf

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp
```

This file can directly be modified to change the configuration of various interface or to introduce new logical interfaces. To demonstrate this, let's go ahead and configure swp1 on each box to be a L3 interface with an IP address in the subnet 10.0.0.0/24. 

SW1:

```
cumulus@cumulus:/$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*.intf

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp

auto swp1
iface swp1
    address 10.0.0.1/24
```

SW2:

```
cumulus@cumulus:/$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*.intf

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp

auto swp1
iface swp1
    address 10.0.0.2/24
```

Bring these interfaces up using the 'ifup' option of the ifupdown2 module on both the switches.

```
cumulus@cumulus:/$ sudo ifup swp1
```

To confirm the status of the interface, you can use 'ip link show' to list all interfaces or specify a particular interface to look at using 'ip link show swp1'.

```
cumulus@cumulus:~$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 0c:db:6b:cc:20:00 brd ff:ff:ff:ff:ff:ff
3: swp1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 0c:db:6b:cc:20:01 brd ff:ff:ff:ff:ff:ff
4: swp2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 0c:db:6b:cc:20:02 brd ff:ff:ff:ff:ff:ff
5: swp3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 0c:db:6b:cc:20:03 brd ff:ff:ff:ff:ff:ff
6: swp4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 0c:db:6b:cc:20:04 brd ff:ff:ff:ff:ff:ff
7: swp5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 0c:db:6b:cc:20:05 brd ff:ff:ff:ff:ff:ff
8: swp6: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 0c:db:6b:cc:20:06 brd ff:ff:ff:ff:ff:ff
```

Looking at only swp1:

```
cumulus@cumulus:~$ ip link show swp1
3: swp1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 0c:db:6b:cc:20:01 brd ff:ff:ff:ff:ff:ff
```

An interesting thing to note here - even if you shut down one side of the link (say, do a 'sudo ifdown swp1' on SPINE1), the other side would still show that the link is up. This behavior is specific to Cumulus VX in virtualized environments only and was confirmed by Cumulus folks (this is not just a Cumulus issue - this behavior is typical of virtual routers/switches and is seen across vendors).


"When using VBox there is a little switch in the middle of the link that holds it up." 


You can poke this middle switch to bring the links down if you're specifically testing link failures. But we're not going to get into that here. 


Clearly, manipulating these network configuration files can be a little tiresome and more importantly, prone to human error. You need to be aware of the kind of syntax that is used within these files and all of the different intricacies that go into the configuration here.

## Cumulus NCLU introduction

This is where Cumulus' NCLU comes in. NCLU (Network Command Line Utility) is essentially a CLI that takes you away from manual manipulation of network files and provides a helpful CLI for the same instead. 


All NCLU commands start with 'net'. You can tab or use a question mark to get all the available options. 

```
cumulus@SPINE1:~$ net 
    abort     :  abandon changes in the commit buffer
    add       :  add/modify configuration
    clear     :  clear counters, BGP neighbors, etc
    commit    :  apply the commit buffer to the system
    del       :  remove configuration
    example   :  detailed examples of common workflows
    help      :  context sensitive information; see section below
    pending   :  show changes staged in the commit buffer
    rollback  :  revert to a previous configuration state
    show      :  show command output
```

To add configuration to an interface, you can use the 'net add interface' CLI. For example, instead of manipulating the /etc/network/interfaces file directly to add an IP address to swp1, I can do this instead:

```
cumulus@cumulus:~$ net add interface swp1 ip address 10.0.0.1/24
```

NCLU has three steps to it:


1. Configure using 'net add [del]' commands.
2. Confirm what is going to be configured using 'net pending'.
3. Push this configuration using 'net commit'.


A complete example of configuring an IP address on swp1 follows:

```
// net add

cumulus@SPINE1:/$ net add interface swp1 ip address 10.0.0.1/24

// net pending

cumulus@SPINE1:~$ net pending
--- /etc/network/interfaces     2019-03-05 16:40:58.268594116 +0000
+++ /run/nclu/ifupdown2/interfaces.tmp  2019-03-05 17:23:54.585798862 +0000
@@ -6,14 +6,12 @@
 # The loopback network interface
 auto lo
 iface lo inet loopback
 
 # The primary network interface
 auto eth0
 iface eth0 inet dhcp
 
 auto swp1
 iface swp1
-       address 10.1.1.1/24
-
-
-
+    address 10.0.0.1/24
+    address 10.1.1.1/24



net add/del commands since the last "net commit"
================================================

User     Timestamp                   Command
-------  --------------------------  ---------------------------------------------
cumulus  2019-03-05 17:23:51.321383  net add interface swp1 ip address 10.0.0.1/24

// net commit

cumulus@SPINE1:~$ net commit
--- /etc/network/interfaces     2019-03-05 16:40:58.268594116 +0000
+++ /run/nclu/ifupdown2/interfaces.tmp  2019-03-05 17:24:00.025799294 +0000
@@ -6,14 +6,12 @@
 # The loopback network interface
 auto lo
 iface lo inet loopback
 
 # The primary network interface
 auto eth0
 iface eth0 inet dhcp
 
 auto swp1
 iface swp1
-       address 10.1.1.1/24
-
-
-
+    address 10.0.0.1/24
+    address 10.1.1.1/24



net add/del commands since the last "net commit"
================================================

User     Timestamp                   Command
-------  --------------------------  ---------------------------------------------
cumulus  2019-03-05 17:23:51.321383  net add interface swp1 ip address 10.0.0.1/24
```

!!! note
    From a link logging perspective, this is done via rsyslog. The defaults can be found in '/etc/rsyslog.d'. 22-linkstate.conf contains information on where link state changes are logged. By default, this goes to /var/log/linkstate. However, with Cumulus VX, you'd soon realize that there is no 'linkstate' file in /var/log. This is because switchd is responsible for this and switchd doesn't do much in Cumulus VX (as it is largely just an interface for the ASIC). Due to this, you cannot track link state changes in Cumulus VX. Shoutout to [Eric Pulvino](https://twitter.com/EricPulvino) for clarifying this.  


We will continue with bridging on Cumulus VX in part II. Each post in this series will show both NCLU configurations and manual changes needed to the relevant network files.
