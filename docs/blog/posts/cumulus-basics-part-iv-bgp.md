---
date:
  created: 2021-12-09
  updated: 2021-12-09
authors:
    - aninchat
blog_toc: true
categories:
  - cumulus
  - bgp
tags:
  - cumulus
  - bgp
---
# Cumulus Basics Part IV - BGP introduction
In this post, we introduce BGP on Cumulus Linux.

## Introduction

The goal of this post is to introduce BGP on Cumulus Linux and then move towards a BGP unnumbered design, in the following post. 

## Topology

We'll be using the following network topology for this post:

![bgp1](/static/images/cumulus/cumulus_part4/cumulus_bgp_1.jpg)

First, we will try to create a traditional BGP scenario with OSPF as an IGP. For now, OSPF is up and running and I have learnt the loopback of each LEAF switch.

<!-- more -->

```
// RIB lookup on LEAF1 for LEAF2s loopback

cumulus@LEAF1:~$ net show route 2.2.2.2
RIB entry for 2.2.2.2
=====================
Routing entry for 2.2.2.2/32
  Known via "ospf", distance 110, metric 200, best
  Last update 00:02:22 ago
  * 172.16.11.11, via swp1
  * 172.16.12.22, via swp2


FIB entry for 2.2.2.2
=====================
2.2.2.2  proto ospf  metric 20 
        nexthop via 172.16.11.11  dev swp1 weight 1
        nexthop via 172.16.12.22  dev swp2 weight 1 

// RIB lookup on LEAF1 for LEAF3s loopback

cumulus@LEAF1:~$ net show route 3.3.3.3
RIB entry for 3.3.3.3
=====================
Routing entry for 3.3.3.3/32
  Known via "ospf", distance 110, metric 200, best
  Last update 00:02:54 ago
  * 172.16.11.11, via swp1
  * 172.16.12.22, via swp2


FIB entry for 3.3.3.3
=====================
3.3.3.3  proto ospf  metric 20 
        nexthop via 172.16.11.11  dev swp1 weight 1
        nexthop via 172.16.12.22  dev swp2 weight 1 
```

I can use these loopbacks now to form an iBGP session between LEAF1 and LEAF2 to provide connectivity between PC1 and PC2.

```
// LEAF1 configuration

cumulus@LEAF1:~$ net add bgp autonomous-system 1
cumulus@LEAF1:~$ net add bgp router-id 1.1.1.1  
cumulus@LEAF1:~$ net add bgp neighbor 2.2.2.2 remote-as 1
cumulus@LEAF1:~$ net add bgp neighbor 2.2.2.2 update-source lo
cumulus@LEAF1:~$ net commit 

// LEAF2 configuration

cumulus@LEAF2:~$ net add bgp autonomous-system 1
cumulus@LEAF2:~$ net add bgp router-id 2.2.2.2
cumulus@LEAF2:~$ net add bgp neighbor 1.1.1.1 remote-as 1
cumulus@LEAF2:~$ net add bgp neighbor 1.1.1.1 update-source lo
cumulus@LEAF2:~$ net commit  
```

An iBGP session is now up between the two:

```
cumulus@LEAF1:~$ net show bgp ipv4 unicast summary 
BGP router identifier 1.1.1.1, local AS number 1 vrf-id 0
BGP table version 0
RIB entries 0, using 0 bytes of memory
Peers 1, using 19 KiB of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
LEAF2(2.2.2.2)  4          1      21      21        0    0    0 00:00:56            0
```

Advertise the host subnets now into BGP:

```
cumulus@LEAF1:~$ net add bgp network 10.1.1.0/24
cumulus@LEAF1:~$ net commit 

cumulus@LEAF2:~$ net add bgp network 20.1.1.0/24
cumulus@LEAF2:~$ net commit    

// IPv4 unicast BGP table on LEAF1

cumulus@LEAF1:~$ net show bgp ipv4 unicast 
BGP table version is 2, local router ID is 1.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, = multipath,
              i internal, r RIB-failure, S Stale, R Removed
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.1.1.0/24      0.0.0.0                  0         32768 i
*>i20.1.1.0/24      2.2.2.2                  0    100      0 i

Displayed  2 routes and 2 total paths 

// IPv4 unicast BGP table on LEAF2

cumulus@LEAF2:~$ net show bgp ipv4 unicast                  
BGP table version is 2, local router ID is 2.2.2.2
Status codes: s suppressed, d damped, h history, * valid, > best, = multipath,
              i internal, r RIB-failure, S Stale, R Removed
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*>i10.1.1.0/24      1.1.1.1                  0    100      0 i
*> 20.1.1.0/24      0.0.0.0                  0         32768 i
```


Let's try and ping from PC1 to PC2 now:

```
PC1> ping 20.1.1.1

20.1.1.1 icmp_seq=1 timeout
20.1.1.1 icmp_seq=2 timeout
20.1.1.1 icmp_seq=3 timeout
20.1.1.1 icmp_seq=4 timeout
20.1.1.1 icmp_seq=5 timeout
```

All pings timeout. This is a common problem that you may run into -  look at what is happening here with more thought. PC1 sources a packet with 10.1.1.1 with an IP destination of 20.1.1.1, with an ICMP header trailing it. This reaches the default gateway, LEAF1. LEAF1 does a mac lookup, realizes it owns the destination mac and thus moves into the IP header to do a RIB lookup and forwards it towards SPINE1 (assuming the packet hash is in such a way that it goes to SPINE1).

```
// RIB lookup on LEAF1 for 20.1.1.1

cumulus@LEAF1:~$ net show route 20.1.1.1
RIB entry for 20.1.1.1
======================
Routing entry for 20.1.1.0/24
  Known via "bgp", distance 200, metric 0, best
  Last update 00:03:17 ago
    2.2.2.2 (recursive)
  *   172.16.11.11, via swp1
  *   172.16.12.22, via swp2


FIB entry for 20.1.1.1
======================
20.1.1.0/24  proto bgp  metric 20 
        nexthop via 172.16.11.11  dev swp1 weight 1
        nexthop via 172.16.12.22  dev swp2 weight 1 

The packet can be sent towards SPINE1/SPINE2. Let's take SPINE1 as the next hop as an example here. Does SPINE1 know how to reach 20.1.1.1? 

// RIB lookup on SPINE1 for 20.1.1.1

cumulus@SPINE1:~$ net show route 20.1.1.1
RIB entry for 20.1.1.1
======================
% Network not in table 
```

SPINE1 has no entry for this prefix and thus your packets get blackholed here. To get around this problem, you either redistribute your BGP table into your IGP (which doesn't make sense considering your BGP table might grow substantially) or you have some sort of a meshed iBGP peering to ensure all boxes receive this route. The cleanest way of doing this would be with route reflectors. So, let's go ahead and make SPINE1/SPINE2 as RRs and have LEAF1/2/3 peer with them as route reflector clients.


I have now modified the configuration appropriately:


```
// IPv4 unicast BGP table on LEAF1

cumulus@LEAF1:~$ net show bgp ipv4 unicast summary 
BGP router identifier 1.1.1.1, local AS number 1 vrf-id 0
BGP table version 0
RIB entries 0, using 0 bytes of memory
Peers 2, using 39 KiB of memory

Neighbor            V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
SPINE1(11.11.11.11) 4          1      22      22        0    0    0 00:00:57            0
SPINE2(22.22.22.22) 4          1       9       9        0    0    0 00:00:20            0

Total number of neighbors 2 

// IPv4 unicast BGP table on LEAF2

cumulus@LEAF2:~$ net show bgp ipv4 unicast summary 
BGP router identifier 2.2.2.2, local AS number 1 vrf-id 0
BGP table version 0
RIB entries 0, using 0 bytes of memory
Peers 2, using 39 KiB of memory

Neighbor            V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
SPINE1(11.11.11.11) 4          1      13      13        0    0    0 00:00:30            0
SPINE2(22.22.22.22) 4          1      23      23        0    0    0 00:01:01            0

Total number of neighbors 2 
```


Go back in and advertise the host subnets again and now we see the intermediate devices (SPINE1/SPINE2) also having the prefixes in RIB.

```
cumulus@SPINE1:~$ net show route 20.1.1.1
RIB entry for 20.1.1.1
======================
Routing entry for 20.1.1.0/24
  Known via "bgp", distance 200, metric 0, best
  Last update 00:00:05 ago
    2.2.2.2 (recursive)
  *   172.16.21.2, via swp2


FIB entry for 20.1.1.1
======================
20.1.1.0/24 via 172.16.21.2 dev swp2  proto bgp  metric 20
```


PC1 can now reach PC2:

```
PC1> ping 20.1.1.1

84 bytes from 20.1.1.1 icmp_seq=1 ttl=61 time=2.354 ms
84 bytes from 20.1.1.1 icmp_seq=2 ttl=61 time=1.812 ms
84 bytes from 20.1.1.1 icmp_seq=3 ttl=61 time=1.590 ms
84 bytes from 20.1.1.1 icmp_seq=4 ttl=61 time=1.872 ms
84 bytes from 20.1.1.1 icmp_seq=5 ttl=61 time=0.829 ms
```


In the next post, we'll take a look at the ingenious BGP unnumbered design and understand how it truly works.