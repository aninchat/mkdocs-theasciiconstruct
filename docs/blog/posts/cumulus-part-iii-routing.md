---
date:
  created: 2021-12-08
  updated: 2021-12-08
authors:
    - aninchat
blog_toc: true
categories:
  - cumulus
tags:
  - cumulus
---
# Cumulus Basics Part III - static routing and OSPF

In this post, we will look at an introduction to routing on Cumulus Linux, with static routing and OSPF.

## Introduction

Initially, Cumulus OS used the Quagga suite for routing capability. However, more recently, there has been a general adoption of a fork of Quagga called FRRouting (FRR) - Cumulus now includes FRR instead of Quagga. Like always, you can either edit the files directly or using Cumulus' NCLU to enable the respective routing features as well. 

## Topology

We'll be using the following network topology for this post:

![static1](/static/images/cumulus/cumulus_part3/cumulus_static_1.jpg)

<!-- more -->

Routing configuration is stored in /etc/frr/frr.conf (like how interface configuration is stored in /etc/network/interfaces). It is important to note that the protocols under FRR run as deamons on the OS and are not enabled by default. To see the list of daemons and their status, you can view the /etc/frr/daemons file:

```
cumulus@SW1:~$ cat /etc/frr/daemons
zebra=yes
bgpd=no
ospfd=no
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
```

'zebra' is the IP routing manager and controls things like static routes. It provides kernel routing table updates, interface lookups, and redistribution of routes between different routing protocols (quoted from 'http://docs.frrouting.org/en/latest/zebra.html').


Let's configure some static routes to provide connectivity between PC1 and PC2. The option to add static routes comes under 'net add routing':

```
cumulus@SW1:~$ net add routing 
    agentx                :  Enable SNMP support for OSPF, OSPFV3, and BGP4 MIBS
    as-path               :  AS_PATH attribute
    community-list        :  Add a community list entry
    defaults              :  Set of configuration defaults used
    enable                :  To make able
    extcommunity-list     :  An extended community list
    import-table          :  Import routes from non-main kernel table
    large-community-list  :  BGP large community-list
    line                  :  A terminal line
    log                   :  Logging control
    mroute                :  Static unicast routes in MRIB for multicast RPF lookup
    password              :  Set a password
    prefix-list           :  Filter updates to/from this neighbor
    protocol              :  Filter routing info exchanged between zebra and protocol
    ptm-enable            :  Enable neighbor check with specified topology
    route                 :  Static routes
    route-map             :  Route-map
    service               :  Service
    zebra                 :  Zebra information
```

Among other things, you can debug zebra, create route-maps from here as well. To serve the purpose of this topology, we simply need to create a static route for 20.1.1.0/24 on SW1 with a next hop of SW2 and a static route for 10.1.1.0/24 on SW2 with a next hop of SW1.

```
cumulus@SW1:~$ net add routing route 20.1.1.0/24 172.16.12.2
cumulus@SW1:~$ net commit

--- /run/nclu/frr/frr.conf.scratchpad.baseline  2019-04-29 14:28:53.585648028 +0000
+++ /run/nclu/frr/frr.conf.scratchpad   2019-04-29 14:28:53.585648028 +0000
@@ -1,9 +1,11 @@
 frr version 4.0+cl3u8
 frr defaults datacenter
 hostname SW1
 username cumulus nopassword
 service integrated-vtysh-config
 log syslog informational
 line vty
 
 end
+ip route 20.1.1.0/24 172.16.12.2
+end


net add/del commands since the last "net commit"
================================================

User     Timestamp                   Command
-------  --------------------------  ---------------------------------------------
cumulus  2019-04-29 14:28:53.587527  net add routing route 20.1.1.0/24 172.16.12.2
```


```
cumulus@SW2:~$ net add routing route 10.1.1.0/24 172.16.12.1
cumulus@SW2:~$ net commit 
--- /run/nclu/frr/frr.conf.scratchpad.baseline  2019-04-29 14:29:37.915465608 +0000
+++ /run/nclu/frr/frr.conf.scratchpad   2019-04-29 14:29:37.915465608 +0000
@@ -1,9 +1,11 @@
 frr version 4.0+cl3u8
 frr defaults datacenter
 hostname SW2
 username cumulus nopassword
 service integrated-vtysh-config
 log syslog informational
 line vty
 
 end
+ip route 10.1.1.0/24 172.16.12.1
+end


net add/del commands since the last "net commit"
================================================

User     Timestamp                   Command
-------  --------------------------  ---------------------------------------------
cumulus  2019-04-29 14:29:37.918018  net add routing route 10.1.1.0/24 172.16.12.1
```


The changes made to /etc/frr/frr.conf are highlighted in the previous outputs (in blue). PC1 can ping PC2 now: 

```
PC-1> ping 20.1.1.1
84 bytes from 20.1.1.1 icmp_seq=1 ttl=62 time=8.933 ms
84 bytes from 20.1.1.1 icmp_seq=2 ttl=62 time=1.999 ms
84 bytes from 20.1.1.1 icmp_seq=3 ttl=62 time=2.573 ms
84 bytes from 20.1.1.1 icmp_seq=4 ttl=62 time=2.043 ms
84 bytes from 20.1.1.1 icmp_seq=5 ttl=62 time=2.237 ms
```

Pretty straightforward, wasn't it?


I am going to undo these changes now and move towards a routing protocol like OSPF instead.

```
cumulus@SW1:~$ net del routing route 20.1.1.0/24 172.16.12.2
cumulus@SW1:~$ net commit

cumulus@SW2:~$ net del routing route 10.1.1.0/24 172.16.12.1
cumulus@SW2:~$ net commit 
```


Let's bring up OSPF now.

```
// SW1 configuration

cumulus@SW1:~$ net add ospf router-id 1.1.1.1
cumulus@SW1:~$ net add interface swp2 ospf area 0     
cumulus@SW1:~$ net add interface swp2 ospf network point-to-point 
cumulus@SW1:~$ net add interface swp1 ospf area 0
cumulus@SW1:~$ net add interface swp1 ospf passive
cumulus@SW1:~$ net commit

// SW2 configuration

cumulus@SW2:~$ net add ospf router-id 2.2.2.2
cumulus@SW2:~$ net add interface swp2 ospf area 0
cumulus@SW2:~$ net add interface swp2 ospf network point-to-point
cumulus@SW2:~$ net add interface swp1 ospf area 0
cumulus@SW2:~$ net add interface swp1 ospf passive 
cumulus@SW2:~$ net commit 
```

Verify that OSPF is up and that the LSDB has the correct information:

```
cumulus@SW1:~$ net show ospf neighbor 

Neighbor ID     Pri State           Dead Time Address         Interface            RXmtL RqstL DBsmL
2.2.2.2           1 Full/DROther      36.571s 172.16.12.2     swp2:172.16.12.1         0     0     0


cumulus@SW2:~$ net show ospf neighbor 

Neighbor ID     Pri State           Dead Time Address         Interface            RXmtL RqstL DBsmL
1.1.1.1           1 Full/DROther      35.519s 172.16.12.1     swp2:172.16.12.2         0     0     0
```

```
cumulus@SW1:~$ net show ospf database 

       OSPF Router with ID (1.1.1.1)

                Router Link States (Area 0.0.0.0)

Link ID         ADV Router      Age  Seq#       CkSum  Link count
1.1.1.1         1.1.1.1           79 0x80000006 0x470d 3
2.2.2.2         2.2.2.2          100 0x80000006 0x1e27 3

cumulus@SW1:~$ net show ospf database router 1.1.1.1

       OSPF Router with ID (1.1.1.1)


                Router Link States (Area 0.0.0.0)

  LS age: 89
  Options: 0x2  : *|-|-|-|-|-|E|-
  LS Flags: 0x3  
  Flags: 0x0
  LS Type: router-LSA
  Link State ID: 1.1.1.1 
  Advertising Router: 1.1.1.1
  LS Seq Number: 80000006
  Checksum: 0x470d
  Length: 60

   Number of Links: 3

    Link connected to: another Router (point-to-point)
     (Link ID) Neighboring Router ID: 2.2.2.2
     (Link Data) Router Interface address: 172.16.12.1
      Number of TOS metrics: 0
       TOS 0 Metric: 100

    Link connected to: Stub Network
     (Link ID) Net: 172.16.12.0
     (Link Data) Network Mask: 255.255.255.0
      Number of TOS metrics: 0
       TOS 0 Metric: 100

    Link connected to: Stub Network
     (Link ID) Net: 10.1.1.0
     (Link Data) Network Mask: 255.255.255.0
      Number of TOS metrics: 0
       TOS 0 Metric: 100


cumulus@SW1:~$ net show ospf database router 2.2.2.2

       OSPF Router with ID (1.1.1.1)


                Router Link States (Area 0.0.0.0)

  LS age: 114
  Options: 0x2  : *|-|-|-|-|-|E|-
  LS Flags: 0x6  
  Flags: 0x0
  LS Type: router-LSA
  Link State ID: 2.2.2.2 
  Advertising Router: 2.2.2.2
  LS Seq Number: 80000006
  Checksum: 0x1e27
  Length: 60

   Number of Links: 3

    Link connected to: another Router (point-to-point)
     (Link ID) Neighboring Router ID: 1.1.1.1
     (Link Data) Router Interface address: 172.16.12.2
      Number of TOS metrics: 0
       TOS 0 Metric: 100

    Link connected to: Stub Network
     (Link ID) Net: 172.16.12.0
     (Link Data) Network Mask: 255.255.255.0
      Number of TOS metrics: 0
       TOS 0 Metric: 100

    Link connected to: Stub Network
     (Link ID) Net: 20.1.1.0
     (Link Data) Network Mask: 255.255.255.0
      Number of TOS metrics: 0
       TOS 0 Metric: 100
```

SW1 and SW2 should have installed 10.1.1.0/24 and 20.1.1.0/24 respectively into RIB/FIB as well.

```
cumulus@SW1:~$ net show route ospf
RIB entry for ospf
==================
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR,
       > - selected route, * - FIB route

O   10.1.1.0/24 [110/100] is directly connected, swp1, 00:02:52
O>* 20.1.1.0/24 [110/200] via 172.16.12.2, swp2, 00:02:42
O   172.16.12.0/24 [110/100] is directly connected, swp2, 00:02:52

cumulus@SW2:~$ net show route ospf
RIB entry for ospf
==================
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR,
       > - selected route, * - FIB route

O>* 10.1.1.0/24 [110/200] via 172.16.12.1, swp2, 00:03:51
O   20.1.1.0/24 [110/100] is directly connected, swp1, 00:04:21
O   172.16.12.0/24 [110/100] is directly connected, swp2, 00:04:21
```

The only thing left to verify is the connectivity from PC1 to PC2. 

```
PC-1> ping 20.1.1.1
84 bytes from 20.1.1.1 icmp_seq=1 ttl=62 time=3.140 ms
84 bytes from 20.1.1.1 icmp_seq=2 ttl=62 time=2.152 ms
84 bytes from 20.1.1.1 icmp_seq=3 ttl=62 time=2.563 ms
84 bytes from 20.1.1.1 icmp_seq=4 ttl=62 time=2.020 ms
84 bytes from 20.1.1.1 icmp_seq=5 ttl=62 time=3.031 ms
```

Works like a charm! In our next post, we'll take a look at implementing BGP with OSPF as an IGP on Cumulus VX. 