---
date:
  created: 2022-12-18
  updated: 2022-12-18
authors:
    - aninchat
blog_toc: true
categories:
  - containerlab
  - junos
  - bgp
  - evpn
  - vxlan
  - cisco
  - nxos
  - multivendor
tags:
  - containerlab
  - junos
  - bgp
  - evpn
  - vxlan
  - cisco
  - nxos
  - multivendor
---
# Cisco's NXOS EVPN Hybrid mode - interoperability with Asymmetric VTEPs
In this post, we pull back the curtain on this new NXOS EVPN Hybrid mode and understand how it enables interoperability with Asymmetric VTEPs.

## tl;dr

Cisco gave this a fancy name - EVPN Hybrid mode. Or you know, maybe just call it Asymmetric IRB? Because that's what it is behind the scenes.

## Topology

![topology](/static/images/multivendor/nxos_evpn_hybrid_mode/topology.jpg)

<!-- more -->

## Understanding why Asymmetric IRB does not work on NXOS

For this, let's consider communication between h1 and h4. We'll configure leaf1 and leaf3 for Asymmetric IRB - this implies that all VLANs, VNIs and IRB interfaces exist on both leafs. In our case, this means VLAN 100, VNI 10100, VLAN 200, VNI 10200 and its respective IRB interfaces are configured on both leaf1 and leaf3. 

### Configuration

For leaf1, which is running NXOS, the relevant configuration is as follows:

```
nv overlay evpn
feature bgp
feature fabric forwarding
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
!
fabric forwarding anycast-gateway-mac 000a.000a.000a
vlan 1,100,200,300
vlan 100
  vn-segment 10100
vlan 200
  vn-segment 10200
!
route-map allow-loopback permit 10
  match interface loopback0 
!
interface Vlan100
  no shutdown
  ip address 10.100.100.254/24
  fabric forwarding mode anycast-gateway
!
interface Vlan200
  no shutdown
  ip address 10.100.200.254/24
  fabric forwarding mode anycast-gateway
!
interface nve1
  no shutdown
  host-reachability protocol bgp
  advertise virtual-rmac
  source-interface loopback0
  member vni 10100
    ingress-replication protocol bgp
  member vni 10200
    ingress-replication protocol bgp
!
interface loopback0
  ip address 192.0.2.1/32
!
router bgp 65421
  router-id 192.0.2.1
  log-neighbor-changes
  address-family ipv4 unicast
    redistribute direct route-map allow-loopback
    maximum-paths 4
  address-family l2vpn evpn
    advertise-pip
  template peer evpn
    update-source loopback0
    ebgp-multihop 2
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 192.0.2.11
    inherit peer evpn
    remote-as 65500
  neighbor 192.0.2.22
    inherit peer evpn
    remote-as 65500
  neighbor 198.51.100.1
    remote-as 65500
    address-family ipv4 unicast
  neighbor 198.51.100.9
    remote-as 65500
    address-family ipv4 unicast
evpn
  vni 10100 l2
    rd 192.0.2.1:100
    route-target import 100:100
    route-target export 100:100
  vni 10200 l2
    rd 192.0.2.1:200
    route-target import 200:200
    route-target export 200:200
```

On leaf3, which is running Junos OS, we create routing-instances of type mac-vrf - one routing-instance per VLAN since we're enabling it for VLAN-based service type. 

```
admin@leaf3# show 

interfaces {
    irb {
        unit 100 {
            virtual-gateway-accept-data;
            family inet {
                address 10.100.100.252/24 {
                    virtual-gateway-address 10.100.100.254;
                }
            }
            virtual-gateway-v4-mac 00:0a:00:0a:00:0a;
        }
        unit 200 {
            virtual-gateway-accept-data;
            family inet {
                address 10.100.200.252/24 {
                    virtual-gateway-address 10.100.200.254;
                }
            }
            virtual-gateway-v4-mac 00:0a:00:0a:00:0a;
        }
    }
    lo0 {
        unit 0 {
            family inet {
                address 192.0.2.3/32;
            }
        }
    }
}
policy-options {
    policy-statement ECMP {
        then {
            load-balance per-flow;
        }
    }
    policy-statement allow-loopback {
        from interface lo0.0;
        then accept;
    }
}
routing-instances {
    v100_mac_vrf {
        instance-type mac-vrf;
        protocols {
            evpn {
                encapsulation vxlan;    
                default-gateway do-not-advertise;
                extended-vni-list all;
            }
        }
        vtep-source-interface lo0.0;
        service-type vlan-based;
        interface xe-0/0/2.0;
        route-distinguisher 192.0.2.3:100;
        vrf-target target:100:100;
        vlans {
            v100 {
                vlan-id 100;
                l3-interface irb.100;
                vxlan {
                    vni 10100;
                }
            }
        }
    }
    v200_mac_vrf {
        instance-type mac-vrf;
        protocols {
            evpn {
                encapsulation vxlan;
                default-gateway do-not-advertise;
                extended-vni-list all;
            }
        }
        vtep-source-interface lo0.0;
        service-type vlan-based;
        interface xe-0/0/3.0;
        route-distinguisher 192.0.2.3:200;
        vrf-target target:200:200;
        vlans {
            v200 {
                vlan-id 200;
                l3-interface irb.200;
                vxlan {
                    vni 10200;
                }
            }
        }
    }
}
routing-options {
    router-id 192.0.2.3;
    autonomous-system 65423;            
    forwarding-table {
        export ECMP;
    }
}
protocols {
    bgp {
        group underlay {
            type external;
            family inet {
                unicast;
            }
            export allow-loopback;
            peer-as 65500;
            multipath;
            neighbor 198.51.100.5;
            neighbor 198.51.100.13;
        }
        group overlay {
            type external;
            multihop;
            local-address 192.0.2.3;
            family evpn {
                signaling;
            }
            peer-as 65500;
            neighbor 192.0.2.11;
            neighbor 192.0.2.22;
        }
    }
}
```

Since each leaf is configured with all IRBs, every VLAN is essentially directly connected to each leaf. This implies that both leaf1 and leaf3 can directly ARP for any host on VLAN100 or VLAN200.

### Packet walk to visualize the problem

h1 has 10.100.100.254 as its default gateway. When pinging h4, it knows that the destination is in a different subnet and it needs to send the packet to its default gateway. ARP process provides the IP-MAC binding for 10.100.100.254 and h1 sends the ICMP request to leaf1.

On leaf1, since the destination MAC address is owned by it, a route lookup is done for 10.100.200.4. This hits the directly connected subnet route:

```
leaf1# show ip route 10.100.200.4
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.100.200.0/24, ubest/mbest: 1/0, attached
    *via 10.100.200.254, Vlan200, [0/0], 00:29:41, direct
```

leaf1 can now ARP for the destination directly. Since this fabric is configured for Ingress Replication (default on Junos OS is Ingress Replication and we have explictly configured Ingress Replication on NXOS), leaf1 uses its flood list for VLAN 200 to package the broadcast ARP into a unicast VXLAN packet and send it to every VTEP (PE) in the flood list. This includes leaf3.

When leaf3 receives this, it decapsulates the VXLAN packet and floods the ARP to all local ports in VLAN 200. h4 now receives this ARP packet, it builds its local ARP cache and sends an ARP reply back.

Let's take a look at this ARP reply:

![arp_reply_h4](/static/images/multivendor/nxos_evpn_hybrid_mode/arp_reply_h4.jpg)

Seems like an ordinary ARP reply, but carefully observe the destination MAC address in the Ethernet header. This is the anycast gateway MAC address. When this ARP reply reaches leaf3, it will consume this packet since it owns the anycast MAC address and leaf1 will never see this ARP reply. Because of this, the ARP entry for 10.100.200.4 will always remain incomplete, and leaf1 has no knowledge of how to forward traffic to this destination.

```
leaf1# show ip arp

Flags: * - Adjacencies learnt on non-active FHRP router
       + - Adjacencies synced via CFSoE
       # - Adjacencies Throttled for Glean
       CP - Added via L2RIB, Control plane Adjacencies
       PS - Added via L2RIB, Peer Sync
       RO - Re-Originated Peer Sync Entry
       D - Static Adjacencies attached to down interface

IP ARP Table for context default
Total number of entries: 4
Address         Age       MAC Address     Interface       Flags
198.51.100.1    00:09:48  5295.1e00.1b08  Ethernet1/1              
198.51.100.9    00:09:29  520a.3600.1b08  Ethernet1/2              
10.100.100.1    00:02:00  aac1.ab3e.c444  Vlan100                  
10.100.200.4    00:00:03  INCOMPLETE      Vlan200 
```

For Asymmetric IRB to work, the VTEP must build its ARP table from EVPN Type-2 MAC+IP routes. Without this functionality, Asymmetric IRB will fail, as we can clearly see. And it appears that NXOS does not do this, which is why it does not support Asymmetric IRB.

The following packet walk provides an easy to understand visual representation of the failure:

![packetwalk](/static/images/multivendor/nxos_evpn_hybrid_mode/packetwalk.jpg)

## Enabling Symmetric IRB between NXOS leafs

Before we look at how EVPN Hybrid mode works, let's first consider Symmetric IRB between leaf1 and leaf2 (both NXOS). The goal is to get h1 to talk to h2 (communication between VLAN 100 and VLAN 200). On the NXOS leafs, Symmetric IRB requires the following sample configuration (consider leaf1 as an example). First, a VLAN is created and the L3VNI is mapped it. This VNI is configured for the customer VRF, which we'll call 'Tenant1':

```
vlan 300
  vn-segment 10300
!
vrf context Tenant1
  vni 10300
  rd auto
  address-family ipv4 unicast
    route-target import 65421:300
    route-target import 65421:300 evpn
    route-target export 65421:300
    route-target export 65421:300 evpn
```

An anycast MAC address is configured on all the leafs (using '*fabric forwarding anycast-gateway-mac*'), and the IRB interfaces are enabled with '*fabric forwarding mode anycast-gateway*'. This enables the anycast gateway functionality on the IRB interface. Each leaf is also configured for its respective VLAN/BD under EVPN along with the import/export route-targets (these are the MAC VRF route-targets for the L2 domain).

You also need to create an IRB interface for the L3VNI VLAN and it is configured to be in the customer VRF.

```
fabric forwarding anycast-gateway-mac 000a.000a.000a
!
interface Vlan100
  no shutdown
  vrf member Tenant1
  ip address 10.100.100.254/24
  fabric forwarding mode anycast-gateway
!
interface Vlan300
  no shutdown
  vrf member Tenant1
  ip forward
```

The L3VNI is enabled under the NVE interface:

```
interface nve1
  no shutdown
  host-reachability protocol bgp
  advertise virtual-rmac
  source-interface loopback0
  member vni 10100
    ingress-replication protocol bgp
  member vni 10300 associate-vrf
```

By default, NXOS does not advertise an EVPN Type-5 subnet route (we'll talk about the need for this route at a later point in this blog post). This is done using the 'redistribute direct ...' configuration under the VRF IPv4 unicast address family under BGP:

```
router bgp 65421
  router-id 192.0.2.1
  log-neighbor-changes
  address-family ipv4 unicast
    redistribute direct route-map allow-loopback
    maximum-paths 4
  address-family l2vpn evpn
    advertise-pip
  template peer evpn
    update-source loopback0
    ebgp-multihop 2
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 192.0.2.11
    inherit peer evpn
    remote-as 65500
  neighbor 192.0.2.22
    inherit peer evpn
    remote-as 65500
  neighbor 198.51.100.1
    remote-as 65500
    address-family ipv4 unicast
  neighbor 198.51.100.9
    remote-as 65500
    address-family ipv4 unicast
  vrf Tenant1
    address-family ipv4 unicast
      redistribute direct route-map permit-all
```

Leaf1 learns h1s MAC address and IPv4 address using ARP/ND. We can initiate a ping from h1 (to its default gateway, 10.100.100.254, which is present on leaf1) to trigger this process.

```
root@h1:~# ping 10.100.100.254
PING 10.100.100.254 (10.100.100.254) 56(84) bytes of data.
64 bytes from 10.100.100.254: icmp_seq=1 ttl=255 time=4.43 ms
64 bytes from 10.100.100.254: icmp_seq=2 ttl=255 time=1.95 ms
64 bytes from 10.100.100.254: icmp_seq=3 ttl=255 time=2.16 ms
64 bytes from 10.100.100.254: icmp_seq=4 ttl=255 time=1.86 ms
^C
--- 10.100.100.254 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 1.856/2.599/4.431/1.064 ms
```

Within the VRF, an ARP entry is created for h1 and this is pushed to BGP EVPN as well. This gets advertised as an EVPN Type-2 MAC+IP route to the spines, and from there to leaf2.

```
leaf1# show ip arp vrf Tenant1 

Flags: * - Adjacencies learnt on non-active FHRP router
       + - Adjacencies synced via CFSoE
       # - Adjacencies Throttled for Glean
       CP - Added via L2RIB, Control plane Adjacencies
       PS - Added via L2RIB, Peer Sync
       RO - Re-Originated Peer Sync Entry
       D - Static Adjacencies attached to down interface

IP ARP Table for context Tenant1
Total number of entries: 1
Address         Age       MAC Address     Interface       Flags
10.100.100.1    00:01:33  aac1.ab3e.c444  Vlan100 

leaf1# show bgp l2vpn evpn 10.100.100.1
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 192.0.2.1:100    (L2VNI 10100)
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.ab3e.c444]:[32]:[10.100.100.1]/272, version 19
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    192.0.2.1 (metric 0) from 0.0.0.0 (192.0.2.1)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10100 10300
      Extcommunity: RT:100:100 RT:65421:300 ENCAP:8 Router MAC:52ca.b100.1b08

  Path-id 1 advertised to peers:
    192.0.2.11         192.0.2.22 
```

The extended communities added to this are important - you see both the L2 route-targets as well as the L3VNI route-target, along with the router MAC address. This router MAC address corresponds to IRB interface MAC address while the L3VNI route-target is used to identify (and import into) the customer VRF:

```
leaf1# show interface vlan100
Vlan100 is up, line protocol is up, autostate enabled
  Hardware is EtherSVI, address is  52ca.b100.1b08
  Internet Address is 10.100.100.254/24
  MTU 1500 bytes, BW 1000000 Kbit, DLY 10 usec,
   reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation ARPA, loopback not set
  Keepalive not supported
  ARP type: ARPA
  Last clearing of "show interface" counters never
  L3 in Switched:
    ucast: 0 pkts, 0 bytes
  L3 out Switched:
    ucast: 0 pkts, 0 bytes
```

On leaf2, the route gets imported into the VRF table with a matching L3VNI route-target. 

```
leaf2# show bgp l2vpn evpn 10.100.100.1
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 192.0.2.1:100
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.ab3e.c444]:[32]:[10.100.100.1]/272, version 21
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: external, path is valid, not best reason: newer EBGP path, no labeled nexthop
  AS-Path: 65500 65421 , path sourced external to AS
    192.0.2.1 (metric 0) from 192.0.2.22 (192.0.2.22)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10100 10300
      Extcommunity: RT:100:100 RT:65421:300 ENCAP:8 Router MAC:52ca.b100.1b08

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: Tenant1 L3-10300
  AS-Path: 65500 65421 , path sourced external to AS
    192.0.2.1 (metric 0) from 192.0.2.11 (192.0.2.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10100 10300
      Extcommunity: RT:100:100 RT:65421:300 ENCAP:8 Router MAC:52ca.b100.1b08

  Path-id 1 not advertised to any peer

Route Distinguisher: 192.0.2.2:3    (L3VNI 10300)
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.ab3e.c444]:[32]:[10.100.100.1]/272, version 20
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported from 192.0.2.1:100:[2]:[0]:[0]:[48]:[aac1.ab3e.c444]:[32]:[10.100.100.1]/272 
  AS-Path: 65500 65421 , path sourced external to AS
    192.0.2.1 (metric 0) from 192.0.2.11 (192.0.2.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10100 10300
      Extcommunity: RT:100:100 RT:65421:300 ENCAP:8 Router MAC:52ca.b100.1b08

  Path-id 1 not advertised to any peer
```

This creates a host route (a /32 route) entry into VRF table:

```
leaf2# show ip route 10.100.100.1 vrf Tenant1 
IP Route Table for VRF "Tenant1"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.100.100.1/32, ubest/mbest: 1/0
    *via 192.0.2.1%default, [20/0], 3d04h, bgp-65422, external, tag 65500, segid: 10300 tunnelid: 0xc0000201 encap: VXLAN
```

In addition to the host route, an EVPN Type-5 route (for the IRB subnet) is also advertised in accordance to the RFC (RFC 9135) for silent hosts (to trigger the gleaning process for such hosts). This subnet route is pulled into the VRF as well:

```
leaf2# show bgp l2vpn evpn 10.100.100.0
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 192.0.2.1:3
BGP routing table entry for [5]:[0]:[0]:[24]:[10.100.100.0]/224, version 14
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: external, path is valid, not best reason: newer EBGP path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65500 65421 , path sourced external to AS
    192.0.2.1 (metric 0) from 192.0.2.22 (192.0.2.22)
      Origin incomplete, MED not set, localpref 100, weight 0
      Received label 10300
      Extcommunity: RT:65421:300 ENCAP:8 Router MAC:52ca.b100.1b08

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: Tenant1 L3-10300
  Gateway IP: 0.0.0.0
  AS-Path: 65500 65421 , path sourced external to AS
    192.0.2.1 (metric 0) from 192.0.2.11 (192.0.2.11)
      Origin incomplete, MED not set, localpref 100, weight 0
      Received label 10300
      Extcommunity: RT:65421:300 ENCAP:8 Router MAC:52ca.b100.1b08

  Path-id 1 not advertised to any peer

Route Distinguisher: 192.0.2.2:3    (L3VNI 10300)
BGP routing table entry for [5]:[0]:[0]:[24]:[10.100.100.0]/224, version 10
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported from 192.0.2.1:3:[5]:[0]:[0]:[24]:[10.100.100.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: 65500 65421 , path sourced external to AS
    192.0.2.1 (metric 0) from 192.0.2.11 (192.0.2.11)
      Origin incomplete, MED not set, localpref 100, weight 0
      Received label 10300
      Extcommunity: RT:65421:300 ENCAP:8 Router MAC:52ca.b100.1b08

  Path-id 1 not advertised to any peer

leaf2# show ip route 10.100.100.0/24 vrf Tenant1 
IP Route Table for VRF "Tenant1"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.100.100.0/24, ubest/mbest: 1/0
    *via 192.0.2.1%default, [20/0], 3d04h, bgp-65422, external, tag 65500, segid: 10300 tunnelid: 0xc0000201 encap: VXLAN
```

A similar process happens in reverse for h2. On leaf1, h2s exact route and a subnet route should exist in the VRF table:

```
leaf1# show ip route vrf Tenant1 
IP Route Table for VRF "Tenant1"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.100.100.0/24, ubest/mbest: 1/0, attached
    *via 10.100.100.254, Vlan100, [0/0], 3d04h, direct
10.100.100.1/32, ubest/mbest: 1/0, attached
    *via 10.100.100.1, Vlan100, [190/0], 00:31:24, hmm
10.100.100.254/32, ubest/mbest: 1/0, attached
    *via 10.100.100.254, Vlan100, [0/0], 3d04h, local
10.100.200.0/24, ubest/mbest: 1/0
    *via 192.0.2.2%default, [20/0], 3d04h, bgp-65421, external, tag 65500, segid: 10300 tunnelid: 0xc0000202 encap: VXLAN
 
10.100.200.2/32, ubest/mbest: 1/0
    *via 192.0.2.2%default, [20/0], 3d04h, bgp-65421, external, tag 65500, segid: 10300 tunnelid: 0xc0000202 encap: VXLAN
```

h1 should now be able to ping h2 and a data-plane captures shows that the VNI in the VXLAN header is the L3VNI:

```
root@h1:~# ping 10.100.200.2
PING 10.100.200.2 (10.100.200.2) 56(84) bytes of data.
64 bytes from 10.100.200.2: icmp_seq=1 ttl=62 time=19.5 ms
64 bytes from 10.100.200.2: icmp_seq=2 ttl=62 time=29.8 ms
64 bytes from 10.100.200.2: icmp_seq=3 ttl=62 time=12.0 ms
64 bytes from 10.100.200.2: icmp_seq=4 ttl=62 time=12.0 ms
64 bytes from 10.100.200.2: icmp_seq=5 ttl=62 time=22.6 ms
^C
--- 10.100.200.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 9ms
rtt min/avg/max/mdev = 11.968/19.167/29.789/6.750 ms
```

![l3vni](/static/images/multivendor/nxos_evpn_hybrid_mode/l3vni.jpg)

## Configuring leaf3 for Asymmetric IRB

Let's configure leaf3 now for Asymmetric IRB. This implies that the IRB interface for both VLAN100 and VLAN200 exist on leaf3, along with their corresponding L2VNIs. Routing-instances are created for both VLAN100 and VLAN200 with service-type VLAN-based.

```
admin@leaf3# show
interfaces {

*snip*

    irb {
        unit 100 {
            virtual-gateway-accept-data;
            family inet {
                address 10.100.100.252/24 {
                    virtual-gateway-address 10.100.100.254;
                }
            }
            virtual-gateway-v4-mac 00:0a:00:0a:00:0a;
        }
        unit 200 {
            virtual-gateway-accept-data;
            family inet {               
                address 10.100.200.252/24 {
                    virtual-gateway-address 10.100.200.254;
                }
            }
            virtual-gateway-v4-mac 00:0a:00:0a:00:0a;
        }
    }
    lo0 {
        unit 0 {
            family inet {
                address 192.0.2.3/32;
            }
        }
    }
}
policy-options {
    policy-statement ECMP {
        then {
            load-balance per-flow;
        }
    }
    policy-statement allow-loopback {
        from interface lo0.0;
        then accept;
    }
}
routing-instances {
    v100_mac_vrf {
        instance-type mac-vrf;
        protocols {
            evpn {
                encapsulation vxlan;
                default-gateway do-not-advertise;
                extended-vni-list all;
            }
        }
        vtep-source-interface lo0.0;
        service-type vlan-based;
        interface xe-0/0/2.0;
        route-distinguisher 192.0.2.3:100;
        vrf-target target:100:100;
        vlans {                         
            v100 {
                vlan-id 100;
                l3-interface irb.100;
                vxlan {
                    vni 10100;
                }
            }
        }
    }
    v200_mac_vrf {
        instance-type mac-vrf;
        protocols {
            evpn {
                encapsulation vxlan;
                default-gateway do-not-advertise;
                extended-vni-list all;
            }
        }
        vtep-source-interface lo0.0;
        service-type vlan-based;
        route-distinguisher 192.0.2.3:200;
        vrf-target target:200:200;
        vlans {
            v200 {
                vlan-id 200;
                l3-interface irb.200;
                vxlan {
                    vni 10200;
                }
            }
        }
    }
}
routing-options {
    router-id 192.0.2.3;
    autonomous-system 65423;
    forwarding-table {
        export ECMP;
    }
}
protocols {
    bgp {
        group underlay {
            type external;
            family inet {
                unicast;
            }                           
            export allow-loopback;
            peer-as 65500;
            multipath;
            neighbor 198.51.100.5;
            neighbor 198.51.100.13;
        }
        group overlay {
            type external;
            multihop;
            local-address 192.0.2.3;
            family evpn {
                signaling;
            }
            peer-as 65500;
            neighbor 192.0.2.11;
            neighbor 192.0.2.22;
        }
    }
}
```

The IRB interfaces come up as long as there is at least one remote VXLAN tunnel that is created (these tunnels can be created by simply exchange EVPN Type-3 IMET routes):

```
admin@leaf3> show ethernet-switching vxlan-tunnel-end-point remote 
Logical System Name       Id  SVTEP-IP         IFL   L3-Idx    SVTEP-Mode    ELP-SVTEP-IP
<default>                 0   192.0.2.3        lo0.0    0  
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode  ELP-IP        Flags
 192.0.2.1        v100_mac_vrf             556       vtep.32770   1788    RNVE      
    VNID          MC-Group-IP      
    10100         0.0.0.0         
 RVTEP-IP         L2-RTT                   IFL-Idx   Interface    NH-Id   RVTEP-Mode  ELP-IP        Flags
 192.0.2.2        v200_mac_vrf             576       vtep.32771   1789    RNVE      
    VNID          MC-Group-IP      
    10200         0.0.0.0 

admin@leaf3> show interfaces irb 
Physical interface: irb, Enabled, Physical link is Up
  Interface index: 640, SNMP ifIndex: 506
  Type: Ethernet, Link-level type: Ethernet, MTU: 1514
  Device flags   : Present Running
  Interface flags: SNMP-Traps
  Link type      : Full-Duplex
  Link flags     : None
  Current address: 02:05:86:71:49:00, Hardware address: 02:05:86:71:49:00
  Last flapped   : Never
    Input packets : 0
    Output packets: 0

  Logical interface irb.100 (Index 567) (SNMP ifIndex 540)
    Flags: Up SNMP-Traps 0x4004000 Encapsulation: ENET2
    Virtual Gateway V4 MAC: 00:0a:00:0a:00:0a
    Bandwidth: 1Gbps
    Routing Instance: v100_mac_vrf Bridging Domain: v100
    Input packets : 0
    Output packets: 8
    Protocol inet, MTU: 1500
    Max nh cache: 75000, New hold nh limit: 75000, Curr nh cnt: 1, Curr new hold cnt: 0, NH drop cnt: 0
      Flags: Sendbcast-pkt-to-re
      Addresses, Flags: Is-Preferred Is-Primary
        Destination: 10.100.100/24, Local: 10.100.100.252, Broadcast: 10.100.100.255
        Destination: 10.100.100/24, Local: 10.100.100.254, Broadcast: 10.100.100.255

  Logical interface irb.200 (Index 568) (SNMP ifIndex 541)
    Flags: Up SNMP-Traps 0x4000 Encapsulation: ENET2
    Virtual Gateway V4 MAC: 00:0a:00:0a:00:0a
    Bandwidth: 1Gbps
    Routing Instance: v200_mac_vrf Bridging Domain: v200
    Input packets : 0
    Output packets: 2
    Protocol inet, MTU: 1514
    Max nh cache: 75000, New hold nh limit: 75000, Curr nh cnt: 1, Curr new hold cnt: 0, NH drop cnt: 0
      Flags: Sendbcast-pkt-to-re
      Addresses, Flags: Is-Preferred Is-Primary
        Destination: 10.100.200/24, Local: 10.100.200.252, Broadcast: 10.100.200.255
        Destination: 10.100.200/24, Local: 10.100.200.254, Broadcast: 10.100.200.255
```

## Why Symmetric IRB on one leaf and Asymmetric IRB on another leaf does not work

Our primary test is to check connectivity between h2 and h3 (since they are in different subnets - h2 is in VLAN200 and h3 is in VLAN100).

h3 can ping its default gateway (10.100.100.254, which exists on leaf3 since it is an anycast gateway):

```
root@h3:~# ping 10.100.100.254
PING 10.100.100.254 (10.100.100.254) 56(84) bytes of data.
64 bytes from 10.100.100.254: icmp_seq=1 ttl=64 time=119 ms
64 bytes from 10.100.100.254: icmp_seq=2 ttl=64 time=114 ms
64 bytes from 10.100.100.254: icmp_seq=3 ttl=64 time=128 ms
64 bytes from 10.100.100.254: icmp_seq=4 ttl=64 time=149 ms
^C
--- 10.100.100.254 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 113.884/127.348/149.132/13.538 ms
```

Naturally, h3 cannot ping h2:

```
root@h3:~# ping 10.100.200.2
PING 10.100.200.2 (10.100.200.2) 56(84) bytes of data.
^C
--- 10.100.200.2 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 90ms
```

This fails because while leaf3 has the required information for h2 (it has an IRB interface for VLAN200 and is thus able to ARP for h2 directly), leaf2 does not have any information about h3 in its forwarding table. It is receiving the EVPN Type-2 MAC+IP route but it does not know what to do with it since there is no corresponding L2VNI (more importantly, no matching import route-target). It simply drops the prefix, as seen below:

```
*snip*

leaf2# show bgp event-history prefixes 
2022-12-17T13:49:54.428360000+00:00 [M 27] [bgp] E_DEBUG [bgp_af_process_nlri:7447] (default) PFX: [L2VPN EVPN] Dropping prefix [2]:[0]:[0]:[48]:[aac1.abc1.c674]:[32]:[10.100.100.3]/144 from peer 192.0.2.22, due to attribu
te policy rejected
2022-12-17T13:49:54.428358000+00:00 [M 27] [bgp] E_DEBUG [bgp_af_process_nlri:7447] (default) PFX: [L2VPN EVPN] Dropping prefix [2]:[0]:[0]:[48]:[aac1.abc1.c674]:[0]:[0.0.0.0]/112 from peer 192.0.2.22, due to attribute pol
icy rejected

*snip*
```

## Enabling EVPN Hybrid mode on NXOS 

This is where EVPN Hybrid mode comes in. Cisco is purposefully very vague about how this really works, but we're going to break it down. From their documentation (https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/102x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-102x/m-evpn-hybrid-irb-mode.html), the most important pieces of information are listed below:


!!! quote "Statement from Cisco documentation"

    NX-OS symmetric IRB VTEPs must be provisioned with all subnets in an IP VRF that are stretched to asymmetric VTEPs in the fabric.
    NX-OS symmetric IRB VTEPs must be provisioned with subnets in an IP VRF that are stretched to asymmetric VTEPs in “hybrid” mode using “fabric forwarding mode anycast-gateway hybrid” CLI under the subnet SVI interface.
    All symmetric IRB VTEPs must have the hybrid mode enabled when interoperating with asymmetric VTEPs in each fabric.

So, you're really configuring the device for Asymmetric IRB, but with the continued existence of the IP VRF and the corresponding L3VNI. This is the additional configuration that we're going to add on leaf2 now:

```
vlan 100
  vn-segment 10100
!
interface Vlan100
  no shutdown
  vrf member Tenant1
  ip address 10.100.100.254/24
  fabric forwarding mode anycast-gateway hybrid
!
interface nve1
  no shutdown
  host-reachability protocol bgp
  advertise virtual-rmac
  source-interface loopback0
  member vni 10100
    ingress-replication protocol bgp
  member vni 10200
    ingress-replication protocol bgp
  member vni 10300 associate-vrf
!
evpn
  vni 10100 l2
    rd 192.0.2.2:100
    route-target import 100:100
    route-target export 100:100
  vni 10200 l2
    rd 192.0.2.2:200
    route-target import 200:200
    route-target export 200:200
```

## Pulling back the curtain

Because of the existence of the L2VNI, the EVPN Type-2 MAC+IP route is now accepted on leaf2:

```
leaf2# show bgp l2vpn evpn 10.100.100.3
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 192.0.2.2:100    (L2VNI 10100)
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.abc1.c674]:[32]:[10.100.100.3]/248, version 222
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, in rib
             Imported from 192.0.2.3:100:[2]:[0]:[0]:[48]:[aac1.abc1.c674]:[32]:[10.100.100.3]/248 
  AS-Path: 65500 65423 , path sourced external to AS
    192.0.2.3 (metric 0) from 192.0.2.22 (192.0.2.22)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10100
      Extcommunity: RT:100:100 ENCAP:8

  Path-id 1 not advertised to any peer

Route Distinguisher: 192.0.2.3:100
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.abc1.c674]:[32]:[10.100.100.3]/248, version 215
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: external, path is valid, not best reason: newer EBGP path, no labeled nexthop
  AS-Path: 65500 65423 , path sourced external to AS
    192.0.2.3 (metric 0) from 192.0.2.11 (192.0.2.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10100
      Extcommunity: RT:100:100 ENCAP:8

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10100
  AS-Path: 65500 65423 , path sourced external to AS
    192.0.2.3 (metric 0) from 192.0.2.22 (192.0.2.22)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10100
      Extcommunity: RT:100:100 ENCAP:8

  Path-id 1 not advertised to any peer
```

This is installed into the MAC address table:

```
leaf2# show mac address-table address aac1.abc1.c674
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan,
        (NA)- Not Applicable
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
C  100     aac1.abc1.c674   dynamic  NA         F      F    nve1(192.0.2.3)
```

So basically, with all of this new configuration, we have configured leaf2 for Asymmetric IRB (only for destinations behind leafs doing Asymmetric IRB). However, remember why Asymmetric IRB did not work in the first place on NXOS? The ARP process was functionally broken. Is it fixed now? Let's take a look:

```
leaf2# show ip arp vrf Tenant1 

Flags: * - Adjacencies learnt on non-active FHRP router
       + - Adjacencies synced via CFSoE
       # - Adjacencies Throttled for Glean
       CP - Added via L2RIB, Control plane Adjacencies
       PS - Added via L2RIB, Peer Sync
       RO - Re-Originated Peer Sync Entry
       D - Static Adjacencies attached to down interface

IP ARP Table for context Tenant1
Total number of entries: 1
Address         Age       MAC Address     Interface       Flags
10.100.200.2    00:00:08  aac1.ab83.9993  Vlan200     
```

Well, that's odd. No ARP for 10.100.100.3. Clearly h3 can ping h2 now, so how is it working?

```
root@h3:~# ping 10.100.200.2
PING 10.100.200.2 (10.100.200.2) 56(84) bytes of data.
64 bytes from 10.100.200.2: icmp_seq=1 ttl=63 time=129 ms
64 bytes from 10.100.200.2: icmp_seq=2 ttl=63 time=276 ms
64 bytes from 10.100.200.2: icmp_seq=3 ttl=63 time=148 ms
64 bytes from 10.100.200.2: icmp_seq=4 ttl=63 time=127 ms
^C
--- 10.100.200.2 ping statistics ---
5 packets transmitted, 4 received, 20% packet loss, time 11ms
rtt min/avg/max/mdev = 127.399/169.990/275.513/61.443 ms
```

The secret lies behind the 'fabric forwarding mode anycast-gateway hybrid' command - looking at the l2route table, you can see that the 'Asymmetric' flag is set (Asy), and the IP-MAC binding is sent directly to Adjacency Manager (AM) instead of installing it in the ARP table:

```
leaf2# show l2route evpn mac-ip all detail 
Flags -(Rmac):Router MAC (Stt):Static (L):Local (R):Remote (V):vPC link 
(Dup):Duplicate (Spl):Split (Rcv):Recv(D):Del Pending (S):Stale (C):Clear
(Ps):Peer Sync (Ro):Re-Originated (Orp):Orphan (Asy):Asymmetric (Gw):Gateway
(Piporp): Directly connected Orphan to PIP based vPC BGW 
(Pipporp): Orphan connected to peer of PIP based vPC BGW 
Topology    Mac Address    Host IP                                 Prod   Flags         Seq No     Next-Hops                              
----------- -------------- --------------------------------------- ------ ---------- ---------- ---------------------------------------
100         aac1.ab3e.c444 10.100.100.1                            BGP    --            0         192.0.2.1 (Label: 10100)               
            Sent To: AM
            encap-type:1
100         aac1.abc1.c674 10.100.100.3                            BGP    Asy           0         192.0.2.3 (Label: 10100)               
            Sent To: AM
            Peer ID: 2
            encap-type:1
100         000a.000a.000a 10.100.100.254                          BGP    Stt,Asy       0         192.0.2.3 (Label: 10100)               
            Sent To: AM
            ESI : 0500.00ff.8f00.0027.7400  
            encap-type:1
200         aac1.ab83.9993 10.100.200.2                            HMM    L,            0         Local                                  
            L3-Info: 10300
            Sent To: BGP
200         000a.000a.000a 10.100.200.254                          BGP    Stt,Asy       0         192.0.2.3 (Label: 10200)               
            ESI : 0500.00ff.8f00.0027.d800  
            encap-type:1
```

We can confirm this from the logs below. BGP EVPN sends this to l2rib, which in turn sends it to AM.

```
leaf2# show system internal l2rib event-history mac-ip 
2022-12-17T15:08:59.587921000+00:00 [M 27] [l2rib] E_DEBUG [l2rib_svr_mac_ip_ent_gpb_encode:587] (100,aac1.abc1.c674,10.100.100.3,5): Encoding MAC-IP best route (ADD, client id 0), esi: (F)
2022-12-17T15:08:59.583040000+00:00 [M 27] [l2rib] E_DEBUG [l2rib_obj_mac_ip_route_create:1434] (100,aac1.abc1.c674,10.100.100.3,5):  ESI: (F), port-channel ifIndex: 0
2022-12-17T15:08:59.582824000+00:00 [M 27] [l2rib] E_DEBUG [l2rib_obj_mac_ip_route_create:1422] (100,aac1.abc1.c674,10.100.100.3,5):  adminDist: 20, SOO: 0, peerID: 2, peer ifIndex: 1191182338
2022-12-17T15:08:59.582823000+00:00 [M 27] [l2rib] E_DEBUG [l2rib_obj_mac_ip_route_create:1410] (100,aac1.abc1.c674,10.100.100.3,5): MAC-IP route created with flags: 32, L3 VNI: 0, seqNum: 0
2022-12-17T15:08:59.582812000+00:00 [M 27] [l2rib] E_DEBUG [l2rib_obj_mac_ip_create:208] (100,aac1.abc1.c674,10.100.100.3): MAC-IP entry created
2022-12-17T15:08:59.582452000+00:00 [M 27] [l2rib] E_DEBUG [l2rib_client_show_mac_ip_route_msg:1462] NH: 192.0.2.3 (Label: 10100)
2022-12-17T15:08:59.582450000+00:00 [M 27] [l2rib] E_DEBUG [8381]: Rcvd MAC-IP ROUTE msg: res 0, esi (F), es_type 0, tag 0, ifindex 0, nh_count 1, pc-ifindex 0
2022-12-17T15:08:59.582447000+00:00 [M 27] [l2rib] E_DEBUG [l2rib_client_show_mac_ip_route_msg:1444] Rcvd MAC-IP ROUTE msg: flags Asy, admin_dist 0, seq 0, soo 0, peerid 0, 
2022-12-17T15:08:59.582445000+00:00 [M 27] [l2rib] E_DEBUG [l2rib_client_show_mac_ip_route_msg:1436] Rcvd MAC-IP ROUTE msg: (100, aac1.abc1.c674, 10.100.100.3), vrf_id 0, encap_type 1, 
2022-12-17T15:08:59.582443000+00:00 [M 27] [l2rib] E_DEBUG [l2rib_client_show_mac_ip_route_msg:1429] Rcvd MAC-IP ROUTE msg: (100, aac1.abc1.c674, 10.100.100.3), l2 vni 0, l3 vni 0

*snip*

leaf2# show system internal adjmgr internal event-history events
2022-12-17T15:08:59.683335000+00:00 [M 27] [adjmgr] E_DEBUG AM upd do work: Event=sentRouteToURIB, AFI=IPv4, routeCount=1
2022-12-17T15:08:59.683219000+00:00 [M 27] [adjmgr] E_DEBUG Append IPv4 adj route add to UPD: Result=Success, IP=10.100.100.3, IOD=77, Interface=Vlan100, prot0PhyIod=76, prot0PhyInterface=nve-peer2, prot1PhyIod=0, prot1Phy
Interface=None, tableId=0x3, adminDistance=250, mobility=0, nhCount=0
2022-12-17T15:08:59.683212000+00:00 [M 27] [adjmgr] E_DEBUG Append adj prot info to UPD: Result=Success, protAdjIndex=0, phyifIndex=0x47000002, phyIfType=71, Encap=1, tunnelId=0xffffffff
2022-12-17T15:08:59.614494000+00:00 [M 27] [adjmgr] E_DEBUG Processed L2RIB Msg. dbgStr=##48,5c,1b,1,1b,10,33,35,36,15,7a,c,67,12,68,9,44,b
2022-12-17T15:08:59.595194000+00:00 [M 27] [adjmgr] E_DEBUG Add to UPD work: Result=Success, IP=10.100.100.3, IOD=77, Interface=Vlan100, AFI=IPv4, workBits=0x1
2022-12-17T15:08:59.589259000+00:00 [M 27] [adjmgr] E_DEBUG Received MAC-IP update with AFI: 2 IP: 10.100.100.3 VRF: 0x0 l2r_ifIdx: 1191182338 peerId: 0x2 seqNum: 0 flags: 0x7 MAC : aac1.abc1.c674
2022-12-17T15:08:59.589230000+00:00 [M 27] [adjmgr] E_DEBUG Processed L2RIB Msg. dbgStr=##48,5d

*snip*
```

Our final confirmation is that the entry exists in the adjacency table:

```
leaf2# show ip adjacency vrf Tenant1 

Flags: # - Adjacencies Throttled for Glean
       G - Adjacencies of vPC peer with G/W bit
       R - Adjacencies learnt remotely
       CP - Added via L2RIB, Control plane Adjacencies
       PS - Added via L2RIB, Peer Sync
       RO - Re-Originated Peer Sync Entry
       CC - Consistency check pending

IP Adjacency Table for VRF Tenant1
Total number of entries: 2
Address         MAC Address     Pref Source     Interface         Mobility Flags
10.100.200.2    aac1.ab83.9993  50   arp        Vlan200                          
10.100.100.3    aac1.abc1.c674  50   am_l2rib   Vlan100                  0 CP R
```

As you can see, the source for 10.100.100.3 is 'am_l2rib' and the 'CP' flag is set, which implies it was learnt via L2RIB. The question that then arises from this is that is Cisco in violation of the RFC? Must we see this entry in the ARP table? No, they are not violating the RFC and it is not necessary to see the entry in the ARP table.

!!! note "Statement from RFC9135"

    For IP-to-MAC bindings learned via EVPN, an implementation may choose to import these bindings directly to the respective forwarding table (such as an adjacency/next-hop table) as opposed to importing them to ARP or ND protocol tables

It's a little sneaky, and easy to miss if you don't know where to look. And perhaps the reason it is implemented this way is to ensure that Asymmetric IRB continues to be natively unsupported on NXOS, but it will work (for interoperability reasons) if you have Symmetric IRB with L3VNIs, and the IRBs are configured in this EVPN Hybrid mode.