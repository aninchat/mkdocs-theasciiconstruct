---
date:
  created: 2021-12-26
  updated: 2021-12-26
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
# Cisco SDA Part II - basic LISP configuration and operation
In this post, we introduce basic LISP configuration and operation using packet captures and packet walks.

## Introduction and topology

Now that we've covered this new routing paradigm that LISP introduces in the previous post and understood some commonly used terms, we will move on to basic LISP operations.  

The topology that we will be using is the following:

![static1](/static/images/cisco/sda_2/lisp_basic_1.jpg)

<!-- more -->

Some generic details about the setup - R1 and R5 have a loopback each (loopback0) with IP addresses of 1.1.1.1/24 and 5.5.5.5/24 respectively. There is an OSPF adjacency between R1 and xTR2 as well as R5 and xTR4. Loopback0 is advertised into OSPF by both R1 and R5. xTR2 and xTR4 learn these respective loopbacks.

```
// R1

R1#show run int lo0
Building configuration...

Current configuration : 93 bytes
!
interface Loopback0
 ip address 1.1.1.1 255.255.255.0
 ip ospf network point-to-point
end

R1#show ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2           0   FULL/  -        00:00:37    10.1.12.2       GigabitEthernet0/0


// R5

R5#show run int lo0
Building configuration...

Current configuration : 93 bytes
!
interface Loopback0
 ip address 5.5.5.5 255.255.255.0
 ip ospf network point-to-point
end

R5#show ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
4.4.4.4           0   FULL/  -        00:00:36    10.1.45.4       GigabitEthernet0/0
```

xTR2, xTR4 and the MS/MR have a /32 loopback each:

```
// xTR2

xTR2#show run int lo0
Building configuration...

Current configuration : 63 bytes
!
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
end

// MS/MR

MS_MR#show run int l0
Building configuration...

Current configuration : 63 bytes
!
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
end

// xTR4

xTR4#show run int lo0
Building configuration...

Current configuration : 63 bytes
!
interface Loopback0
 ip address 4.4.4.4 255.255.255.255
end
```

The LISP cloud has some form of underlay (we don't need to go into details of that) that allows for these loopbacks to talk to each other and have complete reachability between them.  

```
// reachability to MS/MRs loopback    
    
xTR2#ping 3.3.3.3 source 2.2.2.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
Packet sent with a source address of 2.2.2.2 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/4/6 ms

// reachability to R4s loopback   

xTR2#ping 4.4.4.4 source 2.2.2.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 4.4.4.4, timeout is 2 seconds:
Packet sent with a source address of 2.2.2.2 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/4/5 ms
```

Apart from basic IP addressing between each device (and details described earlier), there is nothing else that has been pre-configured on these devices.

## LISP MS/MR configuration

Let's go ahead and start configuring each device role in the topology now. We are going to start with the MS/MR. 

```
// instantiate the LISP process 

MS_MR(config)#router lisp

// mapping sites to prefixes that can be accepted for that site

MS_MR(config-router-lisp)#site SITE_A
MS_MR(config-router-lisp-site)#authentication-key cisco
MS_MR(config-router-lisp-site)#eid-prefix 1.1.1.0/24
MS_MR(config-router-lisp-site)#exit

MS_MR(config-router-lisp)#site SITE_B 
MS_MR(config-router-lisp-site)#authentication-key cisco
MS_MR(config-router-lisp-site)#eid-prefix 5.5.5.0/24
MS_MR(config-router-lisp-site)#exit

// configuring this device to be a MS/MR

MS_MR(config-router-lisp)#ipv4 map-server 
MS_MR(config-router-lisp)#ipv4 map-resolver 
```

On the MS/MR, outside of configuring it as a Map Server and Map Resolver, you also need to define what prefixes it is allowed to accept and store in its EID to RLOC mapping database and against what site. Next, configure the xTRs. This is where you setup the EID advertisement and point to the MS/MR. 

```
// instantiate LISP
// define table and instance ID

xTR2(config)#router lisp
xTR2(config-router-lisp)#eid-table default instance-id 0

// advertise EIDs

xTR2(config-router-lisp-eid-table)#database-mapping 1.1.1.0/24 2.2.2.2 priority 1 weight 50       
xTR2(config-router-lisp-eid-table)#exit

// configure device role to be xTR (both iTR and eTR)
// additionally, specify IP address of MS/MR

xTR2(config-router-lisp)# ipv4 itr map-resolver 3.3.3.3
xTR2(config-router-lisp)# ipv4 itr
xTR2(config-router-lisp)# ipv4 etr map-server 3.3.3.3 key cisco
xTR2(config-router-lisp)# ipv4 etr
xTR2(config-router-lisp)# exit
```

Similar configuration for xTR4, only the database-mapping line changes a bit:

```
// instantiate LISP
// define table and instance ID

xTR4(config)#router lisp
xTR4(config-router-lisp)#eid-table default instance-id 0

// advertise EIDs

xTR4(config-router-lisp-eid-table)#database-mapping 5.5.5.0/24 4.4.4.4 priority 1 weight 50       
xTR4(config-router-lisp-eid-table)#exit

// configure device role to be xTR (both iTR and eTR)
// additionally, specify IP address of MS/MR

xTR4(config-router-lisp)# ipv4 itr map-resolver 3.3.3.3
xTR4(config-router-lisp)# ipv4 itr
xTR4(config-router-lisp)# ipv4 etr map-server 3.3.3.3 key cisco
xTR4(config-router-lisp)# ipv4 etr
xTR4(config-router-lisp)# exit
```

Take a look at the database-mapping statement closely - what it does is advertise the EID (which is in red) along with the RLOC (which is in yellow) to the MS/MR (we are setting the RLOCs to be the loopback0 IP address). The MS/MR now stores this in its EID to RLOC mapping database.  

Now that we've configured the xTRs and the MS/MR, let's take a look at what is really going on behind the scenes. When you configure the xTRs and advertise an EID to RLOC mapping, it sends a Map Register to the MS/MR. 

![static1](/static/images/cisco/sda_2/lisp_basic_2.jpg)

The Map Register packet looks like this:

![static1](/static/images/cisco/sda_2/lisp_basic_3.jpg)

From the Map Register, the MS/MR pulls out relevant information to populate its EID to RLOC mapping database. Additionally, MS/MR sends a Map Notify back (only if the 'M' bit is set in the Map Register which essentially means it expects to get a Map Notify back). 

![static1](/static/images/cisco/sda_2/lisp_basic_4.jpg)

The Map Notify packet looks like this:

![static1](/static/images/cisco/sda_2/lisp_basic_5.jpg)

This process ends here. The MS/MR does not 'push' this information to anyone else. From a xTR perspective, there are two important tables that you should be looking at - the LISP database table and the LISP map-cache. The LISP database table is where it stores the locally attached EIDs that are being advertised to the MS/MR and the LISP map-cache is what is actively used to build the LISP forwarding/data-plane. 


## LISP database table

```
// xTR2s LISP database table

xTR2#show ip lisp database 
LISP ETR IPv4 Mapping Database for EID-table default (IID 0), LSBs: 0x1, 1 entries

1.1.1.0/24
  Locator  Pri/Wgt  Source     State
  2.2.2.2    1/50   cfg-addr   site-self, reachable

// xTR4s LISP database table

xTR4#show ip lisp database 
LISP ETR IPv4 Mapping Database for EID-table default (IID 0), LSBs: 0x1, 1 entries

5.5.5.0/24
  Locator  Pri/Wgt  Source     State
  4.4.4.4    1/50   cfg-addr   site-self, reachable
```

It is important to understand how the database determines if an EID is 'reachable' or not - this is not a data packet driven derivative. There is no active reachability check that is done against this EID. It is purely from the existence of the prefix in RIB. 


For example, I shut down R1s interface to xTR2. With an intermediate dumb device between R1 and xTR2, xTR2s interface stays up despite R1 being down. Eventually, this causes OSPF peering to go down and 1.1.1.0/24 to be pulled from RIB. At this point, the state of this EID in the LISP database is moved to unreachable: 

```
// OSPF peering goes down

%OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.1 on GigabitEthernet0/0 from FULL to DOWN, Neighbor Down: Dead timer expired 

// prefix pulled from RIB

xTR2#show ip route 1.1.1.1
% Network not in table 

// EID marked as unreachable
// Map-Register sent to MS/MR with EID as unreachable

xTR2#show ip lisp database 
LISP ETR IPv4 Mapping Database for EID-table default (IID 0), LSBs: 0x0, 1 entries

*** ALL CONFIGURED LOCAL EID PREFIXES HAVE NO ROUTE ***
***      REPORTING LOCAL RLOCS AS UNREACHABLE       ***

1.1.1.0/24 *** NO ROUTE TO EID PREFIX ***
  Locator  Pri/Wgt  Source     State
  2.2.2.2    1/50   cfg-addr   site-self, unreachable 
```

I can now configure a static route towards that subnet and it becomes reachable again, thus proving what we had theorized earlier:

```
// configuring a static route to 1.1.1.0/24 on xTR2

xTR2(config)#ip route 1.1.1.0 255.255.255.0 10.1.12.1
xTR2(config)#end

// xTR2s LISP database

xTR2#show ip lisp database 
LISP ETR IPv4 Mapping Database for EID-table default (IID 0), LSBs: 0x1, 1 entries

1.1.1.0/24
  Locator  Pri/Wgt  Source     State
  2.2.2.2    1/50   cfg-addr   site-self, reachable
```

## LISP map-cache

Remember how we talked about LISP being a pull model? Well, at this point, with just the EID registrations being done with the MS/MR, the xTRs have no knowledge of each others EIDs in their map-caches (which is used for LISP data-plane). 

Thus, at time=0 (where time=0 is the point when xTRs have just registered EIDs with the MS/MR and no data-plane traffic is flowing right now) the map-cache is empty except for one, very important, default entry.

```
// XTR2s LISP map-cache

xTR2#show ip lisp map-cache       
LISP IPv4 Mapping Cache for EID-table default (IID 0), 1 entries

0.0.0.0/0, uptime: 00:10:02, expires: never, via static send map-request
  Negative cache entry, action: send-map-request  

// xTR4s LISP map-cache

xTR4#show ip lisp map-cache 
LISP IPv4 Mapping Cache for EID-table default (IID 0), 1 entries

0.0.0.0/0, uptime: 00:05:45, expires: never, via static send map-request
  Negative cache entry, action: send-map-request
```

This map-cache entry triggers a 'Map Request' to the MS/MR (this is the 'pull' that forms the basis of the LISP model), essentially requesting for information about the EID subnet it is trying to reach. The MS/MR looks at its site database for the EID. If it finds it, it forwards the Map Request to the RLOC that registered the EID. This RLOC now responds to the source RLOC and the source RLOC installs the EID to RLOC mapping (information gleaned from the Map Reply) in its map-cache. This new entry is now used for data-plane forwarding.  

## LISP process - end to end

Now that we have a general understanding of the tables in use, let's look at an entire flow, end to end by having R1 ping R5s loopback (5.5.5.5) sourcing its own loopback (1.1.1.1). R1s default gateway is xTR2, so after determining that the destination is not in its subnet, R1 ARPs for its default gateway (xTR2), resolves that and sends the packet to xTR2.

![static1](/static/images/cisco/sda_2/lisp_basic_6.jpg)
 
xTR2 looks at the destination mac, realize it owns it and strips the L2 header. It then does a lookup against the destination IP address in the IP header. This is where some of the magic begins. While RIB shows no route to the destination, look closely at the CEF entry:

```
// RIB lookup on xTR2

xTR2#show ip route 5.5.5.5
% Network not in table

// CEF lookup on xTR2

xTR2#show ip cef 5.5.5.5 detail
0.0.0.0/0, epoch 0, flags [default route handler, cover dependents, check lisp eligibility, default route]
  LISP remote EID: 0 packets 0 bytes fwd action signal, cfg as EID space
  LISP source path list
    attached to LISP0
  Covered dependent prefixes: 1
    notify cover updated: 1
  1 IPL source [unresolved]
  no route
```

While the route is still a 'no route' entry, it has appropriate actions set to leak the packet to the CPU and into the LISP process. LISP now processes the packet by looking at its map-cache table:

```
xTR2#show ip lisp map-cache 5.5.5.5
LISP IPv4 Mapping Cache for EID-table default (IID 0), 1 entries

0.0.0.0/0, uptime: 00:47:34, expires: never, via static send map-request
  Sources: static send map-request
  State: send-map-request, last modified: 00:47:34, map-source: local
  Idle, Packets out: 0(0 bytes)
  Configured as EID address space
  Negative cache entry, action: send-map-request 
```

A lookup in the LISP map-caches results in a hit against the 0.0.0.0/0 entry that LISP installs by default. This is a catch-all entry that never expires and its sole purpose is to trigger a Map Request to the MS/MR. 


Having hit this entry, the LISP process on xTR2 now generates a Map Request for EID 5.5.5.5 and sends that to the MS/MR. 

![static1](/static/images/cisco/sda_2/lisp_basic_7.jpg)

The Map Request packet looks like this:

![static1](/static/images/cisco/sda_2/lisp_basic_8.jpg)

The MS/MR looks at its site database to determine if any RLOC has registered this EID:

```
MS_MR#show lisp site 5.5.5.5
LISP Site Registration Information

Site name: SITE_B
Allowed configured locators: any
Requested EID-prefix:
  EID-prefix: 5.5.5.0/24 
    First registered:     00:01:09
    Last registered:      00:00:09
    Routing table tag:    0
    Origin:               Configuration
    Merge active:         No
    Proxy reply:          No
    TTL:                  1d00h
    State:                complete
    Registration errors:  
      Authentication failures:   0
      Allowed locators mismatch: 0
    ETR 4.4.4.4, last registered 00:00:09, no proxy-reply, map-notify
                 TTL 1d00h, no merge, hash-function sha1, nonce 0x7BE12FC9-0xAF5DF791
                 state complete, no security-capability
                 xTR-ID 0xDB225608-0x437E7B02-0xCC1C81E3-0x1E9A0224
                 site-ID unspecified
      Locator  Local  State      Pri/Wgt  Scope
      4.4.4.4  yes    up           1/50   IPv4 none
```

It finds an active entry for this against SITE_B, with a RLOC of 4.4.4.4. However, notice one of the flags that is set against the eTR - no proxy-reply. This means that the MS/MR cannot proxy for this EID and it must forward the Map Request to the RLOC that registered this EID - 4.4.4.4 in this case. 

That is what it does next:

![static1](/static/images/cisco/sda_2/lisp_basic_9.jpg)

When xTR4 gets this, it looks at its database. It is the authoritative owner for this EID so it sends a Map Reply back to the iTR RLOC, 2.2.2.2 in this case.

![static1](/static/images/cisco/sda_2/lisp_basic_10.jpg)

The Map Reply packet looks like this:

![static1](/static/images/cisco/sda_2/lisp_basic_11.jpg)

xTR2 processes the Map Reply and inserts a new, more specific (when compared to 0.0.0.0/0) entry for the EID in its LISP map-cache:

```
xTR2#show ip lisp map-cache 
LISP IPv4 Mapping Cache for EID-table default (IID 0), 2 entries

0.0.0.0/0, uptime: 01:30:31, expires: never, via static send map-request
  Negative cache entry, action: send-map-request
5.5.5.0/24, uptime: 00:00:10, expires: 23:59:49, via map-reply, complete
  Locator  Uptime    State      Pri/Wgt
  4.4.4.4  00:00:10  up           1/50 


xTR2#show ip lisp map-cache 5.5.5.5
LISP IPv4 Mapping Cache for EID-table default (IID 0), 2 entries

5.5.5.0/24, uptime: 00:00:14, expires: 23:59:45, via map-reply, complete
  Sources: map-reply
  State: complete, last modified: 00:00:14, map-source: 4.4.4.4
  Active, Packets out: 0(0 bytes)
  Locator  Uptime    State      Pri/Wgt
  4.4.4.4  00:00:14  up           1/50 
    Last up-down state change:         00:00:14, state change count: 1
    Last route reachability change:    00:00:14, state change count: 1
    Last priority / weight change:     never/never
    RLOC-probing loc-status algorithm:
      Last RLOC-probe sent:            never
```

A lot of good information here - you can see the EID that has been added to the map-cache, along with its RLOC. Outside of this, the source of this was a Map Reply and this entry is valid for the next 23 hours, 59 minutes and 45 seconds. 

Where does this expiry time come from? This is actually specified in the Map Reply itself - it is the 'Record TTL' field. Look closely at the packet - this field has a value of 1440 (in minutes), which is 24 hours.

We're not done yet - once the map-cache has been built, it is pushed down to CEF as well:

```
xTR2#show ip cef 5.5.5.5 detail
5.5.5.0/24, epoch 0, flags [default route handler, subtree context, check lisp eligibility, default route]
  SC owned,sourced: LISP remote EID - locator status bits 0x00000001
  LISP remote EID: 4 packets 400 bytes fwd action encap
  LISP source path list
    nexthop 4.4.4.4 LISP0
  2 IPL sources [unresolved, active source]
    Dependent covered prefix type inherit, cover 0.0.0.0/0
  recursive via 0.0.0.0/0
    no route
```

Remember that earlier, we were hitting the 0.0.0.0/0 entry for the same prefix. Now, a more specific entry has been added for 5.5.5.0/24; subsequent packets hit this instead and do not require any further map requests to be generated. This entry forces a LISP encapsulation of the packet with a next-hop of 4.4.4.4 as you can see from the output above. 


This encapsulated packet is sent towards the MS/MR, since recursively, that is the next hop for 4.4.4.4.

![static1](/static/images/cisco/sda_2/lisp_basic_12.jpg)

The encapsulated packet looks like this:

![static1](/static/images/cisco/sda_2/lisp_basic_13.jpg)


!!! note 
    Look at the source IP address in the outer IP header. It is not 2.2.2.2. This is because the LISP process populates the outer source IP address with the IP address of the egress interface. To change this behavior (and make it more consistent/predictable), configure the command 'ip lisp source-locator <interface>' on each egress LISP interface **

At this point, regular routing takes over (the destination IP address in the outer IP header carries the packet to the destination RLOC) and the packet is forwarded to xTR4. 

![static1](/static/images/cisco/sda_2/lisp_basic_14.jpg)

The destination mac and the destination IP address in the outer IP header is owned by xTR4 itself (4.4.4.4), so it strips off the outer L2 and L3 headers (decapsulates the packet). It parses through the UDP and LISP headers and then does a route lookup against the inner destination IP address. The packet is then forwarded based on this lookup, towards R5.

```
xTR4#show ip route 5.5.5.5
Routing entry for 5.5.5.0/24
  Known via "ospf 1", distance 110, metric 2, type intra area
  Last update from 10.1.45.5 on GigabitEthernet0/0, 02:16:52 ago
  Routing Descriptor Blocks:
  * 10.1.45.5, from 5.5.5.5, 02:16:52 ago, via GigabitEthernet0/0
      Route metric is 2, traffic share count is 1

xTR4#show ip cef 5.5.5.5
5.5.5.0/24
  nexthop 10.1.45.5 GigabitEthernet0/0
```

![static1](/static/images/cisco/sda_2/lisp_basic_15.jpg)

This entire process now happens in the reverse direction, starting from R5. Result? We have reachability between R1s and R5s loopbacks.

```
R1#ping 5.5.5.5 source 1.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 5.5.5.5, timeout is 2 seconds:
Packet sent with a source address of 1.1.1.1 
..!!!
Success rate is 60 percent (3/5), round-trip min/avg/max = 7/7/9 ms
```

Why were two pings lost? One ICMP timeout is how long it took for one map resolution to be completed - this needs to be done in both directions, thus two ICMP requests did not get any responses back.

This concludes a very long post. See you in the next one!