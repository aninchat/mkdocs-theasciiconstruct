---
date:
  created: 2022-01-02
  updated: 2022-01-02
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
# Cisco SDA Part VIII - DHCP challenges in SDA
In this post, we look at various DHCP challenges in Cisco's SD-Access fabric and how it is solved.

## Introduction and topology

Remember that in SD-Access, we do not use vanilla LISP. To achieve macro segmentation, multi-instance LISP (VRF-aware LISP) is used. However, this poses a problem for DHCP. Consider the following topology for this (this topology is also a simple example of SD-Access design):

![static1](/static/images/cisco/sda_8/dhcp_1.jpg)

<!-- more -->

## Tracing the DHCP Discover

Host1 is in VLAN 1029 and it sends a DHCP discover. This hits interface VLAN 1029, which has an IP helper-address that points to 172.168.1.1 (which is the DHCP server). The full configuration of this interface VLAN is:

```
Edge1#show run int vlan 1029
Building configuration...

Current configuration : 314 bytes
!
interface Vlan1029
 description Configured from Cisco DNA-Center
 mac-address 0000.0c9f.f464
 vrf forwarding Campus_VN
 ip address 176.169.71.1 255.255.255.0
 ip helper-address 172.168.1.1
 no ip redirects
 ip route-cache same-interface
 no lisp mobility liveness test
 lisp mobility 176_169_71_0-Campus_VN
end
```

However, this interface VLAN is in VRF 'Campus_VN'. Inside the routing table for this VRF, does a route to the DHCP server exist? 

```
Edge1#show ip route vrf Campus_VN 172.168.1.1 

Routing Table: Campus_VN
% Network not in table
```

So, there's no route to the DHCP server. Why would we ever look at the VRF routing table and not invoke LISP in the first place?  Think back to what a DHCP discover is like. The source IP address of the DHCP discover is 0.0.0.0 while the destination IP address is 255.255.255.255. 

Remember the major LISP rules? Well, we fail the source EID check right here because 0.0.0.0 will never match a known source EID. This means that we cannot invoke LISP and fall back to normal routing, which implies a lookup in the respective routing table (in this case, for VRF 'Campus_VN'). With no route to the DHCP server in this RIB, the packet is dropped and the host will never get an IP address. 

Do you see the problem that multi-instance LISP presents now? The fix is very, very simple but it is one that perplexed me for a while - I could see the fix (or the configuration rather) but I did not know what it fixes or why that configuration existed in the first place. In hindsight of course, everything makes perfect sense.

For those who want to take a stab at the configuration that fixes this issue, stop now! For those who are already tired of reading this old man's drivel, the answer is to configure the edges as proxy-iTRs. Why? Because proxy-iTRs do not perform source EID checks. That means the DHCP discover gets elevated into the overlay since LISP will get invoked:

```
// DHCP Discover hits this entry in CEF

Edge1#show ip cef vrf Campus_VN 172.168.1.1 
0.0.0.0/0
  attached to LISP0.4100

// detailed output of the same CEF entry  

Edge1#show ip cef vrf Campus_VN 172.168.1.1 detail
0.0.0.0/0, epoch 0, flags [cover dependents, check lisp eligibility, default route]
  LISP remote EID: 2 packets 903 bytes fwd action signal, cfg as EID space
  LISP source path list
    attached to LISP0.4100
  Covered dependent prefixes: 2
    notify cover updated: 2
  1 IPL source [no flags]
  attached to LISP0.4100

// since LISP is invoked, we look at the map-cache
// no specific entry here, so it hits the default 0.0.0.0/0 entry  

Edge1#show ip lisp eid-table vrf Campus_VN map-cache 172.168.1.1
LISP IPv4 Mapping Cache for EID-table vrf Campus_VN (IID 4100), 3 entries

0.0.0.0/0, uptime: 00:35:21, expires: never, via static-send-map-request
  Sources: static-send-map-request
  State: send-map-request, last modified: 00:35:21, map-source: local
  Exempt, Packets out: 2(903 bytes) (~ 00:29:50 ago)
  Configured as EID address space
  Negative cache entry, action: send-map-request 
```

### DHCP server known to LISP

At this point, one of two things can happen: 

1. The route to the DHCP server is redistributed into the LISP site database and the Border knows of this. If that is the case, the Border replies back to the map request with its own RLOC information and the packet is encapsulated to it. 

![static1](/static/images/cisco/sda_8/dhcp_2.jpg)

### DHCP server unknown to LISP

2. The Border has no idea of the EID and there is a miss in the site database. In that case, it responds with a negative map reply and the Edge will encapsulate the packet and send it to the Border anyway (since 'use-petr' will be configured as the Border). 

![static1](/static/images/cisco/sda_8/dhcp_3.jpg)

Both are valid designs and are okay to run with. Either of them gets the packet to the border from where it is routed to the DHCP server.

![static1](/static/images/cisco/sda_8/dhcp_4.jpg)

## Tracing the DHCP Offer and the problem it presents

The DHCP server gets this and uses the 'giaddr' to determine which pool to allocate the address from (your traditional DHCP functionality, nothing magical here). It sends an offer back - the offer will have a source IP address of the DHCP server and the destination IP address of the 'giaddr' address.

And here's the second major problem that DHCP needs to deal with in SD-Access - this 'giaddr' address is essentially the interface VLANs IP address, which (as you might have guessed it) is the anycast gateway IP address across all Edge's. Essentially, the same VLAN and IP address mapping will exist across all Edge's. 

![static1](/static/images/cisco/sda_8/dhcp_5.jpg)

And here is the problem - how can the DHCP offer be directed to the Edge that actually sourced the DHCP discover? 

![static1](/static/images/cisco/sda_8/dhcp_6.jpg)

### DHCP option82

This is where option82 comes in. When the DHCP discover is sourced by the Edge, it fills in the option82 field as well. This contains two important things - its RLOC and the VXLAN-ID (VNID) to be used. Take a look at the following packet capture:

![static1](/static/images/cisco/sda_8/dhcp_7.jpg)

This is a DHCP discover sourced by our Edge in this topology (Edge1). The discover is built with the relevant options (including option82). The inner IP header has a source IP address of interface VLAN 1029s and a destination IP address of the DHCP server. 

The VXLAN header has a VNI that maps to the instance ID for this VRF. The outer IP header has a source IP address of loopback0 of this Edge and a destination IP address of loopback0 of the Border - essentially the source RLOC and the destination RLOC.

Let's try decoding agent remote ID of option 82:

![static1](/static/images/cisco/sda_8/dhcp_8.jpg)

So, when this offer hits the Border, it is leaked to the CPU and this RLOC + VNI information is extracted from option82. The border then directs this towards the specified RLOC - 192.168.70.8, which is the Edge that sourced the discover in the first place. 

This concludes a very important aspect of SD-Access - one that should work seamlessly and without notice but something that needs to be very carefully understood.