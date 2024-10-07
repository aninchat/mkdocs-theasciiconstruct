---
date:
  created: 2021-12-27
  updated: 2021-12-27
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
# Cisco SDA Part III - LISP and non-LISP sites
In this post, we look at how a LISP site talks to non-LISP sites.

## Introduction and topology

Understanding how a LISP site talks to a non-LISP site (and vice versa) is very crucial to LISP and the bigger picture that we're building towards - SDA. 

The topology that we'll work with is a slightly modified version of what we had before - another router has been added that will facilitate conversation between LISP and non-LISP:

![static1](/static/images/cisco/sda_3/non_lisp_1.jpg)

<!-- more -->

This new router, xTR6, connects to another router, R7 which in turn can reach the Internet, where 4.2.2.2 is. R7 is sending a default route to xTR6. The focus of this post is to understand how R1s loopback, 1.1.1.1 (which is in the LISP space) can talk to 4.2.2.2 (which is in the non-LISP space).

To facilitate such a conversation, two additional functionalities were added into LISP - proxy iTR (PiTR) and proxy eTR (PeTR). When a device performs both functions, it is called as a PxTR. Thus, xTR6 in this topology, is a PxTR.

*PiTR* - this box is responsible for being a proxy for EIDs in the LISP space and advertising these EID spaces (via some form of aggregation) into the non-LISP space. The intention of this is to draw traffic for the LISP space towards itself. 
*PeTR* - this box connects the LISP space to non LISP space. It receives encapsulated traffic, decapsulates it and forwards it natively towards non-LISP sites.

To understand how this works, we will look at each direction individually - LISP to non-LISP and non-LISP to LISP.

## LISP to non-LISP 

Configuration for both of these functionalities (PiTR and PeTR) is fairly simple. Let's start off by having xTR6 be a normal xTR. It has the following configuration to begin with:

```
xTR6#show run | sec router lisp
router lisp
 eid-table default instance-id 0
  exit
 !
 ipv4 itr map-resolver 3.3.3.3
 ipv4 itr
 ipv4 etr map-server 3.3.3.3 key cisco
 ipv4 etr
 exit
```

To enable PeTR functionality, simply use 'ipv4 proxy-etr'.

```
xTR6(config)#router lisp
xTR6(config-router-lisp)#ipv4 proxy-etr
xTR6(config-router-lisp)#end
```

xTR6 is now configured to decapsulate LISP encapsulated packets and forward them natively. But how would it receive traffic in the first place, when our destination (4.2.2.2) is not even advertised in the LISP space (there are no database mappings configured for this)? This is where LISP starts to become really cool!

Let's go step by step - R1 sends an ICMP request to xTR2. It hits the 0.0.0.0/0 entry in CEF and subsequently leaks to the LISP process, where it hits the 0.0.0.0/0 entry again (which we should be very familiar with by now). This triggers LISP to generate a map request for 4.2.2.2.

```
// xTR2s 0.0.0.0/0 CEF entry hit

xTR2#show ip cef 4.2.2.2 detail
0.0.0.0/0, epoch 0, flags [default route handler, cover dependents, check lisp eligibility, default route]
  LISP remote EID: 6 packets 600 bytes fwd action signal, cfg as EID space
  LISP source path list
    attached to LISP0
  Covered dependent prefixes: 2
    notify cover updated: 2
  1 IPL source [unresolved]
  no route

// xTR2s 0.0.0.0/0 LISP map-cache entry hit

xTR2#show ip lisp map-cache 4.2.2.2
LISP IPv4 Mapping Cache for EID-table default (IID 0), 1 entries

0.0.0.0/0, uptime: 00:02:17, expires: never, via static send map-request
  Sources: static send map-request
  State: send-map-request, last modified: 00:02:17, map-source: local
  Idle, Packets out: 0(0 bytes)
  Configured as EID address space
  Negative cache entry, action: send-map-request
```

![static1](/static/images/cisco/sda_3/non_lisp_2.jpg)

When the MS/MR receives a map request, it looks at its site database to determine if this has been configured or not and if there are any active RLOCs for this EID.

```
MS_MR#show lisp site 4.2.2.2
% Could not find instance-id 0 EID 4.2.2.2 in site database.
```

If there is an EID miss (which is what happens in this case, as you can see above), it is required to send a negative map reply back with certain action bits set (remember, there are 3 action bits available). One of these action codes is natively-forward, which is what the MS/MR uses here.

![static1](/static/images/cisco/sda_3/non_lisp_3.jpg)

The negative map reply looks like this:

![static1](/static/images/cisco/sda_3/non_lisp_4.jpg)

Notice how the EID in the map reply is not 4.2.2.2/32. Per RFC (RFC 6833), the map resolver "should return the least-specific prefix that both matches the original query and does not match any EID-Prefix known to exist in the LISP-capable infrastructure" in order to minimize the number of negative cache entries needed. 

When xTR2 processes this map reply, it installs this into the LISP map-cache. From here, it gets pushed into the CEF table as well.

```
xTR2#show ip lisp map-cache 
LISP IPv4 Mapping Cache for EID-table default (IID 0), 3 entries

0.0.0.0/0, uptime: 12:04:44, expires: never, via static send map-request
  Negative cache entry, action: send-map-request
4.0.0.0/8, uptime: 00:14:57, expires: 00:14:54, via map-reply, forward-native
  Encapsulating to proxy ETR
5.5.5.0/24, uptime: 11:20:41, expires: 12:41:55, via map-reply, complete
  Locator  Uptime    State      Pri/Wgt
  4.4.4.4  11:18:04  up           1/50

xTR2#show ip cef 4.2.2.2 detail
4.0.0.0/8, epoch 0, flags [default route handler, subtree context, check lisp eligibility, default route]
  SC owned,sourced: LISP remote EID - locator status bits 0x00000000
  LISP remote EID: 4 packets 400 bytes fwd action fwd native via petr
  LISP source path list
    nexthop 6.6.6.6 LISP0
  2 IPL sources [unresolved, active source]
    Dependent covered prefix type inherit, cover 0.0.0.0/0
  recursive via 0.0.0.0/0
    no route 
```

Whoa - why does this entry say that it needs to be encapsulated to a 'proxy eTR'? I need to pull back the curtain a little bit now - prior to this, I had configured an additional command on xTR2 which wasn't show earlier:

```
xTR2(config)#router lisp
xTR2(config-router-lisp)#ipv4 use-petr 6.6.6.6
xTR2(config-router-lisp)#end
```

This allows all negative cache entries with forward-native set to be sent to a particular IP address (as configured in the command). In our case, this IP address is 6.6.6.6, which connects to the non-LISP site. This IP address, in general, is typically set to point to your PeTR.

So, xTR2 can now encapsulate subsequent packets with a destination IP address of 6.6.6.6 (in the outer IP header) and forward it on.

![static1](/static/images/cisco/sda_3/non_lisp_5.jpg)

xTR6 receives this, decapsulates it and does another CEF lookup on the inner IP header. This hits the 0.0.0.0/0 entry which leaks the packet to LISP, where it hits the 0.0.0.0/0 entry again, causing a map request to be sent to the MS/MR for the destination IP address, 4.2.2.2:

```
// xTR6s 0.0.0.0/0 CEF entry hit

xTR6#show ip cef 4.2.2.2 detail
0.0.0.0/0, epoch 0, flags [cover dependents, check lisp eligibility, default route]
  LISP remote EID: 1 packets 0 bytes fwd action signal, cfg as EID space
  LISP source path list
    attached to LISP0
  Covered dependent prefixes: 1
    notify cover updated: 1
  1 IPL source [no flags]
  attached to LISP0

// xTR6s 0.0.0.0/0 LISP map-cache entry hit

xTR6#show ip lisp map-cache 4.2.2.2
LISP IPv4 Mapping Cache for EID-table default (IID 0), 2 entries

0.0.0.0/0, uptime: 00:31:20, expires: never, via static send map-request
  Sources: static send map-request
  State: send-map-request, last modified: 00:31:20, map-source: local
  Idle, Packets out: 1(100 bytes) (~ 00:20:19 ago)
  Configured as EID address space
  Negative cache entry, action: send-map-request
```

!!! note
    I did have to enable PiTR functionality for this part to work. Without this, we run into other problems (such as routing loops) which I get to later in the post **

This can be visualized as below:

![static1](/static/images/cisco/sda_3/non_lisp_6.jpg)

Again, the MS/MR responds back with a negative map reply and the native forward bit set, since it does not have any EIDs that match 4.2.2.2 in its site database:

![static1](/static/images/cisco/sda_3/non_lisp_7.jpg)

xTR6 processes this map reply (remember that the MS/MR responds back with the least specific prefix, which in this case was 4.0.0.0/8) and gleans that this prefix needs to be natively forwarded. The next hop now gets picked up from the best match in RIB/FIB (which is the next hop for the default route received from R7). 

```
// xTR6 sends a map request to the MS/MR for 4.2.2.2/32

*Jun  7 00:57:19.419: LISP: Send map request for EID prefix IID 0 4.2.2.2/32
*Jun  7 00:57:19.419: LISP-0: Remote EID IID 0 prefix 4.2.2.2/32, Send map request (1) (sources: <signal>, state: incomplete, rlocs: 0).
*Jun  7 00:57:19.419: LISP-0: EID-AF IPv4, Sending map-request from 4.2.2.2 to 4.2.2.2 for EID 4.2.2.2/32, ITR-RLOCs 1, nonce 0x7798929C-0x4F5ECE13 (encap src 10.1.36.6, dst 3.3.3.3), FromPITR.

// xTR6 starts processing map reply

*Jun  7 00:57:19.424: LISP: Processing received Map-Reply(2) message on GigabitEthernet0/0 from 10.1.36.3:4342 to 6.6.6.6:4342
*Jun  7 00:57:19.425: LISP: Received map reply nonce 0x7798929C-0x4F5ECE13, records 1
*Jun  7 00:57:19.425: LISP: Processing Map-Reply mapping record for IID 0 4.0.0.0/8, ttl 15, action forward, authoritative, 0 locators
*Jun  7 00:57:19.425: LISP-0: Map Request IID 0 prefix 4.2.2.2/32 remote EID prefix[LL], Received reply with rtt 6ms.
*Jun  7 00:57:19.425: LISP: Processing mapping information for EID prefix IID 0 4.0.0.0/8
*Jun  7 00:57:19.426: LISP-0: AF IID 0 IPv4, Persistent db: ignore writing request, disabled.
*Jun  7 00:57:19.426: LISP-0: Remote EID IID 0 prefix 4.0.0.0/8, Change state to forward-native (sources: map-rep, state: unknown, rlocs: 0).
*Jun  7 00:57:19.426: LISP-0: Remote EID IID 0 prefix 4.0.0.0/8, Starting idle timer (delay 00:02:30) (sources: map-rep, state: forward-native, rlocs: 0).
*Jun  7 00:57:19.426: LISP-0: Remote EID IID 0 prefix 4.2.2.2/32, Change state to deleted (sources: <>, state: incomplete, rlocs: 0).
*Jun  7 00:57:19.427: LISP-0: Remote EID IID 0 prefix 4.2.2.2/32, Map-reply from 10.1.36.3 returned less specific 4.0.0.0/8 (sources: <>, state: deleted, rlocs: 0).

// next-hop pulled from an internal CEF walk

*Jun  7 00:57:19.427: LISPreid: Default:4.0.0.0/8 Add, action fwd native, lsb 0x0
*Jun  7 00:57:19.427: LISPreid: Default:0.0.0.0/0 Null modify of pco 0xF296E08 linked to glean for LISP0
*Jun  7 00:57:19.428: LISPreid: Default:4.0.0.0/8 Added LISP IPL src, ok
*Jun  7 00:57:19.428: LISPreid: Default:4.0.0.0/8 Created pco 0x1064C550 linked to IP adj out of GigabitEthernet0/1, addr 10.1.67.7 0F8B85F8
*Jun  7 00:57:19.429: LISPreid: Default:0.0.0.0/0 Null modify of pco 0xF296E08 linked to glean for LISP0
*Jun  7 00:57:19.429: LISPreid: Default:4.0.0.0/8 Added LISP_FWD_NATIVE src, success
*Jun  7 00:57:19.430: LISPreid: Default:4.2.2.2/32 Remove, action not specd, lsb 0x0, match found
*Jun  7 00:57:19.430: LISPreid: Default:4.0.0.0/8 Null modify of pco 0x1064C550 linked to IP adj out of GigabitEthernet0/1, addr 10.1.67.7 0F8B85F8
*Jun  7 00:57:19.430: LISPreid: Default:4.2.2.2/32 Removed LISP src, success
*Jun  7 00:57:19.431: LISPreid: Default:4.2.2.2/32 Removing LISP IPL src
*Jun  7 00:57:19.438: LISPreid: Default:4.0.0.0/8 Null modify of pco 0x1064C550 linked to IP adj out of GigabitEthernet0/1, addr 10.1.67.7 0F8B85F8
*Jun  7 00:57:19.439: LISPreid: Default:4.2.2.2/32 Removed LISP subtree, success
```

The end result?

```
xTR6#show ip cef 4.2.2.2 detail
4.0.0.0/8, epoch 0, flags [check lisp eligibility]
  LISP remote EID: 3 packets 300 bytes fwd action fwd native
  LISP fwd-native source
    Dependent covered prefix type LISP-FWD, cover 0.0.0.0/0
  1 IPL source [no flags]
  nexthop 10.1.67.7 GigabitEthernet0/1
```

Subsequent packets for 4.2.2.2 now hit the above entry and get forwarded to R7: 

![static1](/static/images/cisco/sda_3/non_lisp_8.jpg)

This concludes our discussion on how packets get forwarded from a LISP site to a non-LISP site. 

## Non-LISP to LISP

What about the reverse direction though (non-LISP to LISP)? This is where things get even more interesting and a very important rule of LISP is revealed. 

So, in the reverse path, R7 forwards the ICMP reply from 4.2.2.2 to xTR6. Just like before, xTR6 does a CEF lookup for the destination (1.1.1.1). However, whenever a box is enabled for iTR functionality, it also does a lookup in the local LISP database to check whether the source is a locally known EID or not. If not, the LISP process is not invoked at all. Instead, traditional forwarding would happen without leaking any packets to the LISP process.


Currently, xTR6 is not configured to be a PiTR (remember, we had only configured the command for PeTR) - it behaves as an iTR as of now. So, technically, we can potentially create a routing loop here to prove what I just said. What is our setup like? xTR6 is getting a default route from R7 (just like a typical enterprise design):

```
xTR6#show ip cef 1.1.1.1 detail
0.0.0.0/0, epoch 0, flags [check lisp eligibility, default route]
  LISP remote EID: 0 packets 0 bytes fwd action signal, cfg as EID space
  LISP source path list
    attached to LISP0
  1 IPL source [no flags]
  recursive via 10.1.67.7
    attached to GigabitEthernet0/1
```

Does xTR6 have the source in its LISP database as a local EID? No, it does not.

```
xTR6#show ip lisp forwarding eid local 4.2.2.2
Prefix
% No local EID prefix in IPv4:Default matching 4.2.2.2
```

This means that even though we're hitting the 0.0.0.0/0 entry in CEF and it is associated to LISP0 (so that packets can leak to the LISP process), it will only consider the next-hop from RIB/FIB and not leak packets to LISP. This causes xTR6 to send the packet back to R7. R7 sends it back to xTR6 again and a routing loop ensues. 

![static1](/static/images/cisco/sda_3/non_lisp_9.jpg)


To bypass this check and have a device be a proxy for EIDs in the LISP space, you configure it as a PiTR. Additionally, you must remember that as a PiTR, there is no default 0.0.0.0/0 entry in the LISP map-cache (unlike a iTR where this is present by default). 

This implies that you must create some manual map cache entries. This is done using the 'map-cache' commands. 

```
// configure a default catch-all map-cache entry

xTR6(config)#router lisp
xTR6(config-router-lisp)#eid-table default instance-id 0
xTR6(config-router-lisp-eid-table)#map-cache 0.0.0.0/0 map-request
xTR6(config-router-lisp-eid-table)#exit

// note that PiTR cannot be configured until iTR is disabled
// a box cannot have both roles at the same time

xTR6(config-router-lisp)#ipv4 proxy-itr 6.6.6.6
% Disable ITR functionality first.

xTR6(config-router-lisp)#no ipv4 itr

// the IP address below acts as a source-locator
// packets encap'd by this PiTR have a source of 6.6.6.6

xTR6(config-router-lisp)#ipv4 proxy-itr 6.6.6.6
xTR6(config-router-lisp)#end
```

Now, we should be able to reach 4.2.2.2 from R1s loopback, 1.1.1.1:

```
R1#ping 4.2.2.2 source 1.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 4.2.2.2, timeout is 2 seconds:
Packet sent with a source address of 1.1.1.1 
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 5/8/12 ms
```

Let's quickly take a look at the entire end to end process in this direction of traffic flow (non-LISP to LISP).

R7 sends the ICMP reply from 4.2.2.2 to xTR6 (typically, the PiTR is configured to aggregate its EID spaces and send that to its peer so as to draw traffic towards itself for the LISP space. However, for the sake of simplicity, I have configured a static route on R7 for 1.1.1.0/24 that points to xTR6). 

![static1](/static/images/cisco/sda_3/non_lisp_10.jpg)

This reaches xTR6, which is now configured to be a PiTR. It bypasses the source EID lookup, does a CEF lookup, which leaks the packet to LISP, where it hits the 0.0.0.0/0 entry we had manually configured:

```
// xTR6s CEF lookup for 1.1.1.1

xTR6#show ip cef 1.1.1.1 detail
0.0.0.0/0, epoch 0, flags [check lisp eligibility, default route]
  LISP remote EID: 0 packets 0 bytes fwd action signal, cfg as EID space
  LISP source path list
    attached to LISP0
  1 IPL source [no flags]
  attached to LISP0

// xTR6s LISP map-cache lookup for 1.1.1.1  

xTR6#show ip lisp map-cache 1.1.1.1
LISP IPv4 Mapping Cache for EID-table default (IID 0), 1 entries

0.0.0.0/0, uptime: 00:05:50, expires: never, via static send map-request
  Sources: static send map-request
  State: send-map-request, last modified: 00:05:50, map-source: local
  Idle, Packets out: 0(0 bytes)
  Configured as EID address space
  Negative cache entry, action: send-map-request
```

This triggers a map request for 1.1.1.1 that is sent to the MS/MR:

![static1](/static/images/cisco/sda_3/non_lisp_11.jpg)

The MS/MR looks at its site database to see if this EID is configured and if any RLOC has registered this. It finds a hit with 2.2.2.2 as the RLOC:

```
MS_MR#show lisp site 1.1.1.1
LISP Site Registration Information

Site name: SITE_A
Allowed configured locators: any
Requested EID-prefix:
  EID-prefix: 1.1.1.0/24 
    First registered:     00:03:04
    Last registered:      00:00:06
    Routing table tag:    0
    Origin:               Configuration
    Merge active:         No
    Proxy reply:          No
    TTL:                  1d00h
    State:                complete
    Registration errors:  
      Authentication failures:   0
      Allowed locators mismatch: 0
    ETR 2.2.2.2, last registered 00:00:06, no proxy-reply, map-notify
                 TTL 1d00h, no merge, hash-function sha1, nonce 0xD799F064-0x5BB5D363
                 state complete, no security-capability
                 xTR-ID 0xF1FF77A5-0x014680A5-0xA290EB87-0x388BA5CE
                 site-ID unspecified
      Locator  Local  State      Pri/Wgt  Scope
      2.2.2.2  yes    up           1/50   IPv4 none
```

It forwards this request to xTR2. 

![static1](/static/images/cisco/sda_3/non_lisp_12.jpg)

xTR2, in turn, sends a map reply back to xTR6 after processing it.

![static1](/static/images/cisco/sda_3/non_lisp_13.jpg)

xTR6 processes this map reply and inserts the prefix in the LISP map-cache and pushes that to the CEF table as well:

```
// xTR6s LISP map-cache for 1.1.1.1

xTR6#show ip lisp map-cache 1.1.1.1
LISP IPv4 Mapping Cache for EID-table default (IID 0), 3 entries

1.1.1.0/24, uptime: 00:06:41, expires: 23:53:18, via map-reply, complete
  Sources: map-reply
  State: complete, last modified: 00:06:41, map-source: 2.2.2.2
  Idle, Packets out: 2(200 bytes) (~ 00:06:38 ago)
  Locator  Uptime    State      Pri/Wgt
  2.2.2.2  00:06:41  up           1/50 
    Last up-down state change:         00:06:41, state change count: 1
    Last route reachability change:    00:06:41, state change count: 1
    Last priority / weight change:     never/never
    RLOC-probing loc-status algorithm:
      Last RLOC-probe sent:            never

// xTR6s CEF table for 1.1.1.1      

xTR6#show ip cef 1.1.1.1 detail
1.1.1.0/24, epoch 0, flags [subtree context, check lisp eligibility]
  SC owned,sourced: LISP remote EID - locator status bits 0x00000001
  LISP remote EID: 2 packets 200 bytes fwd action encap
  LISP source path list
    nexthop 2.2.2.2 LISP0
  2 IPL sources [no flags]
  nexthop 2.2.2.2 LISP0
```

xTR6 can now encapsulate subsequent packets and send them to xTR2.

![static1](/static/images/cisco/sda_3/non_lisp_14.jpg)

xTR2 decapsulates them and sends them to R1. 

![static1](/static/images/cisco/sda_3/non_lisp_15.jpg)
