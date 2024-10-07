---
date:
  created: 2021-12-14
  updated: 2021-12-14
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
# Cumulus Part IX - Understanding VXLAN EVPN Route-Target control

In this post, we look at how route-targets extended communities can be used to control VXLAN BGP EVPN routes in Cumulus Linux.

## Introduction 

This post assumes that the reader has a general understanding of L2/L3 VNIs and asymmetric/symmetric IRB.

Cumulus, by default, uses auto RTs for L2 and L3 VNIs. This makes for a very easy experience (almost plug and play like) when building VXLAN BGP EVPN fabrics. But it also doesn't help you understand much of how route-targets are being imported across and how to completely control this. 

It's always good to learn to drive a stick, before moving to an automatic. So, the goal of this blog is to help understand how L2/L3 VNI RTs are exported/imported and how you can control what goes into your customer VRF.  This will also allow you to control whether you want asymmetric or symmetric IRB (see my previous Cumulus blog posts to understand what is asymmetric and symmetric IRB).

<!-- more -->

## Topology

For this post, we're going to use the following topology and incrementally add/delete/modify certain aspects of this network. 

![rt1](/static/images/cumulus/cumulus_part9/cumulus_RTs_1.jpg)

## RTs in an L2VNI environment

To begin with, this is a pure L2 VNI setup with two servers (Server1 and Server2) deployed in VLAN 10, mapped to VNI 10010. The network is a BGP unnumbered core, with loopbacks of each leaf acting as the VXLAN tunnel IPs. 


Let's review the BGP configuration:

```
router bgp 64521
  bgp router-id 1.1.1.1
  neighbor swp1 interface remote-as external
  neighbor swp2 interface remote-as external
  
  address-family ipv4 unicast
    network 1.1.1.1/32 
  
  address-family l2vpn evpn
    neighbor swp1 activate
    neighbor swp2 activate
    advertise-all-vni
```


Typical BGP configuration - we're advertising all VNIs into EVPN and the BGP L2VPN EVPN peering is activated against both Spine1 and Spine2. By default, Cumulus Linux (FRR, really) uses a model of ASN:VNI to derive the VNI RTs. 


In our case, this will be 64521:10010 for VNI 10010. We can confirm using the following:

```
cumulus@Leaf1:mgmt:~$ net show bgp l2vpn evpn vni 10010
VNI: 10010 (known to the kernel)
  Type: L2
  Tenant-Vrf: default
  RD: 1.1.1.1:2
  Originator IP: 1.1.1.1
  Mcast group: 0.0.0.0
  Advertise-gw-macip : Disabled
  Advertise-svi-macip : Disabled
  Import Route Target:
    64521:10010
  Export Route Target:
    64521:10010
```

PC1s mac address is advertised as a type-2 route using this export RT:

```
cumulus@Leaf1:mgmt:~$ net show bgp l2vpn evpn route rd 1.1.1.1:2 mac 00:50:79:66:68:06 
BGP routing table entry for 1.1.1.1:2:[2]:[00:50:79:66:68:06]/352
Paths: (1 available, best #1)
  Advertised to non peer-group peers:
  Spine1(swp1) Spine2(swp2)
  Route [2]:[0]:[48]:[00:50:79:66:68:06] VNI 10010
  Local
    1.1.1.1 from 0.0.0.0 (1.1.1.1)
      Origin IGP, weight 32768, valid, sourced, local, bestpath-from-AS Local, best (First path received)
      Extended Community: ET:8 RT:64521:10010
      Last update: Sat Jul 24 16:30:57 2021
```

This is correctly imported on Leaf2. Cumulus does not show the route imported into the local RD in the BGP table, however, bgpd informs zebra and zebra has installed it in the MAC address table.

```
cumulus@Leaf2:mgmt:~$ net show bgp l2vpn evpn route rd 1.1.1.1:2 mac 00:50:79:66:68:06
BGP routing table entry for 1.1.1.1:2:[2]:[00:50:79:66:68:06]/352
Paths: (2 available, best #1)
  Advertised to non peer-group peers:
  Spine1(swp1) Spine2(swp2)
  Route [2]:[0]:[48]:[00:50:79:66:68:06] VNI 10010
  65550 64521
    1.1.1.1 from Spine1(swp1) (11.11.11.11)
      Origin IGP, valid, external, bestpath-from-AS 65550, best (Router ID)
      Extended Community: RT:64521:10010 ET:8
      Last update: Sat Jul 24 16:30:59 2021
  Route [2]:[0]:[48]:[00:50:79:66:68:06] VNI 10010
  65550 64521
    1.1.1.1 from Spine2(swp2) (22.22.22.22)
      Origin IGP, valid, external
      Extended Community: RT:64521:10010 ET:8
      Last update: Sat Jul 24 16:30:59 2021
```

The MAC address table also shows this entry, against a remote VTEP of 1.1.1.1, which is Leaf1. 

```
cumulus@Leaf2:mgmt:~$ net show bridge macs 00:50:79:66:68:06

VLAN      Master  Interface  MAC                TunnelDest  State  Flags               LastSeen
--------  ------  ---------  -----------------  ----------  -----  ------------------  --------
10        bridge  vni10      00:50:79:66:68:06                     extern_learn        00:00:09
untagged          vni10      00:50:79:66:68:06  1.1.1.1            self, extern_learn  00:00:09
```

### Adding manual RTs

Let's add a manual export RT for VNI 10010, on Leaf1:

```
cumulus@Leaf1:mgmt:~$ net add bgp l2vpn evpn vni 10010 route-target export 1:10010
cumulus@Leaf1:mgmt:~$ net commit
```


This is correctly added to the prefix that is being advertised via BGP EVPN:

```
cumulus@Leaf1:mgmt:~$ net show bgp l2vpn evpn route type 2
BGP table version is 9, local router ID is 1.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-1 prefix: [1]:[ESI]:[EthTag]:[IPlen]:[VTEP-IP]:[Frag-id]
EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
                    Extended Community
Route Distinguisher: 1.1.1.1:2
*> [2]:[0]:[48]:[00:50:79:66:68:06]
                    1.1.1.1                            32768 i
                    ET:8 RT:1:10010

Displayed 1 prefixes (1 paths) (of requested type)
```

Leaf2 is still importing this though - why, and how?

```
// BGP EVPN table
 
cumulus@Leaf2:mgmt:~$ net show bgp l2vpn evpn route type 2 
BGP table version is 9, local router ID is 2.2.2.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-1 prefix: [1]:[ESI]:[EthTag]:[IPlen]:[VTEP-IP]:[Frag-id]
EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
                    Extended Community
Route Distinguisher: 1.1.1.1:2
*> [2]:[0]:[48]:[00:50:79:66:68:06]
                    1.1.1.1                                0 65550 64521 i
                    RT:1:10010 ET:8
*  [2]:[0]:[48]:[00:50:79:66:68:06]
                    1.1.1.1                                0 65550 64521 i
                    RT:1:10010 ET:8

Displayed 1 prefixes (2 paths) (of requested type) 

// MAC address table

cumulus@Leaf2:mgmt:~$ net show bridge macs 00:50:79:66:68:06

VLAN      Master  Interface  MAC                TunnelDest  State  Flags               LastSeen
--------  ------  ---------  -----------------  ----------  -----  ------------------  --------
10        bridge  vni10      00:50:79:66:68:06                     extern_learn        00:03:14
untagged          vni10      00:50:79:66:68:06  1.1.1.1            self, extern_learn  00:03:14
```

This is the first important thing to remember with auto-derived RTs on Cumulus Linux - there is an implicit *:VNI import when using auto-RTs. This is necessary because when you follow an eBGP peering model, the AS numbers will naturally be different and a ASN:VNI import model will not work when using your own ASN for the import RT. 


Let's add a manual, incorrect RT now on Leaf2:

```
cumulus@Leaf2:mgmt:~$ net add bgp l2vpn evpn vni 10010 route-target import 1:10
cumulus@Leaf2:mgmt:~$ net commit
```


We no longer see the entry in the MAC address table anymore, even though BGP EVPN has received it:

```
cumulus@Leaf2:mgmt:~$ net show bridge macs 00:50:79:66:68:06                   

VLAN  Master  Interface  MAC  TunnelDest  State  Flags  LastSeen
----  ------  ---------  ---  ----------  -----  -----  --------
* no output *

cumulus@Leaf2:mgmt:~$ net show bgp l2vpn evpn route type 2 
BGP table version is 11, local router ID is 2.2.2.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-1 prefix: [1]:[ESI]:[EthTag]:[IPlen]:[VTEP-IP]:[Frag-id]
EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
                    Extended Community
Route Distinguisher: 1.1.1.1:2
*> [2]:[0]:[48]:[00:50:79:66:68:06]
                    1.1.1.1                                0 65550 64521 i
                    RT:1:10010 ET:8
*  [2]:[0]:[48]:[00:50:79:66:68:06]
                    1.1.1.1                                0 65550 64521 i
                    RT:1:10010 ET:8

Displayed 1 prefixes (2 paths) (of requested type) 
```


Once we add the correct RT to be imported, we see it in the mac table again:

```
cumulus@Leaf2:mgmt:~$ net add bgp l2vpn evpn vni 10010 route-target import 1:10010
cumulus@Leaf2:mgmt:~$ net commit

cumulus@Leaf2:mgmt:~$ net show bridge macs 00:50:79:66:68:06

VLAN      Master  Interface  MAC                TunnelDest  State  Flags               LastSeen
--------  ------  ---------  -----------------  ----------  -----  ------------------  --------
10        bridge  vni10      00:50:79:66:68:06                     extern_learn        00:00:17
untagged          vni10      00:50:79:66:68:06  1.1.1.1            self, extern_learn  00:00:17 
```

Remember, this import RT also controls what is inserted into the EVPN ARP cache. Assuming corresponding SVIs were deployed as well, you should see the EVPN ARP cache populated with this entry if the correct import RTs are configured (via the type-2 MAC plus IP route).

```
cumulus@Leaf2:mgmt:~$ net show evpn arp-cache vni 10010
Number of ARPs (local and remote) known for this VNI: 2
Flags: I=local-inactive, P=peer-active, X=peer-proxy
Neighbor        Type   Flags State    MAC               Remote ES/VTEP                 Seq #'s
10.10.10.101    remote       active   00:50:79:66:68:06 1.1.1.1                        0/0
10.10.10.102    local        active   00:50:79:66:68:07                                0/0  
```

## RTs with symmetric IRB

Let's now convert our network to a symmetric IRB topology. Server2 is moved to VLAN 20, with an IP address of 20.20.20.102. VLANs 10 and 20 are present on both Leaf1 and Leaf2, acting as anycast gateways for their respective subnets. 

![rt2](/static/images/cumulus/cumulus_part9/cumulus_RTs_2.jpg)

Both Leaf1 and Leaf2 have VNIs created for VLANs 10 and 20. Example below from Leaf1:

```
interface vni10
  bridge-access 10
  mstpctl-bpduguard yes
  mstpctl-portbpdufilter yes
  vxlan-id 10010
  vxlan-local-tunnelip 1.1.1.1

interface vni20
  bridge-access 20
  mstpctl-bpduguard yes
  mstpctl-portbpdufilter yes
  vxlan-id 10020
  vxlan-local-tunnelip 1.1.1.1
```

A L3VNI (VNI 10040) is created for symmetric routing and mapped to VLAN 40. Each of the servers are moved into a new VRF, called VRF1. The L3VNI is mapped to this VRF as well. 

```
interface vlan10
  address 10.10.10.1/24
  hwaddress 00:10:00:10:00:10
  vlan-id 10
  vlan-raw-device bridge
  vrf VRF1

interface vlan20
  address 20.20.20.1/24
  hwaddress 00:20:00:20:00:20
  vlan-id 20
  vlan-raw-device bridge
  vrf VRF1

interface vlan40
  vlan-id 40
  vlan-raw-device bridge
  vrf VRF1
```

The same import RT logic applies for L3VNIs also - if there's no manual import configured for the L3VNI, then the default *:VNI import is applied. It is crucial to understand how the import/export RTs for the L3VNI is controlled - this is done under the VRF specific address-family in BGP. 


For example, before setting any manual import/export RTs, the L3VNI has auto-derived it:


```
cumulus@Leaf1:mgmt:~$ net show bgp l2vpn evpn vni 10040
VNI: 10040 (known to the kernel)
  Type: L3
  Tenant VRF: VRF1
  RD: 20.20.20.1:3
  Originator IP: 1.1.1.1
  Advertise-gw-macip : n/a
  Advertise-svi-macip : n/a
  Advertise-pip: Yes
  System-IP: 1.1.1.1
  System-MAC: 50:00:00:03:00:03
  Router-MAC: 50:00:00:03:00:03
  Import Route Target:
    64521:10040
  Export Route Target:
    64521:10040 
```


We'll now configure this manually instead, as an example, on Leaf1. This goes under the BGP configuration itself:

```
net add bgp vrf VRF1 autonomous-system 64521
net add bgp vrf VRF1 l2vpn evpn route-target import 2:10040
net add bgp vrf VRF1 l2vpn evpn route-target export 1:10040
```

Notice how these are specific to the VRF. Now, Leaf1 should be adding a RT of 1:10040 to the prefixes.  The final BGP configuration in this case:

```
router bgp 64521
  bgp router-id 1.1.1.1
  neighbor swp1 interface remote-as external
  neighbor swp2 interface remote-as external
  
  address-family ipv4 unicast
    network 1.1.1.1/32 
  
  address-family l2vpn evpn
    neighbor swp1 activate
    neighbor swp2 activate
    advertise-all-vni
    
    vni 10020
      route-target import 2:10
      route-target export 2:10
    
    vni 10010
      route-target import 1:10
      route-target export 1:10

router bgp 64521 vrf VRF1
  
  address-family l2vpn evpn
    route-target export 1:10040
    route-target import 2:10040
```


Looking at the BGP EVPN table, we can see the RTs are correctly added:

```
cumulus@Leaf1:mgmt:~$ net show bgp l2vpn evpn route rd 1.1.1.1:3 type 2
EVPN type-1 prefix: [1]:[ESI]:[EthTag]:[IPlen]:[VTEP-IP]:[Frag-id]
EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]

BGP routing table entry for 1.1.1.1:3:[2]:[00:50:79:66:68:06]/352
Paths: (1 available, best #1)
  Advertised to non peer-group peers:
  Spine1(swp1) Spine2(swp2)
  Route [2]:[0]:[48]:[00:50:79:66:68:06] VNI 10010/10040
  Local
    1.1.1.1 from 0.0.0.0 (1.1.1.1)
      Origin IGP, weight 32768, valid, sourced, local, bestpath-from-AS Local, best (First path received)
      Extended Community: ET:8 RT:1:10 RT:1:10040 Rmac:50:00:00:03:00:03
      Last update: Fri Jul 30 09:43:56 2021
BGP routing table entry for 1.1.1.1:3:[2]:[00:50:79:66:68:06]:[10.10.10.101]/352
Paths: (1 available, best #1)
  Advertised to non peer-group peers:
  Spine1(swp1) Spine2(swp2)
  Route [2]:[0]:[48]:[00:50:79:66:68:06]:[32]:[10.10.10.101] VNI 10010/10040
  Local
    1.1.1.1 from 0.0.0.0 (1.1.1.1)
      Origin IGP, weight 32768, valid, sourced, local, bestpath-from-AS Local, best (First path received)
      Extended Community: ET:8 RT:1:10 RT:1:10040 Rmac:50:00:00:03:00:03
      Last update: Fri Jul 30 09:43:56 2021

Displayed 2 prefixes (2 paths) with this RD (of requested type) 
```

There are two distinct RTs added here - one for the corresponding L2VNI and another for the L3VNI. 


It is important to understand the impact of importing each RT - on Leaf2, importing the RT for the L3VNI is what imports the /32 route (from the type-2 MAC plus IP route) into the VRF routing table, while importing the L2VNI will pull the MAC address into the MAC address table (this is done using the type-2 MAC only route) and create an entry in the EVPN ARP cache (using the type-2 MAC plus IP route).


Let's confirm on Leaf2:

```
// MAC address table

cumulus@Leaf2:mgmt:~$ net show bridge macs 00:50:79:66:68:06

VLAN      Master  Interface  MAC                TunnelDest  State  Flags               LastSeen
--------  ------  ---------  -----------------  ----------  -----  ------------------  --------
10        bridge  vni10      00:50:79:66:68:06                     extern_learn        00:00:43
untagged          vni10      00:50:79:66:68:06  1.1.1.1            self, extern_learn  00:00:43

// EVPN ARP cache

cumulus@Leaf2:mgmt:~$ net show evpn arp-cache vni 10010     
Number of ARPs (local and remote) known for this VNI: 1
Flags: I=local-inactive, P=peer-active, X=peer-proxy
Neighbor        Type   Flags State    MAC               Remote ES/VTEP                 Seq #'s
10.10.10.101    remote       active   00:50:79:66:68:06 1.1.1.1                        0/0 

// VRF1 route table

cumulus@Leaf2:mgmt:~$ net show route vrf VRF1 ipv4          
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

VRF VRF1:
K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 01:46:41
C>* 10.10.10.0/24 is directly connected, vlan10, 01:46:41
B>* 10.10.10.101/32 [20/0] via 1.1.1.1, vlan40 onlink, weight 1, 00:21:36
C>* 20.20.20.0/24 is directly connected, vlan20, 01:46:41 
```


On Leaf2, as you can see, we have enough information to route both asymmetrically and symmetrically. Remember, the lookup is always a longest prefix match - which means the /32 route is hit and the packet is routed symmetrically. 


## Controlling RTs

Knowing what we know of RT import/export, we can fully control how we want our traffic to flow (asymmetric or symmetric).


On Leaf1, let's import a different RT under the VRF address-family.

```
router bgp 64521
  bgp router-id 1.1.1.1
  neighbor swp1 interface remote-as external
  neighbor swp2 interface remote-as external
  
  address-family ipv4 unicast
    network 1.1.1.1/32 
  
  address-family l2vpn evpn
    neighbor swp1 activate
    neighbor swp2 activate
    advertise-all-vni
    
    vni 10020
      route-target import 2:10
      route-target export 2:10
    
    vni 10010
      route-target import 1:10
      route-target export 1:10

router bgp 64521 vrf VRF1
  
  address-family l2vpn evpn
    route-target export 1:10040
    route-target import 2:10041 
```

This causes the type-2 MAC plus IP route to not be imported as a /32 route in the VRF table. Now, the longest prefix match is the subnet route itself:


```
cumulus@Leaf1:mgmt:~$ net show route vrf VRF1 ipv4
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

VRF VRF1:
K>* 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 07:27:56
C>* 10.10.10.0/24 is directly connected, vlan10, 07:27:56
C>* 20.20.20.0/24 is directly connected, vlan20, 07:27:56 
```

This means that the destination is directly connected to Leaf1 and it can ARP for it. Using the EVPN ARP cache, Leaf1 already knows PC2s mac address, and there's no need to ARP for it again. Thus, PC1 to PC2, traffic will flow asymmetrically. A packet capture confirms that the VNI added to the VXLAN header is 10020.

![rt3](/static/images/cumulus/cumulus_part9/cumulus_RTs_3.jpg)


The return path is symmetric because Leaf2 still has that /32 entry imported into the VRF table. A packet capture confirms that the return packet has the L3VNI (10040) added to the VXLAN header. 

![rt4](/static/images/cumulus/cumulus_part9/cumulus_RTs_4.jpg)

I hope this was informative, and I'll see you in the next one.