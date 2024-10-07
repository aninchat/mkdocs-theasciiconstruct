---
date:
  created: 2021-12-13
  updated: 2021-12-13
authors:
    - aninchat
blog_toc: true
categories:
  - cumulus
  - vxlan
  - evpn
tags:
  - cumulus
  - vxlan
  - evpn
---
# Cumulus Basics Part VIII - VXLAN symmetric routing and multi-tenancy

In this post, we look at VXLAN routing with symmetric IRB and multi-tenancy on Cumulus Linux.

## Introduction

Now that we've configured and verified a working asymmetric VXLAN routing solution in the previous post, let's take a look at the greener side of the grass (well, it depends on where you stand) - symmetric IRB. This post is going to introduce VRFs into the picture that pave the way for multi-tenancy in VXLAN solutions. 

## Topology

We continue to use the same topology as our previous post:

![symm1](/static/images/cumulus/cumulus_part8/cumulus_symmetric_1.jpg)

<!-- more -->

Any configuration we added for asymmetric routing has been removed. We simply have our L2VNIs created on LEAF1 and LEAF3 and the BGP peerings are up. 


Before we begin with the actual configuration, let's understand the logic behind symmetric IRB. This functions as a bridge-route-route-bridge model where the packet is bridged to your source VTEP from the host and is then routed into a new type of VNI called the L3VNI. This takes the packet through your fabric core to the destination VTEP, where it is routed from the L3VNI into the local L2VNI and then bridged across to the destination host. 


In addition to the L3VNIs, we also introduce VRFs here. VRFs allow for multi-tenancy. Imagine this - instead of having a dedicated core infrastructure for every customer, you could have this core infrastructure common to multiple customers with the kind of segregation that VRFs can provide. The L3VNIs are tied to the customer VRFs directly and in that sense, the L3VNIs should be unique per VRF.


The configuration for symmetric IRB is not very complicated. Let's take LEAF1 as an example and start to configure this:

```
// first, create a VLAN for your L3VNI 

cumulus@LEAF1:~$ net add vlan 40 alias VLAN for L3VNI 10040

// next, create the L3VNI, map it to the VLAN and configure the tunnel source 

cumulus@LEAF1:~$ net add vxlan vni40 vxlan id 10040
cumulus@LEAF1:~$ net add vxlan vni40 vxlan local-tunnelip 1.1.1.1
cumulus@LEAF1:~$ net add vxlan vni40 bridge access 40
 
// now create the tenant VRF and map it to the L3VNI

cumulus@LEAF1:~$ net add vrf TENANT1 vni 10040
```

Now that we have some of these pieces configured, we need to start making sense of this and putting it all together. 


Naturally, to segregate your customers, they need to be put in their respective tenant VRFs. This means that the customer VLAN (particularly, the first L3 hop, which is the corresponding SVI in this case) needs to be in the tenant VRF. Additionally, you add the corresponding VLAN for the L3VNI in the same tenant VRF. 

```
// customer VLAN goes into the tenant VRF

cumulus@LEAF1:~$ net add vlan 10 vrf TENANT1

// VLAN corresponding to the L3VNI goes into tenant VRF as well

cumulus@LEAF1:~$ net add vlan 40 vrf TENANT1
```


Confirm that the customer facing SVI and the SVI corresponding to the L3VNI are up and in the correct VRF:

```
// customer facing SVI 

cumulus@LEAF1:~$ net show interface vlan10
    Name    MAC                Speed  MTU   Mode
--  ------  -----------------  -----  ----  ------------
UP  vlan10  50:00:00:04:00:03  N/A    1500  Interface/L3

IP Details
-------------------------  -------------
IP:                        10.1.1.254/24
IP Neighbor(ARP) Entries:  1

cl-netstat counters
-------------------
RX_OK  RX_ERR  RX_DRP  RX_OVR  TX_OK  TX_ERR  TX_DRP  TX_OVR
-----  ------  ------  ------  -----  ------  ------  ------
   22       0       0       0     36       0       0       0

Routing
-------
  Interface vlan10 is up, line protocol is up
  Link ups:       1    last: 2019/06/06 03:30:36.79
  Link downs:     1    last: 2019/06/06 03:30:36.79
  PTM status: disabled
  vrf: TENANT1
  index 10 metric 0 mtu 1500 speed 0
  flags: <UP,BROADCAST,RUNNING>,MULTICAST>
  Type: Ethernet
  HWaddr: 50:00:00:04:00:03
  inet 10.1.1.254/24
  inet6 fe80::5200:ff:fe04:3/64
  Interface Type Vlan
  VLAN Id 10
  Link ifindex 9(bridge)

// SVI corresponding to the L3VNI  

cumulus@LEAF1:~$ net show interface vlan40
    Name    MAC                Speed  MTU   Mode
--  ------  -----------------  -----  ----  -------------
UP  vlan40  50:00:00:04:00:03  N/A    1500  NotConfigured

Alias
-----
VLAN for L3VNI 10040

cl-netstat counters
-------------------
RX_OK  RX_ERR  RX_DRP  RX_OVR  TX_OK  TX_ERR  TX_DRP  TX_OVR
-----  ------  ------  ------  -----  ------  ------  ------
   13       0       0       0     20       0       0       0

Routing
-------
  Interface vlan40 is up, line protocol is up
  Link ups:       3    last: 2019/06/06 03:32:58.69
  Link downs:     2    last: 2019/06/06 03:32:58.69
  PTM status: disabled
  vrf: TENANT1
  Description: VLAN for L3VNI 10040
  index 12 metric 0 mtu 1500 speed 0
  flags: <UP,BROADCAST,RUNNING>,MULTICAST>
  Type: Unknown
  HWaddr: 50:00:00:04:00:03
  inet6 fe80::5200:ff:fe04:3/64
  Interface Type Vlan
  VLAN Id 40
  Link ifindex 9(bridge)
```


Both the L2VNI and the L3VNI is going to be assigned a RD/RT value. You can confirm this using 'net show bgp evpn vni":

```
cumulus@LEAF1:~$ net show bgp evpn vni 
Advertise Gateway Macip: Disabled
Advertise SVI Macip: Disabled
Advertise All VNI flag: Enabled
BUM flooding: Head-end replication
Number of L2 VNIs: 1
Number of L3 VNIs: 1
Flags: * - Kernel
  VNI        Type RD                    Import RT                 Export RT                 Tenant VRF                           
* 10010      L2   1.1.1.1:3             1:10010                   1:10010                  TENANT1                              
* 10040      L3   10.1.1.254:2          1:10040                   1:10040                  TENANT1
```


Similar configuration is done on LEAF3:

```
cumulus@LEAF3:~$ net add vlan 40
cumulus@LEAF3:~$ net add vxlan vni40 vxlan id 10040
cumulus@LEAF3:~$ net add vxlan vni40 vxlan local-tunnelip 3.3.3.3
cumulus@LEAF3:~$ net add vxlan vni40 bridge access 40
cumulus@LEAF3:~$ net add vrf TENANT1 vni 10040
cumulus@LEAF3:~$ net add vlan 30 vrf TENANT1
cumulus@LEAF3:~$ net add vlan 40 vrf TENANT1
```


Let's take a quick peek at the BGP EVPN table and see what is in there:

```
// LEAF1s BGP EVPN table

cumulus@LEAF1:~$ net show bgp l2vpn evpn route
BGP table version is 10, local router ID is 1.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-2 prefix: [2]:[ESI]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[ESI]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 1.1.1.1:3
*> [2]:[0]:[0]:[48]:[00:50:79:66:68:06]
                    1.1.1.1                            32768 i
*> [2]:[0]:[0]:[48]:[00:50:79:66:68:06]:[32]:[10.1.1.1]
                    1.1.1.1                            32768 i
*> [3]:[0]:[32]:[1.1.1.1]
                    1.1.1.1                            32768 i
Route Distinguisher: 2.2.2.2:2
* i[3]:[0]:[32]:[2.2.2.2]
                    2.2.2.2                  0    100      0 i
*>i[3]:[0]:[32]:[2.2.2.2]
                    2.2.2.2                  0    100      0 i


// LEAF3s BGP EVPN table

cumulus@LEAF3:~$ net show bgp l2vpn evpn route
BGP table version is 3, local router ID is 3.3.3.3
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-2 prefix: [2]:[ESI]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[ESI]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 1.1.1.1:3
* i[2]:[0]:[0]:[48]:[00:50:79:66:68:06]
                    1.1.1.1                  0    100      0 i
*>i[2]:[0]:[0]:[48]:[00:50:79:66:68:06]
                    1.1.1.1                  0    100      0 i
* i[2]:[0]:[0]:[48]:[00:50:79:66:68:06]:[32]:[10.1.1.1]
                    1.1.1.1                  0    100      0 i
*>i[2]:[0]:[0]:[48]:[00:50:79:66:68:06]:[32]:[10.1.1.1]
                    1.1.1.1                  0    100      0 i
* i[3]:[0]:[32]:[1.1.1.1]
                    1.1.1.1                  0    100      0 i
*>i[3]:[0]:[32]:[1.1.1.1]
                    1.1.1.1                  0    100      0 i
Route Distinguisher: 2.2.2.2:2
* i[3]:[0]:[32]:[2.2.2.2]
                    2.2.2.2                  0    100      0 i
*>i[3]:[0]:[32]:[2.2.2.2]
                    2.2.2.2                  0    100      0 i
Route Distinguisher: 3.3.3.3:2
*> [3]:[0]:[32]:[3.3.3.3]
                    3.3.3.3                            32768 i
```

It appears that PC1s mac and IP address have already been learnt and installed in the BGP EVPN table. However, there is no information about PC3 in here (most likely because PC3 has not sent any traffic so far). So, let's take this opportunity to generate some traffic from PC3 and understand how the control-plane is built with L3VNIs.



We will enable some debugs to understand what is going on. As explained in the previous post, you must enable logging of syslogs at the debugging level and then enable the debugs. The following debugs were enabled:

```
LEAF3# show debug
Zebra debugging status:
  Zebra event debugging is on
  Zebra packet detail debugging is on
  Zebra kernel debugging is on
  Zebra RIB debugging is on
  Zebra VXLAN debugging is on

BGP debugging status:
  BGP updates debugging is on (inbound)
  BGP updates debugging is on (outbound)
  BGP zebra debugging is on
  BGP vpn label event debugging is on
```

The same debugs were enabled on LEAF1 as well so as to capture simultaneous debugging information from both. 


We generate some traffic from PC3 now (by pinging its default gateway, which is SVI30 on LEAF3). Remember, all debugs are redirected to /var/log/frr/frr.log. Snippets of relevant debug logs are below:


```
// LEAF3

	// LEAF3 zebra learns PC3s mac address and notifies BGP

	2019-06-06T10:01:51.630081+00:00 LEAF3 zebra[936]: ADD MAC 00:50:79:66:68:08 intf swp2(4) VID 30 -> VNI 10030
	2019-06-06T10:01:51.630191+00:00 LEAF3 zebra[936]: Send MACIP Add flags 0x0 MAC 00:50:79:66:68:08 IP  seq 0 L2-VNI 10030 to bgp
	2019-06-06T10:01:51.630301+00:00 LEAF3 zebra[936]: netlink_parse_info: netlink-listen (NS 0) type RTM_NEWROUTE(24), len=116, seq=0, pid=0
	2019-06-06T10:01:51.630570+00:00 LEAF3 zebra[936]: RTM_NEWROUTE ipv6 unicast proto  NS 0

	// bgpd receives the notification from zebra

	2019-06-06T10:01:51.632750+00:00 LEAF3 bgpd[948]: 0:Recv MACIP Add flags 0x0 MAC 00:50:79:66:68:08 IP  VNI 10030 seq 0 state 0
	2019-06-06T10:01:51.683158+00:00 LEAF3 bgpd[948]: group_announce_route_walkcb: afi=l2vpn, safi=evpn, p=[2]:[00:50:79:66:68:08]/224
	2019-06-06T10:01:51.683739+00:00 LEAF3 bgpd[948]: subgroup_process_announce_selected: p=[2]:[00:50:79:66:68:08]/224, selected=0x338517eed0

	// bgpd sends an update to its peer about this new learn

	2019-06-06T10:01:51.684041+00:00 LEAF3 bgpd[948]: u2:s2 send UPDATE w/ attr: nexthop 3.3.3.3, localpref 100, extcommunity ET:8 RT:1:10030 RT:1:10040 	Rmac:50:00:00:03:00:02, path
	2019-06-06T10:01:51.684266+00:00 LEAF3 bgpd[948]: u2:s2 send MP_REACH for afi/safi 25/70
	2019-06-06T10:01:51.684441+00:00 LEAF3 bgpd[948]: u2:s2 send UPDATE RD 3.3.3.3:2 [2]:[00:50:79:66:68:08]/224 label 10030/10040 l2vpn evpn
	2019-06-06T10:01:51.684808+00:00 LEAF3 bgpd[948]: u2:s2 send UPDATE len 121 numpfx 1
	2019-06-06T10:01:51.685071+00:00 LEAF3 bgpd[948]: u2:s2 swp1 send UPDATE w/ nexthop 3.3.3.3
	2019-06-06T10:01:51.685410+00:00 LEAF3 bgpd[948]: u2:s2 swp3 send UPDATE w/ nexthop 3.3.3.3

// LEAF1

	// LEAF1s bgpd receives the update from SPINE1

	2019-06-06T10:01:51.929153+00:00 LEAF1 bgpd[890]: swp1 rcvd UPDATE w/ attr: nexthop 3.3.3.3, localpref 100, metric 0, extcommunity RT:1:10030 RT:1:10040 ET:8 	Rmac:50:00:00:03:00:02, originator 3.3.3.3, clusterlist 11.11.11.11, path
	2019-06-06T10:01:51.929462+00:00 LEAF1 bgpd[890]: swp1 rcvd UPDATE wlen 0 attrlen 126 alen 0
	2019-06-06T10:01:51.929665+00:00 LEAF1 bgpd[890]: swp1 rcvd RD 3.3.3.3:2 [2]:[00:50:79:66:68:08]:[30.1.1.1]/224 label 10030/10040 l2vpn evpn

	// LEAF1s bgpd installs the prefix in the BGP EVPN table

	2019-06-06T10:01:51.929940+00:00 LEAF1 bgpd[890]: installing evpn prefix [2]:[00:50:79:66:68:08]:[30.1.1.1]/224 as ip prefix 30.1.1.1/32 in vrf TENANT1

	// LEAF1s bgpd receives the update from SPINE2

	2019-06-06T10:01:51.930173+00:00 LEAF1 bgpd[890]: swp2 rcvd UPDATE w/ attr: nexthop 3.3.3.3, localpref 100, metric 0, extcommunity RT:1:10030 RT:1:10040 ET:8 	Rmac:50:00:00:03:00:02, originator 3.3.3.3, clusterlist 22.22.22.22, path
	2019-06-06T10:01:51.930381+00:00 LEAF1 bgpd[890]: swp2 rcvd UPDATE wlen 0 attrlen 126 alen 0
	2019-06-06T10:01:51.930550+00:00 LEAF1 bgpd[890]: swp2 rcvd RD 3.3.3.3:2 [2]:[00:50:79:66:68:08]:[30.1.1.1]/224 label 10030/10040 l2vpn evpn

	// LEAF1s bgpd installs the prefix in the BGP EVPN table

	2019-06-06T10:01:51.930799+00:00 LEAF1 bgpd[890]: installing evpn prefix [2]:[00:50:79:66:68:08]:[30.1.1.1]/224 as ip prefix 30.1.1.1/32 in vrf TENANT1

	// LEAF1s bgpd notifies zebra

	2019-06-06T10:01:51.981956+00:00 LEAF1 bgpd[890]: bgp_zebra_announce: p=30.1.1.1/32, bgp_is_valid_label: 2
	2019-06-06T10:01:51.982328+00:00 LEAF1 bgpd[890]: Tx route add VRF 14 30.1.1.1/32 metric 0 tag 0 flags 0x1409 nhnum 1
	2019-06-06T10:01:51.982524+00:00 LEAF1 bgpd[890]:   nhop [1]: 303:303:: if 12 VRF 14
	2019-06-06T10:01:51.982848+00:00 LEAF1 bgpd[890]: bgp_zebra_announce: 30.1.1.1/32: announcing to zebra (recursion set)


	// LEAF1s zebra gets this message from bgpd and processes it
	
	2019-06-06T10:01:51.983503+00:00 LEAF1 zebra[881]: zebra message comes from socket [16]
	2019-06-06T10:01:51.983768+00:00 LEAF1 zebra[881]: Tx RTM_NEWNEIGH family ipv4 IF vlan40(12) Neigh 3.3.3.3 MAC 50:00:00:03:00:02 flags 0x10 state 0x40
	2019-06-06T10:01:51.983959+00:00 LEAF1 zebra[881]: netlink_talk: netlink-cmd (NS 0) type RTM_NEWNEIGH(28), len=48 seq=46 flags 0x505
	2019-06-06T10:01:51.984200+00:00 LEAF1 zebra[881]: netlink_parse_info: netlink-cmd (NS 0) ACK: type=RTM_NEWNEIGH(28), seq=46, pid=4294963174
	2019-06-06T10:01:51.984498+00:00 LEAF1 zebra[881]: Tx RTM_NEWNEIGH family bridge IF vni40(13) VLAN 40 MAC 50:00:00:03:00:02 dst 3.3.3.3
	2019-06-06T10:01:51.984770+00:00 LEAF1 zebra[881]: netlink_talk: netlink-cmd (NS 0) type RTM_NEWNEIGH(28), len=64 seq=47 flags 0x505
	2019-06-06T10:01:51.985005+00:00 LEAF1 zebra[881]: netlink_parse_info: netlink-cmd (NS 0) ACK: type=RTM_NEWNEIGH(28), seq=47, pid=4294963174
	2019-06-06T10:01:51.985202+00:00 LEAF1 zebra[881]: rib_add_multipath: 14:30.1.1.1/32: Inserting route rn 0x653ba3c640, re 0x653b8a7e60 (type 9) existing (nil)
```

    
As you can see, this gets learnt in the BGP EVPN table on LEAF1 and pushed to RIB/FIB as well (for simplicity sake, I have shut down all links to SPINE2):


```
// LEAF1s BGP EVPN table

cumulus@LEAF1:~$ net show bgp l2vpn evpn route
BGP table version is 4, local router ID is 1.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-2 prefix: [2]:[ESI]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[ESI]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 1.1.1.1:3
*> [2]:[0]:[0]:[48]:[00:50:79:66:68:06]
                    1.1.1.1                            32768 i
*> [2]:[0]:[0]:[48]:[00:50:79:66:68:06]:[32]:[10.1.1.1]
                    1.1.1.1                            32768 i
*> [3]:[0]:[32]:[1.1.1.1]
                    1.1.1.1                            32768 i
Route Distinguisher: 2.2.2.2:2
*>i[3]:[0]:[32]:[2.2.2.2]
                    2.2.2.2                  0    100      0 i
Route Distinguisher: 3.3.3.3:2
*>i[2]:[0]:[0]:[48]:[00:50:79:66:68:08]
                    3.3.3.3                  0    100      0 i
*>i[2]:[0]:[0]:[48]:[00:50:79:66:68:08]:[32]:[30.1.1.1]
                    3.3.3.3                  0    100      0 i
*>i[3]:[0]:[32]:[3.3.3.3]
                    3.3.3.3                  0    100      0 i

// LEAF1s RIB/FIB                    

cumulus@LEAF1:~$ net show route vrf TENANT1 30.1.1.1
RIB entry for 30.1.1.1 in vrf TENANT1
=====================================
Routing entry for 30.1.1.1/32
  Known via "bgp", distance 200, metric 0, vrf TENANT1, best
  Last update 00:04:26 ago
  * 3.3.3.3, via vlan40 onlink


FIB entry for 30.1.1.1 in vrf TENANT1
=====================================
30.1.1.1 via 3.3.3.3 dev vlan40  proto bgp  metric 20 onlink
```


The BGP update itself looks like this:

![symm2](/static/images/cumulus/cumulus_part8/cumulus_symmetric_2.jpg)

The control-plane exchange can be visualized as following:

![symm3](/static/images/cumulus/cumulus_part8/cumulus_symmetric_3.jpg)

## Understanding the data-plane

The ICMP request from PC1 hits LEAF1. Since the destination mac address is owned by LEAF1, it strips off the Layer2 header and does a lookup against the destination IP address in the IP header.

```
cumulus@LEAF1:~$ net show route vrf TENANT1 30.1.1.1
RIB entry for 30.1.1.1 in vrf TENANT1
=====================================
Routing entry for 30.1.1.1/32
  Known via "bgp", distance 200, metric 0, vrf TENANT1, best
  Last update 00:04:26 ago
  * 3.3.3.3, via vlan40 onlink


FIB entry for 30.1.1.1 in vrf TENANT1
=====================================
30.1.1.1 via 3.3.3.3 dev vlan40  proto bgp  metric 20 onlink
```

This gets encapsulated with the appropriate headers and the outer destination IP address is set to 3.3.3.3. The VNI inside the VXLAN header is set to 10040 which is what VLAN 40 is associated to. 

![symm4](/static/images/cumulus/cumulus_part8/cumulus_symmetric_4.jpg)

The data-plane packet, post encapsulation:

![symm5](/static/images/cumulus/cumulus_part8/cumulus_symmetric_5.jpg)

As you can see, the source and destination IP addresses in the outer header belong to the source VTEP and the destination VTEP respectively. A UDP header follows, wherein the destination port signifies the following header as VXLAN. Inside the VXLAN header, the VNI is set to 10040, which is the L3VNI shared between VTEPs, uniquely identifying the VRF. 


This encapsulated packet is sent towards the SPINE1/SPINE2. 

![symm6](/static/images/cumulus/cumulus_part8/cumulus_symmetric_6.jpg)


Assuming it hits SPINE1, a traditional route lookup is done against the destination IP address in the outer header (which is 3.3.3.3 in this case). This is advertised in the underlay and SPINE1 knows that the next-hop for this is LEAF3. 

```
cumulus@SPINE1:~$ net show route 3.3.3.3
RIB entry for 3.3.3.3
=====================
Routing entry for 3.3.3.3/32
  Known via "bgp", distance 200, metric 0, best
  Last update 02:35:42 ago
  * fe80::5200:ff:fe03:3, via swp3


FIB entry for 3.3.3.3
=====================
3.3.3.3 via 169.254.0.1 dev swp3  proto bgp  metric 20 onlink 
```


SPINE1 now forwards this to LEAF3 (remember, the packet is still encapsulated and is simply being routed in the underlay till it reaches the destination VTEP):

![symm7](/static/images/cumulus/cumulus_part8/cumulus_symmetric_7.jpg)


The packet reaches LEAF3 now. The destination mac address in the outer Ethernet header is owned by it so this gets stripped. The destination IP address in the outer IP header is also owned by it, so this gets stripped as well. 


LEAF3 parses the UDP header and understands that a VXLAN header follows. It then parses the VXLAN header - the most relevant information here is the embedded VNI. 


Why is this VNI (the L3VNI) so important? LEAF3 (the destination VTEP) uses this VNI to determine which VRF table should be consulted to do the inner destination IP address lookup in. This is how the separation of customers is achieved end to end. 

```
cumulus@LEAF3:~$ net show vrf vni 
VRF                                   VNI        VxLAN IF             L3-SVI               State Rmac              
TENANT1                               10040      vni40                vlan40               Up    50:00:00:03:00:02 
TENANT2                               10400      vni400               vlan400              Up    50:00:00:03:00:02
```


LEAF3 can now look at the VRF table for TENANT1 to determine how to get to 30.1.1.1:


```
cumulus@LEAF3:~$ net show route vrf TENANT1 30.1.1.1
RIB entry for 30.1.1.1 in vrf TENANT1
=====================================
Routing entry for 30.1.1.0/24
  Known via "connected", distance 0, metric 0, vrf TENANT1, best
  Last update 00:10:47 ago
  * directly connected, vlan30


FIB entry for 30.1.1.1 in vrf TENANT1
=====================================
30.1.1.0/24 dev vlan30  proto kernel  scope link  src 30.1.1.254 
```

Remember, the EVPN arp-cache table should also be populated with information about 30.1.1.1:

```
cumulus@LEAF3:~$ net show evpn arp-cache vni all
VNI 10030 #ARP (IPv4 and IPv6, local and remote) 3

IP                   Type   State    MAC               Remote VTEP          
fe80::5200:ff:fe03:2 local  active   50:00:00:03:00:02
30.1.1.1             local  active   00:50:79:66:68:08
30.1.1.254           local  active   50:00:00:03:00:02

VNI 10300 #ARP (IPv4 and IPv6, local and remote) 2

IP                   Type   State    MAC               Remote VTEP          
fe80::5200:ff:fe03:2 local  active   50:00:00:03:00:02
30.1.1.254           local  active   50:00:00:03:00:02 
```

The packet is now forwarded to PC3 and it can respond back. The same process occurs in the reverse direction as well. 