---
date:
  created: 2021-12-07
  updated: 2021-12-07
authors:
    - aninchat
blog_toc: true
categories:
  - cumulus
tags:
  - cumulus
---
# Cumulus Part II - Bridging

This second blog on Cumulus looks at basic layer2 functionality in Cumulus Linux.

This post is going to introduce you to basic Layer2 functionality on the Cumulus platform. Like before, we are going to be working with Cumulus VX. 

## Topology

We will be using the following network topology for this post:

![bridging1](/static/images/cumulus/cumulus_part2/cumulus_bridging_1.jpg)

PC1 and PC2 are two end clients in the same Layer2 domain (VLAN 10) that want to communicate with each other.

<!-- more -->

## Bridges in Cumulus Linux

Cumulus uses the traditional concept of a Linux bridge for bridging. It also allows you to configure a VLAN-aware bridge which is what we will be using today (more documentation about this can be found [here](https://docs.cumulusnetworks.com/display/DOCS/VLAN-aware+Bridge+Mode). 


A 'bridge' interface will be created when the first VLAN is created on the system. So, let's start by creating a VLAN using Cumulus' NCLU and looking at the relevant changes that will be made to the /etc/network/interfaces file:

```
// create a VLAN

cumulus@SPINE1:~$ net add vlan 10 

// view changes that will be made and to what file using 'net pending'

cumulus@SPINE1:~$ net pending
--- /etc/network/interfaces     2018-11-13 18:46:52.000000000 +0000
+++ /run/nclu/ifupdown2/interfaces.tmp  2019-03-25 15:05:05.712745615 +0000
@@ -3,10 +3,15 @@
 
 source /etc/network/interfaces.d/*.intf
 
 # The loopback network interface
 auto lo
 iface lo inet loopback
 
 # The primary network interface
 auto eth0
 iface eth0 inet dhcp
+
+auto bridge
+iface bridge
+    bridge-vids 10
+    bridge-vlan-aware yes



net add/del commands since the last "net commit"
================================================

User     Timestamp                   Command
-------  --------------------------  ---------------
cumulus  2019-03-25 15:04:49.012290  net add vlan 10
```

From the above, you can see that a 'bridge' interface is created by adding 'iface bridge' and under this, you specify what VLANs are part of this bridge using 'bridge-vids <>'. The 'bridge-vlan-aware yes' option under the bridge makes this a VLAN-aware bridge as opposed to a traditional Linux bridge. 


Do the same thing on SPINE2 as well. 


Now, let's start mapping these VLANs to our interfaces. We want swp3 on SPINE1 and swp4 on SPINE2 to be untagged, host facing interfaces in VLAN 10 while the interconnection between SPINE1 and SPINE2 (via swp1) needs to allow multiple VLANs, making it a trunk. 

```
// map bridge VLANs to interfaces

cumulus@SPINE1:~$ net add interface swp3 bridge access 10 
cumulus@SPINE1:~$ net add interface swp1 bridge trunk vlan 10-20
 
// view changes that will be made and to what file using 'net pending'
 
cumulus@SPINE1:~$ net pending
--- /etc/network/interfaces     2018-11-13 18:46:52.000000000 +0000
+++ /run/nclu/ifupdown2/interfaces.tmp  2019-03-25 15:08:04.817750296 +0000
@@ -3,10 +3,24 @@
 
 source /etc/network/interfaces.d/*.intf
 
 # The loopback network interface
 auto lo
 iface lo inet loopback
 
 # The primary network interface
 auto eth0
 iface eth0 inet dhcp
+
+auto swp1
+iface swp1
+    bridge-vids 10-20
+
+auto swp3
+iface swp3
+    bridge-access 10
+
+auto bridge
+iface bridge
+    bridge-ports swp1 swp3
+    bridge-vids 10-20
+    bridge-vlan-aware yes



net add/del commands since the last "net commit"
================================================

User     Timestamp                   Command
-------  --------------------------  -----------------------------------------------
cumulus  2019-03-25 15:04:49.012290  net add vlan 10
cumulus  2019-03-25 15:07:10.463736  net add interface swp3 bridge access 10
cumulus  2019-03-25 15:08:01.690646  net add interface swp1 bridge trunk vlans 10-20
```

Notice how under the bridge interface, there is an interface mapping that is created using 'bridge-ports'. This tells the bridge what interfaces are part of it.


Similar configuration needs to be done on SPINE2. Finally, we need to do a 'net commit' to commit these changes. 


At this point, PC1 can successfully ping PC2:

```
PC-1> ping 10.1.1.2
84 bytes from 10.1.1.2 icmp_seq=1 ttl=64 time=3.108 ms
84 bytes from 10.1.1.2 icmp_seq=2 ttl=64 time=2.412 ms
84 bytes from 10.1.1.2 icmp_seq=3 ttl=64 time=1.585 ms

*snip*
```

## Final configuration 

Let's take a final look at the configuration of both SPINE1 and SPINE2 that was needed to make this work:

```
SPINE1:

cumulus@SPINE1:~$ cat /etc/network/interfaces
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
    bridge-vids 10-20

auto swp3
iface swp3
    bridge-access 10

auto bridge
iface bridge
    bridge-ports swp1 swp3
    bridge-vids 10-20
    bridge-vlan-aware yes

SPINE2:

cumulus@SPINE2:~$ cat /etc/network/interfaces
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
    bridge-vids 10-20

auto swp4
iface swp4
    bridge-access 10

auto bridge
iface bridge
    bridge-ports swp1 swp4
    bridge-vids 10-20
    bridge-vlan-aware yes
```

I hope this was informative. See y'all in the next post!