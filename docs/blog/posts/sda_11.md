---
date:
  created: 2022-01-05
  updated: 2022-01-05
authors:
    - aninchat
blog_toc: true
categories:
  - sda
  - lisp
tags:
  - sda
  - lisp
---
# Cisco SDA Part XI - understanding ARP in SDA
In this post, we look at how an ARP packet flows within a SD-Access fabric.

## Introduction and topology

This probably should have been one of my first posts for SDA but here we are. I've recently come to realize that there is a lot of misconception about how ARP works within the SDA solution - the defacto answer appears to be that it is a part of BUM traffic (broadcast, unknown unicast, multicast), and thus, it will be flooded (implying that there is a dependency on some form of replication, either head-end or via an underlay multicast infrastructure). 

This is not true and it's time to bust that myth! We will continue to use the same topology, only this time, Host1 and Host2 are part of the same subnet (192.2.11.0/24) and the same VN - Corp_VN. 

![static1](/static/images/cisco/sda_11/arp_1.jpg)

<!-- more -->

## Baselining the network

First things first - let's gather some preliminary data to establish a baseline before we start testing. Both hosts have generated some minimal traffic, due to which their IP addresses and macs are present in the IPDT (IP device tracking) tables and the LISP database. 

```
// Edge1s IPDT IP table

Edge1#show device-tracking database interface gig1/0/23
portDB has 2 entries for interface Gi1/0/23, 2 dynamic 
Codes: L - Local, S - Static, ND - Neighbor Discovery, ARP - Address Resolution Protocol, DH4 - IPv4 DHCP, DH6 - IPv6 DHCP, PKT - Other Packet, API - API created
Preflevel flags (prlvl):
0001:MAC and LLA match     0002:Orig trunk            0004:Orig access           
0008:Orig trusted trunk    0010:Orig trusted access   0020:DHCP assigned         
0040:Cga authenticated     0080:Cert authenticated    0100:Statically assigned   


    Network Layer Address               Link Layer Address Interface        vlan prlvl  age   state     Time left        
API 192.2.11.100                            000c.2993.006c  Gi1/0/23       1026  0005    5mn REACHABLE  1536 ms try 0    
ND  FE80::1D22:B4A3:26A2:B064               000c.2993.006c  Gi1/0/23       1026  0005    3mn REACHABLE  130 s try 0   


// Edge2s IPDT IP table

Edge2#show device-tracking database interface gig1/0/6
portDB has 2 entries for interface Gi1/0/6, 2 dynamic 
Codes: L - Local, S - Static, ND - Neighbor Discovery, ARP - Address Resolution Protocol, DH4 - IPv4 DHCP, DH6 - IPv6 DHCP, PKT - Other Packet, API - API created
Preflevel flags (prlvl):
0001:MAC and LLA match     0002:Orig trunk            0004:Orig access           
0008:Orig trusted trunk    0010:Orig trusted access   0020:DHCP assigned         
0040:Cga authenticated     0080:Cert authenticated    0100:Statically assigned   


    Network Layer Address               Link Layer Address Interface        vlan prlvl  age   state     Time left        
ARP 192.2.11.200                            000c.29f2.c674  Gi1/0/6        1026  0005   20s  REACHABLE  287 s try 0      
ND  FE80::4CEA:EE0:9E2D:B08                 000c.29f2.c674  Gi1/0/6        1026  0005  175s  REACHABLE  125 s try 0  


// Edge1s IPDT mac table

Edge1#show device-tracking database mac                    
 MAC              Interface      vlan prlvl      state          time left policy
 7872.5dab.d2e6   Gi1/0/24       2045 NO TRUST   MAC-REACHABLE  294 s     LISP-DT-GUARD-VLAN 
 7486.0b05.1fff   Vl2045         2045 TRUSTED    MAC-STALE      N/A       LISP-DT-GUARD-VLAN 
 7486.0b05.1ff8   Vl1026         1026 TRUSTED    MAC-STALE      N/A       LISP-DT-GUARD-VLAN 
 7486.0b05.1fee   Vl1028         1028 TRUSTED    MAC-DOWN       N/A       LISP-DT-GUARD-VLAN 
 7486.0b05.1fdc   Vl1027         1027 TRUSTED    MAC-DOWN       N/A       LISP-DT-GUARD-VLAN 
 000c.2993.006c   Gi1/0/23       1026 NO TRUST   MAC-REACHABLE  269 s     IPDT_MAX_10 
 0000.0c9f.f85c   Vl2045         2045 TRUSTED    MAC-REACHABLE  N/A       LISP-DT-GUARD-VLAN 
 0000.0c9f.f463   Vl1028         1028 TRUSTED    MAC-DOWN       N/A       LISP-DT-GUARD-VLAN 
 0000.0c9f.f462   Vl1027         1027 TRUSTED    MAC-DOWN       N/A       LISP-DT-GUARD-VLAN 
 0000.0c9f.f461   Vl1026         1026 TRUSTED    MAC-REACHABLE  N/A       LISP-DT-GUARD-VLAN

// Edge2s IPDT mac table

Edge2#show device-tracking database mac                
 MAC              Interface      vlan prlvl      state          time left policy
 7486.0b05.7e7f   Vl2045         2045 TRUSTED    MAC-STALE      N/A       LISP-DT-GUARD-VLAN 
 7486.0b05.7e78   Vl1026         1026 TRUSTED    MAC-STALE      N/A       LISP-DT-GUARD-VLAN 
 7486.0b05.7e6e   Vl1028         1028 TRUSTED    MAC-DOWN       N/A       LISP-DT-GUARD-VLAN 
 7486.0b05.7e5c   Vl1027         1027 TRUSTED    MAC-DOWN       N/A       LISP-DT-GUARD-VLAN 
 380e.4d5b.0d40   Gi1/0/24       2045 NO TRUST   MAC-REACHABLE  290 s     LISP-DT-GUARD-VLAN 
 000c.29f2.c674   Gi1/0/6        1026 NO TRUST   MAC-REACHABLE  275 s     IPDT_MAX_10 
 0000.0c9f.f85c   Vl2045         2045 TRUSTED    MAC-REACHABLE  N/A       LISP-DT-GUARD-VLAN 
 0000.0c9f.f463   Vl1028         1028 TRUSTED    MAC-DOWN       N/A       LISP-DT-GUARD-VLAN 
 0000.0c9f.f462   Vl1027         1027 TRUSTED    MAC-DOWN       N/A       LISP-DT-GUARD-VLAN 
 0000.0c9f.f461   Vl1026         1026 TRUSTED    MAC-REACHABLE  N/A       LISP-DT-GUARD-VLAN
```

LISP should have picked these up dynamically as well, so we should see these addresses in the LISP database as well. The allocated instance-ID for IPv4 is 4100 and for Ethernet, it is 8195.

```
// Edge1 and Edge2s IPv4 LISP database

Edge1#show lisp instance-id 4100 ipv4 database 
LISP ETR IPv4 Mapping Database for EID-table vrf Corp_VN (IID 4100), LSBs: 0x1
Entries total 2, no-route 0, inactive 1

192.2.11.100/32, dynamic-eid SDA_Prod-IPV4, inherited from default locator-set rloc_70ac0729-86f6-44d0-a1c9-254b98507c24
  Locator       Pri/Wgt  Source     State
  192.2.101.70   10/10   cfg-intf   site-self, reachable 

Edge2#show lisp instance-id 4100 ipv4 database 
LISP ETR IPv4 Mapping Database for EID-table vrf Corp_VN (IID 4100), LSBs: 0x1
Entries total 2, no-route 0, inactive 1

192.2.11.200/32, dynamic-eid SDA_Prod-IPV4, inherited from default locator-set rloc_842654a1-cb84-42ec-9187-2e3a46790cc1
  Locator       Pri/Wgt  Source     State
  192.2.101.71   10/10   cfg-intf   site-self, reachable

// Edge1 and Edge2s Ethernet LISP database

Edge1#show lisp instance-id 8195 ethernet database 
LISP ETR MAC Mapping Database for EID-table Vlan 1026 (IID 8195), LSBs: 0x1
Entries total 1, no-route 0, inactive 0

000c.2993.006c/48, dynamic-eid Auto-L2-group-8195, inherited from default locator-set rloc_70ac0729-86f6-44d0-a1c9-254b98507c24
  Locator       Pri/Wgt  Source     State
  192.2.101.70   10/10   cfg-intf   site-self, reachable

Edge2#show lisp instance-id 8195 ethernet database
LISP ETR MAC Mapping Database for EID-table Vlan 1026 (IID 8195), LSBs: 0x1
Entries total 3, no-route 0, inactive 0

000c.29f2.c674/48, dynamic-eid Auto-L2-group-8195, inherited from default locator-set rloc_842654a1-cb84-42ec-9187-2e3a46790cc1
  Locator       Pri/Wgt  Source     State
  192.2.101.71   10/10   cfg-intf   site-self, reachable
```

Perfect. Since these are registered in the LISP database, we should see them in the site table on the control-plane as well. Let's quickly check Border1 for this:

```
// site table for IPv4

Border1#show lisp instance-id 4100 ipv4 server 
LISP Site Registration Information
* = Some locators are down or unreachable
# = Some registrations are sourced by reliable transport

Site Name      Last      Up     Who Last             Inst     EID Prefix
               Register         Registered           ID       
site_uci       never     no     --                   4100     0.0.0.0/0
               never     no     --                   4100     192.2.11.0/24
               15:12:30  yes#   192.2.101.70:14450   4100     192.2.11.100/32
               15:12:30  yes#   192.2.101.71:17807   4100     192.2.11.200/32

// site table for Ethernet

Border1#show lisp instance-id 8195 ethernet server 
LISP Site Registration Information
* = Some locators are down or unreachable
# = Some registrations are sourced by reliable transport

Site Name      Last      Up     Who Last             Inst     EID Prefix
               Register         Registered           ID       
site_uci       never     no     --                   8195     any-mac
               15:12:39  yes#   192.2.101.71:17807   8195     0000.0c9f.f461/48
               00:26:37  yes#   192.2.101.70:14450   8195     000c.2993.006c/48
               01:50:40  yes#   192.2.101.71:17807   8195     000c.29f2.c674/48
               15:12:39  yes#   192.2.101.71:17807   8195     7486.0b05.7e78/48
```

One last thing to verify before we start our testing - let's make sure the hosts do not have each others IP address resolved already. 

```
// Host1s ARP table

C:\Users\Host1>arp -a

Interface: 192.2.11.100 --- 0xb
  Internet Address      Physical Address      Type
  192.2.11.1            00-00-0c-9f-f4-61     dynamic
  192.2.11.255          ff-ff-ff-ff-ff-ff     static
  224.0.0.22            01-00-5e-00-00-16     static
  224.0.0.252           01-00-5e-00-00-fc     static

// Host2s ARP table

C:\Users\Host2>arp -a

Interface: 192.2.11.200 --- 0xb
  Internet Address      Physical Address      Type
  192.2.11.1            00-00-0c-9f-f4-61     dynamic
  192.2.11.255          ff-ff-ff-ff-ff-ff     static
  224.0.0.22            01-00-5e-00-00-16     static
  224.0.0.252           01-00-5e-00-00-fc     static
```

Only the gateway IPs are resolved and the hosts have no knowledge of each other. 

## ARP packet flow

Naturally, our test is very simple - we will ping from Host1 to Host2. 

```
C:\Users\admin-PC1>ping 192.2.11.200

Pinging 192.2.11.200 with 32 bytes of data:
Reply from 192.2.11.200: bytes=32 time=843ms TTL=128
Reply from 192.2.11.200: bytes=32 time=1ms TTL=128
Reply from 192.2.11.200: bytes=32 time=1ms TTL=128
Reply from 192.2.11.200: bytes=32 time=1ms TTL=128

Ping statistics for 192.2.11.200:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 5ms, Average = 1ms
```

Pings work too. Let's try to understand what happened here. 

Host1 generates an ARP request for Host2s IP address, since it knows it is in the same subnet. This hits Edge1. 

![static1](/static/images/cisco/sda_11/arp_2.jpg)

A packet capture on Gi1/0/23 on Edge1 confirms that the ARP is a broadcast as it comes into Edge1:

![static1](/static/images/cisco/sda_11/arp_3.jpg)

Here's where things get interesting. The ARP will not be flooded immediately. Edge1 will first send out a map-request for the IP address that is present in the "Target IP address" of the ARP packet:

![static1](/static/images/cisco/sda_11/arp_4.jpg)

The control-plane nodes maintain an address-resolution table which has IP-mac mapping entries for all hosts in the fabric. This is similar to how BGP EVPN, in a VXLAN fabric, maintains an EVPN ARP suppression table. 

You can view this table using the following command:

```
Border1#show lisp instance-id 8195 ethernet server address-resolution 

Address-resolution data for router lisp 0 instance-id 8195

L3 InstID    Host Address                                Hardware Address
     4100    192.2.11.100/32                             000c.2993.006c
     4100    192.2.11.200/32                             000c.29f2.c674
     4100    FE80::1D22:B4A3:26A2:B064/128               000c.2993.006c
     4100    FE80::4CEA:EE0:9E2D:B08/128                 000c.29f2.c674
```

When this map-request (generated from an ARP request) comes to the control-plane (Border1, in this case), it looks at this table for a hit. If there is a hit, it sends a map-reply with the mac address of the host. 

![static1](/static/images/cisco/sda_11/arp_5.jpg)

Wireshark doesn't have the proper dissectors to  understand this packet fully but here's the map-reply anyway:

![static1](/static/images/cisco/sda_11/arp_6.jpg)

When Edge1 gets this, it builds another map-request - this time, for the mac address it just got from the control-plane. This leads to a second round of map-request/map-reply. When Border1 gets the map-request, it looks at the Ethernet site table (since the request is for a mac address) and responds back with the RLOC associated with this mac (which is Edge2). 

![static1](/static/images/cisco/sda_11/arp_7.jpg)

Inline, this looks like so:

![static1](/static/images/cisco/sda_11/arp_8.jpg)

Border1 replies back with the RLOC:

![static1](/static/images/cisco/sda_11/arp_9.jpg)

Inline, this looks like so:

![static1](/static/images/cisco/sda_11/arp_10.jpg)

At this point, Edge1 should have its LISP map-cache built as well:

```
Edge1#show lisp instance-id 8195 ethernet map-cache detail 
LISP MAC Mapping Cache for EID-table Vlan 1026 (IID 8195), 1 entries

000c.29f2.c674/48, uptime: 03:57:51, expires: 20:02:08, via map-reply, complete
  Sources: map-reply
  State: complete, last modified: 03:57:51, map-source: 192.2.101.71
  Idle, Packets out: 0(0 bytes)
  Encapsulating dynamic-EID traffic
  Locator       Uptime    State      Pri/Wgt     Encap-IID
  192.2.101.71  03:57:51  up          10/10        -
    Last up-down state change:         03:57:51, state change count: 1
    Last route reachability change:    03:57:51, state change count: 1
    Last priority / weight change:     never/never
    RLOC-probing loc-status algorithm:
      Last RLOC-probe sent:            03:57:51 (rtt 2ms)
```

Remember to look at the Ethernet instance-ID and not the IPv4 one. What happens now? The ARP is converted into a unicast message on Edge1 and is encapsulated with a destination IP of Edge2:

![static1](/static/images/cisco/sda_11/arp_11.jpg)

Inline, this looks like this:

![static1](/static/images/cisco/sda_11/arp_12.jpg)

Notice how the ARP is no longer a broadcast - it has been converted into a unicast ARP on Edge1 and encapsulated with a VXLAN header with a VNID of the Ethernet service (for that VLAN). The destination IP is of Loopback0 of Edge2. 

Once Edge2 gets this, it decapsulates the packet and unicasts it to Host2. This entire process now happens in reverse when Host2 sends an ARP reply in response to the ARP request. 

So, one final question to close this post out - when do we really use some form of replication for ARP? Naturally, when there is no hit in the address-resolution table of the control-plane. This also implies that this is perfect for silent hosts and this is exactly where L2 flooding comes into the picture. 

With L2 flooding (and an underlay multicast infrastructure), we can flood ARPs through the fabric and force a silent host to respond, thus bringing it alive in the fabric. L2 flooding should be another post on its own so we'll get to it some day!

Thanks for reading, I hope this was informative! Till the next one, hopefully in better times (yes, this quarantine/social-distancing sucks but it is necessary). 