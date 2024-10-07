---
date:
  created: 2022-01-04
  updated: 2022-01-04
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
# Cisco SDA Part X - understanding L2 handoff
In this post, we take a detailed look at the L2 handoff feature in Cisco's SD-Access.

## Introduction and topology

Fair warning - this is going to be a long, long post. Get yourself some coffee because you're going to be here for a while!

We're going to continue working with the following topology for this post, with a legacy network added to the existing infrastructure:

![static1](/static/images/cisco/sda_10/L2_handoff_1.jpg)

<!-- more -->

This is a fairly common scenario that you might run into with customer's migrating to SD-Access. The premise is this - say you have subnet X that you're going to be migrating into SD-Access but it cannot be done in one shot or one window. The requirement is to have this subnet co-exist in the fabric as well as outside the fabric, within a legacy portion of the network. 

From the perspective of our topology, we have the 192.2.21.0/24 subnet that needs to extend between the legacy network and the SD-Access fabric. For these kind of situations, a feature called L2 handoff was introduced. This post will walk you through the provisioning of L2 handoff via DNAC and help you understand what kind of configuration is pushed to make it work. The biggest question that I always try to answer will also be addressed - HOW does it really work?

## L2 handoff configuration

The L2 handoff configuration flow is quite similar to L3 handoff. Under 'Provision', go to the 'Fabric' tab. Under 'Fabric Infrastructure', you should see your fabric:

![static1](/static/images/cisco/sda_10/L2_handoff_2.jpg)

At this point in time, only one border can be used to do L2 handoff. In my case, I'd like to use Border1 for this purpose, so let's click on Border1. Once you do this, you should see the various details about Border1:

![static1](/static/images/cisco/sda_10/L2_handoff_3.jpg)

Within the "Fabric" tab here, click "Configure":

![static1](/static/images/cisco/sda_10/L2_handoff_4.jpg)

In this page, you should see two major options - Layer 3 Handoff and Layer 2 Handoff. For the purposes of this post, we are interested in L2 handoff so that's where our focus will be. As you can see, all VNs that have an IP pool assigned to them will be listed here. 

In our case, we are interested in extended an IP pool within the Guest_VN, so that's what we are going to select. Click on the name of the VN itself here. Once you do this, the following page will be displayed:

![static1](/static/images/cisco/sda_10/L2_handoff_5.jpg)

This is where some input needs to be given - you need to provide the external link that connects to the legacy network (this is a drop down menu and lists all the interfaces on the box - you simply need to choose the relevant interface). Additionally, you need to provide the external VLAN number - this is the legacy VLAN number that is in use in the legacy network for this subnet. 

In my case, the interface that connects to the legacy switch is Gi1/0/12 and the legacy VLAN is 666. Let's input this into the GUI:

![static1](/static/images/cisco/sda_10/L2_handoff_6.jpg)

Click on 'Save' now and you should go one page back. The VN that you chose to do L2 handoff for should now have its checkbox checked. After this, click on 'Add' in the bottom right corner and DNAC should start pushing the relevant configuration to Border1. 

So, what we've seen so far is the "magic" portion of it. The point and click. But what we really want to understand is what happens behind the scenes, don't we? Here's what is pushed on Border1 in terms of L2 handoff configuration:

```
// removing loopback that matched anycast gateway IP for this subnet

%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:!exec: enable
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:no interface Loopback1027 

//enabling role-based enforcement for this new VLAN

%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:cts role-based enforcement vlan-list 666

// creating L2 VLAN for this legacy VLAN

%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:vlan 666
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:name 192_2_21_0-Guest_VN
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:exit 

//configuring the L2 handoff link as a trunk link allowing all VLANs

%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:interface GigabitEthernet1/0/12 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:no spanning-tree portfast 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:switchport 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:switchport mode trunk 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:switchport trunk allowed vlan all 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:exit 

//configuring a dynamic EID for this VN and a database-mapping for the L2 handoff subnet

%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:router lisp 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:instance-id 4099
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:dynamic-eid 192_2_21_0-Guest_VN-IPV4
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:database-mapping 192.2.21.0/24  locator-set rloc_23bd3133-ba7b-4058-9552-e37e69cbc843 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:exit-dynamic-eid 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:service ipv4 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:route-export site-registrations 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:distance site-registrations 250
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:map-cache site-registration

// creating a corresponding SVI for the legacy VLAN and enabling LISP mobility
// for the new dynamic EID that was created

%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:interface Vlan666 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:no lisp mobility liveness test 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:no ip redirects 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:mac-address 0000.0c9f.f45e
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:description Configured from Cisco DNA-Center
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:vrf forwarding Guest_VN 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:ip address 192.2.21.1 255.255.255.0
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:ip helper-address 192.2.201.224
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:ip route-cache same-interface 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:lisp mobility 192_2_21_0-Guest_VN-IPV4
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:exit 

// calling the legacy VLAN under a ethernet instance ID

%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:instance-id 8190
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:service ethernet 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:eid-table vlan 666
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:database-mapping mac locator-set rloc_23bd3133-ba7b-4058-9552-e37e69cbc843 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:exit-service-ethernet 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:remote-rloc-probe on-route-change 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:exit-instance-id 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:site site_uci 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:description map-server configured from Cisco DNA-Center
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:authentication-key uci

// enabling DHCP snooping for the legacy VLAN

%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:ip dhcp snooping vlan 666
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:ip dhcp relay information option 
%PARSER-5-CFGLOG_LOGGEDCMD: User:aninchat  logged command:ip domain lookup 
```

Okay, we're getting closer to understanding what's really happening behind the scenes but we aren't there yet. What is the significance of all of this configuration? Let's look at some of the more important pieces of configuration.

## Understanding the L2 handoff configuration

### SVI creation on the border, along with the LISP configuration

This is done in conjunction with removing the SVI on the legacy switch (presumably, this legacy switch was acting like the core of your network and was the first L3 hop and the default gateway for the hosts). The intention is to have Border1 as the default gateway for the legacy hosts now. 

Along with this, the LISP configuration allows for the IP addresses of the legacy hosts to be picked up natively by LISP (via the LISP dynamic EID configuration). This allows for the LISP database to be populated, from where the prefixes are pushed into RIB/FIB as directly connected routes. 

Let's confirm all of this for Host3, which is a host in the legacy network. 

```
Border1#show lisp eid-table vrf Guest_VN ipv4 database 
LISP ETR IPv4 Mapping Database for EID-table vrf Guest_VN (IID 4099), LSBs: 0x1
Entries total 3, no-route 0, inactive 1

192.2.21.21/32, dynamic-eid SDA_Guest-IPV4, inherited from default locator-set rloc_23bd3133-ba7b-4058-9552-e37e69cbc843, auto-discover-rlocs
  Locator       Pri/Wgt  Source     State
  192.2.101.65   10/10   cfg-intf   site-self, reachable
192.2.21.101/32, Inactive, expires: 23:04:11
192.2.201.224/32, route-import, inherited from default locator-set rloc_23bd3133-ba7b-4058-9552-e37e69cbc843, auto-discover-rlocs
  Locator       Pri/Wgt  Source     State
  192.2.101.65   10/10   cfg-intf   site-self, reachable
```

As you can see, the LISP database has picked up this prefix as a /32 entry and the RLOC is Border1 itself. This is also marked as reachable, which implies it can now be registered to the control-plane (again, which is also Border1). Confirm that we see an entry in the site table of Border1:

```
Border1#show lisp eid-table vrf Guest_VN ipv4 server 192.2.21.21
LISP Site Registration Information

Site name: site_uci
Description: map-server configured from Cisco DNA-Center
Allowed configured locators: any
Requested EID-prefix:

  EID-prefix: 192.2.21.21/32 instance-id 4099 
    First registered:     00:58:55
    Last registered:      00:00:15
    Routing table tag:    0
    Origin:               Dynamic, more specific of 192.2.21.0/24
    Merge active:         Yes
    Proxy reply:          Yes
    TTL:                  1d00h
    State:                complete
    Registration errors:  
      Authentication failures:   0
      Allowed locators mismatch: 0
    ETR 192.2.101.65, last registered 00:00:15, proxy-reply, map-notify
                      TTL 1d00h, merge, hash-function sha1, nonce 0x82F1568F-0xA0070DA9
                      state complete, no security-capability
                      xTR-ID 0xCEF75710-0x6FF58983-0x14550F97-0xD2031D6E
                      site-ID unspecified
      Locator       Local  State      Pri/Wgt  Scope
      192.2.101.65  yes    up          10/10   IPv4 none
    Merged locators
      Locator       Local  State      Pri/Wgt  Scope        Registering ETR
      192.2.101.65  yes    up          10/10   IPv4 none    192.2.101.65:4342 
```

The site table for this VRF has this entry. Perfect! The last thing to confirm is the RIB/FIB, as this is what will eventually be used in the forwarding plane:

```
// RIB entry

Border1#show ip route vrf Guest_VN 192.2.21.21

Routing Table: Guest_VN
Routing entry for 192.2.21.21/32
  Known via "lisp", distance 10, metric 1, type unknown
  Redistributing via bgp 65003
  Advertised by bgp 65003 metric 10
  Last update from 192.2.21.21 on Vlan666, 00:56:10 ago
  Routing Descriptor Blocks:
  * 192.2.21.21, from 0.0.0.0, 00:56:10 ago, via Vlan666
      Route metric is 1, traffic share count is 1

// CEF entry

Border1#show ip cef vrf Guest_VN 192.2.21.21 detail
192.2.21.21/32, epoch 0, flags [attached, subtree context]
  SC owned,sourced: LISP local EID - 
  SC inherited: LISP cfg dyn-EID - LISP configured dynamic-EID
  LISP EID attributes: localEID Yes, c-dynEID Yes, d-dynEID Yes
  SC owned,sourced: LISP generalised SMR - [disabled, not inheriting, 0xFFA5C97148 locks: 1]
  Adj source: IP adj out of Vlan666, addr 192.2.21.21 FFA5FE1200
    Dependent covered prefix type adjfib, cover 192.2.21.0/24
  2 IPL sources [no flags]
  nexthop 192.2.21.21 Vlan666
```

This looks good too. There's a direct adjacency off of VLAN666 and the next hop is the host IP address itself. Border1 can simply ARP for the host directly and once resolved, it can forward the traffic to it. 

```
// ARP entry for Host3

Border1#show ip arp vrf Guest_VN 192.2.21.21
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  192.2.21.21             0   000c.29f4.3a5e  ARPA   Vlan666

// mac entry for the corresponding mac address of Host3

Border1#show mac address-table address 000c.29f4.3a5e
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
 666    000c.29f4.3a5e    DYNAMIC     Gi1/0/12
Total Mac Addresses for this criterion: 1
```

As you can see, the host is reachable via the L2 handoff link.

### Fabric hosts

Now that we've established how a legacy host should be learnt via L2 handoff, let's take a look at the fabric hosts. In our topology, we have two fabric hosts - Host1 with an IP address of 192.2.21.19 and and Host2 with an IP address of 192.2.21.20. We're not going to go into too much detail of how LISP registers these; what I want to focus on is what's really happening on Border1 with these hosts. 

Once LISP has registered these /32 IP addresses of these hosts to the map-server (Border1/Border2), you should see them in the site table:

```
Border1#show lisp eid-table vrf Guest_VN ipv4 server 
LISP Site Registration Information
* = Some locators are down or unreachable
# = Some registrations are sourced by reliable transport

Site Name      Last      Up     Who Last             Inst     EID Prefix
               Register         Registered           ID       
site_uci       never     no     --                   4099     0.0.0.0/0
               never     no     --                   4099     192.2.21.0/24
               01:45:23  yes#   192.2.101.70:51604   4099     192.2.21.19/32
               01:23:24  yes#   192.2.101.71:17807   4099     192.2.21.20/32
               00:00:21  yes    192.2.101.65:4342    4099     192.2.21.21/32
               00:00:21  yes    192.2.101.65:4342    4099     192.2.201.224/32
```

Good - we certainly see those hosts in there. Now, from the site table, a particular LISP configuration pulls these host entries into the RIB, with an AD of 250 and installs them against Null0. 

```
Border1#show run | sec router lisp
router lisp

 *snip*

 !
 instance-id 4099
  remote-rloc-probe on-route-change
  dynamic-eid SDA_Guest-IPV4
   database-mapping 192.2.21.0/24 locator-set rloc_23bd3133-ba7b-4058-9552-e37e69cbc843
   exit-dynamic-eid
  !
  service ipv4
   eid-table vrf Guest_VN
   map-cache 0.0.0.0/0 map-request
   route-import database bgp 65003 route-map DENY-Guest_VN locator-set rloc_23bd3133-ba7b-4058-9552-e37e69cbc843
   route-export site-registrations
   distance site-registrations 250
   map-cache site-registration
   exit-service-ipv4
  !
  exit-instance-id
 !
 
 *snip*

 exit-router-lisp
```

Look at the RIB now and confirm that these entries are present against Null0:

```
Border1#show ip route vrf Guest_VN 

Routing Table: Guest_VN
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is 192.2.100.10 to network 0.0.0.0

B*    0.0.0.0/0 [20/0] via 192.2.100.10, 7w0d
      192.2.21.0/24 is variably subnetted, 5 subnets, 2 masks
C        192.2.21.0/24 is directly connected, Vlan666
L        192.2.21.1/32 is directly connected, Vlan666
l        192.2.21.19/32 [250/1], 01:41:14, Null0
l        192.2.21.20/32 [250/1], 01:23:43, Null0
l        192.2.21.21/32 [10/1] via 192.2.21.21, 01:10:15, Vlan666
      192.2.100.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.2.100.8/30 is directly connected, Vlan3016
L        192.2.100.9/32 is directly connected, Vlan3016
      192.2.201.0/32 is subnetted, 1 subnets
B        192.2.201.224 [20/0] via 192.2.100.10, 7w0d
```

Perfect! So, now, we have our fabric hosts in the RIB as well. Why are these prefixes against Null0 though? We'll understand this once we take a look at the packet walks.

## Packet walk for North to South traffic

Okay, this is going to be two fold.  We're going to take a look at a host beyond the Fusion trying to reach a host within the fabric and as well as a host in the legacy network. Let's use the DNAC as a source host in this case. 

### DNAC to fabric hosts (Host1/Host2) reachability

From DNAC, we're going to do a simple ping to 192.2.21.19 and 192.2.21.20. The packet gets routed through the infrastructure that is above the borders and it eventually reaches Border1. 

![static1](/static/images/cisco/sda_10/L2_handoff_7.jpg)

On Border1, a forwarding lookup is done and it matches the Null0 entry for these prefixes. 

```
Border1#show ip route vrf Guest_VN 192.2.21.19

Routing Table: Guest_VN
Routing entry for 192.2.21.19/32
  Known via "lisp", distance 250, metric 1, type intra area
  Redistributing via bgp 65003
  Advertised by bgp 65003 metric 10
  Routing Descriptor Blocks:
  * directly connected, via Null0
      Route metric is 1, traffic share count is 1
```

This is why Null0 is important - one of the rules for triggering the LISP process is that the prefix should point to Null0. So, this implies that when this entry is hit, the LISP process is now invoked. 

LISP now tries to locate this EID. It goes into its map-cache to determine if there is an entry for this or not. Remember Border1s LISP configuration? It had a dynamic EID created for 192.2.21.0/24 - this also creates a corresponding entry in the map-cache table with an action of 'send-map-request'. 

```
Border1#show lisp eid-table vrf Guest_VN ipv4 map-cache            
LISP IPv4 Mapping Cache for EID-table vrf Guest_VN (IID 4099), 7 entries

0.0.0.0/0, uptime: 7w0d, expires: never, via static-send-map-request
  Negative cache entry, action: send-map-request
0.0.0.0/1, uptime: 7w0d, expires: 00:00:41, via map-reply, forward-native
  Encapsulating to proxy ETR 
192.2.21.0/24, uptime: 7w0d, expires: never, via dynamic-EID, send-map-request
  Negative cache entry, action: send-map-request

*snip* 
```

Border1 can now query the map-resolver (it'll just query itself since it is a map-resolver also) and determine where this EID is located. 

We can see that we should hit the following entry in the site table:

```
Border1#show lisp eid-table vrf Guest_VN ipv4 server 192.2.21.19
LISP Site Registration Information

Site name: site_uci
Description: map-server configured from Cisco DNA-Center
Allowed configured locators: any
Requested EID-prefix:

  EID-prefix: 192.2.21.19/32 instance-id 4099 
    First registered:     22:09:16
    Last registered:      22:09:16
    Routing table tag:    0
    Origin:               Dynamic, more specific of 192.2.21.0/24
    Merge active:         No
    Proxy reply:          Yes
    TTL:                  1d00h
    State:                complete
    Registration errors:  
      Authentication failures:   0
      Allowed locators mismatch: 0
    ETR 192.2.101.70:51604, last registered 22:09:16, proxy-reply, map-notify
                            TTL 1d00h, no merge, hash-function sha1, nonce 0xE204F906-0x7BD3976D
                            state complete, no security-capability
                            xTR-ID 0x774DBE71-0x20FB1182-0x5798970B-0x64432A97
                            site-ID unspecified
                            sourced by reliable transport
      Locator       Local  State      Pri/Wgt  Scope
      192.2.101.70  yes    up          10/10   IPv4 none
```

![static1](/static/images/cisco/sda_10/L2_handoff_8.jpg)

Once this map-request/map-reply process is over, the map-cache should be appropriately built with this entry and CEF should be overwritten as well, with Edge1 as the next hop for this host. 

```
Border1#show lisp eid-table vrf Guest_VN ipv4 map-cache 192.2.21.19
LISP IPv4 Mapping Cache for EID-table vrf Guest_VN (IID 4099), 7 entries

192.2.21.19/32, uptime: 00:09:39, expires: 23:50:20, via map-reply, complete
  Sources: map-reply, site-registration
  State: complete, last modified: 00:09:39, map-source: 192.2.101.70
  Exempt, Packets out: 25(14400 bytes) (~ 00:00:42 ago)
  Configured as EID address space
  Encapsulating dynamic-EID traffic
  Locator       Uptime    State      Pri/Wgt     Encap-IID
  192.2.101.70  00:09:39  up          10/10        -
    Last up-down state change:         00:09:39, state change count: 1
    Last route reachability change:    22:28:41, state change count: 1
    Last priority / weight change:     never/never
    RLOC-probing loc-status algorithm:
      Last RLOC-probe sent:            00:09:39 (rtt 2ms)
```

Our final confirmation is the CEF entry:

```
Border1#show ip cef vrf Guest_VN 192.2.21.19
192.2.21.19/32
  nexthop 192.2.101.70 LISP0.4099
Border1#show ip cef vrf Guest_VN 192.2.21.19 detail
192.2.21.19/32, epoch 0, flags [subtree context, check lisp eligibility]
  SC owned,sourced: LISP remote EID - locator status bits 0x00000001
  LISP remote EID: 39 packets 22464 bytes fwd action encap, cfg as EID space, dynamic EID need encap
  SC inherited: LISP cfg dyn-EID - LISP configured dynamic-EID
  LISP EID attributes: localEID No, c-dynEID Yes, d-dynEID No
  LISP source path list
    nexthop 192.2.101.70 LISP0.4099
  2 IPL sources [no flags]
  nexthop 192.2.101.70 LISP0.4099  
```

The packet is now encapsulated and sent to Edge1:

![static1](/static/images/cisco/sda_10/L2_handoff_9.jpg)

Inline, this packet looks like this:

![static1](/static/images/cisco/sda_10/L2_handoff_10.jpg)

### DNAC to legacy host

Let's now take a look at how a packet reaches the legacy host (Host3). This is very straightforward. Again, the packet gets routed towards the borders and reaches Border1. 

![static1](/static/images/cisco/sda_10/L2_handoff_11.jpg)

Border1 does a lookup in its forwarding table: 

```
Border1#show ip cef vrf Guest_VN 192.2.21.21 detail 
192.2.21.21/32, epoch 0, flags [attached, subtree context]
  SC owned,sourced: LISP local EID - 
  SC inherited: LISP cfg dyn-EID - LISP configured dynamic-EID
  LISP EID attributes: localEID Yes, c-dynEID Yes, d-dynEID Yes
  SC owned,sourced: LISP generalised SMR - [disabled, not inheriting, 0xFFA5C97148 locks: 1]
  Adj source: IP adj out of Vlan666, addr 192.2.21.21 FFA5FE1200
    Dependent covered prefix type adjfib, cover 192.2.21.0/24
  2 IPL sources [no flags]
  nexthop 192.2.21.21 Vlan666
```

This is a directly connected host so the ARP/CAM table will lead the packet out of the L2 handoff link. The following EPC capture on the legacy switch proves that the packet is a native IP packet, with only a 802.1Q header with a VLAN ID of 666:

```
Frame 42: 102 bytes on wire (816 bits), 102 bytes captured (816 bits) on interface 0
    Interface id: 0
    Encapsulation type: Ethernet (1)
    Arrival Time: Mar 28, 2020 11:53:31.778445000 UTC
    [Time shift for this packet: 0.000000000 seconds]
    Epoch Time: 1585396411.778445000 seconds
    [Time delta from previous captured frame: 0.044992000 seconds]
    [Time delta from previous displayed frame: 0.044992000 seconds]
    [Time since reference or first frame: 6.960737000 seconds]
    Frame Number: 42
    Frame Length: 102 bytes (816 bits)
    Capture Length: 102 bytes (816 bits)
    [Frame is marked: False]
    [Frame is ignored: False]
    [Protocols in frame: eth:vlan:ip:icmp:data]
Ethernet II, Src: Cisco_9f:f4:62 (00:00:0c:9f:f4:62), Dst: Vmware_f4:3a:5e (00:0c:29:f4:3a:5e)
    Destination: Vmware_f4:3a:5e (00:0c:29:f4:3a:5e)
        Address: Vmware_f4:3a:5e (00:0c:29:f4:3a:5e)
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: Cisco_9f:f4:62 (00:00:0c:9f:f4:62)
        Address: Cisco_9f:f4:62 (00:00:0c:9f:f4:62)
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: 802.1Q Virtual LAN (0x8100)
802.1Q Virtual LAN, PRI: 0, CFI: 0, ID: 666
    000. .... .... .... = Priority: Best Effort (default) (0)
    ...0 .... .... .... = CFI: Canonical (0)
    .... 0010 1001 1010 = ID: 666
    Type: IP (0x0800)
Internet Protocol Version 4, Src: 172.17.22.11 (172.17.22.11), Dst: 192.2.21.21 (192.2.21.21)
    Version: 4
    Header length: 20 bytes
    Differentiated Services Field: 0x00 (DSCP 0x00: Default; ECN: 0x00: Not-ECT (Not ECN-Capable Transport))
        0000 00.. = Differentiated Services Codepoint: Default (0x00)
        .... ..00 = Explicit Congestion Notification: Not-ECT (Not ECN-Capable Transport) (0x00)
    Total Length: 84
    Identification: 0x9a62 (39522)
    Flags: 0x02 (Don't Fragment)
        0... .... = Reserved bit: Not set
        .1.. .... = Don't fragment: Set
        ..0. .... = More fragments: Not set
    Fragment offset: 0
    Time to live: 61
    Protocol: ICMP (1)
    Header checksum: 0x0c13 [validation disabled]
        [Good: False]
        [Bad: False]
    Source: 172.17.22.11 (172.17.22.11)
    Destination: 192.2.21.21 (192.2.21.21)
Internet Control Message Protocol
    Type: 8 (Echo (ping) request)
    Code: 0
    Checksum: 0x1edf [correct]
    Identifier (BE): 27426 (0x6b22)
    Identifier (LE): 8811 (0x226b)
    Sequence number (BE): 1 (0x0001)
    Sequence number (LE): 256 (0x0100)
    Timestamp from icmp data: Mar 28, 2020 12:15:37.000000000 UTC
    [Timestamp from icmp data (relative): -1325.221555000 seconds]
    Data (48 bytes) 
```

This can be visualized like so:

![static1](/static/images/cisco/sda_10/L2_handoff_12.jpg)

## Packet walk for East to West traffic

For this, we're going to take a look at a packet walk between Host1 (a fabric host) and Host3 (a legacy host). Since Host1 and Host3 are in the same subnet, when Host1 pings Host3, it will ARP for Host3 directly. 

ARP is a funny little thing within SDA so I'm not going to go into too much detail. Essentially, when Edge1 gets the ARP for Host3, it will query the map-server for Host3s IP address and learn its mac address. It then queries the mac address and the map-resolver will return Border1 as the RLOC in this case. 

The ARP is now encapsulated and sent to Border1. Visually, this should look like:

![static1](/static/images/cisco/sda_10/L2_handoff_13.jpg)

This is a unicast encapsulation; the destination IP address in the outer header is unicast - let's confirm that via a packet capture.  The following packet capture is taken inbound on Border1 as the packet comes from Edge1:

![static1](/static/images/cisco/sda_10/L2_handoff_14.jpg)

Remember, the inner ARP is still a broadcast. The packet is decapsulated by Border1 and this inner ARP is flooded in VLAN 666 (which maps to the VNID we see in the VXLAN header - 8196). 

This flooded ARP goes out the L2 handoff link and reaches Host3 via regular broadcast flood:

![static1](/static/images/cisco/sda_10/L2_handoff_15.jpg)

Similar process happens in reverse - Host3 replies to the ARP, which eventually gets encapsulated by Border1 and sent to Edge1, where it is decapsulated and sent to Host1. Host1 now has Host3s IP address resolved to its mac address so it can generate an ICMP echo and send it to Host3. This gets encapsulated and sent to Border1, since it is the RLOC for the legacy host. 

On the Border1, the forwarding table will say that this is directly connected:

```
// RIB entry

Border1#show ip route vrf Guest_VN 192.2.21.21

Routing Table: Guest_VN
Routing entry for 192.2.21.21/32
  Known via "lisp", distance 10, metric 1, type unknown
  Redistributing via bgp 65003
  Advertised by bgp 65003 metric 10
  Last update from 192.2.21.21 on Vlan666, 00:56:10 ago
  Routing Descriptor Blocks:
  * 192.2.21.21, from 0.0.0.0, 00:56:10 ago, via Vlan666
      Route metric is 1, traffic share count is 1

// CEF entry

Border1#show ip cef vrf Guest_VN 192.2.21.21 detail
192.2.21.21/32, epoch 0, flags [attached, subtree context]
  SC owned,sourced: LISP local EID - 
  SC inherited: LISP cfg dyn-EID - LISP configured dynamic-EID
  LISP EID attributes: localEID Yes, c-dynEID Yes, d-dynEID Yes
  SC owned,sourced: LISP generalised SMR - [disabled, not inheriting, 0xFFA5C97148 locks: 1]
  Adj source: IP adj out of Vlan666, addr 192.2.21.21 FFA5FE1200
    Dependent covered prefix type adjfib, cover 192.2.21.0/24
  2 IPL sources [no flags]
  nexthop 192.2.21.21 Vlan666
```

The packet is decap'd and sent out natively towards Host3. A similar process happens in the reverse direction - the native packet reaches Border1. Because the destination mac address is of Border1 itself, a routing lookup is done which results in either the Null0 entry for 192.2.21.19 or a LISP resolved entry:

```
Border1#show ip cef vrf Guest_VN 192.2.21.19
192.2.21.19/32
  nexthop 192.2.101.70 LISP0.4099
Border1#show ip cef vrf Guest_VN 192.2.21.19 detail
192.2.21.19/32, epoch 0, flags [subtree context, check lisp eligibility]
  SC owned,sourced: LISP remote EID - locator status bits 0x00000001
  LISP remote EID: 39 packets 22464 bytes fwd action encap, cfg as EID space, dynamic EID need encap
  SC inherited: LISP cfg dyn-EID - LISP configured dynamic-EID
  LISP EID attributes: localEID No, c-dynEID Yes, d-dynEID No
  LISP source path list
    nexthop 192.2.101.70 LISP0.4099
  2 IPL sources [no flags]
  nexthop 192.2.101.70 LISP0.4099 
```

Visually, the entire forwarding path is like so in the direction of Host1->Host3:

![static1](/static/images/cisco/sda_10/L2_handoff_16.jpg)

And like so in the direction of Host3->Host1:

![static1](/static/images/cisco/sda_10/L2_handoff_17.jpg)