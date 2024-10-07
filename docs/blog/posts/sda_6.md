---
date:
  created: 2021-12-30
  updated: 2021-12-30
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
# Cisco SDA Part VI - LISP mobility - Solicit Map Requests (SMRs)
In this post, we look at SMRs and how these are essential for a host mobility event, within the LISP architecture.

## Introduction and topology

We start this post with the assumption that a host mobility event has occurred (see previous post for details on host mobility) and that the EID 1.1.1.1/24 is moved from behind xTR2 to behind xTR6. 

The state of the topology is like so:

![static1](/static/images/cisco/sda_6/smr_1.jpg)

<!-- more -->

When a mobility event like this occurs, only the MS/MR is notified of this, along with the original RLOC. Remember that LISP works using a pull based model. This implies that the actual datapath remains unchanged on the xTRs even after a mobility event. What this means is that if 5.5.5.5/24 tries to reach 1.1.1.1/24, the data-plane programming still points to xTR2. 

Confirm the same by looking at the LISP map-cache and the CEF table on xTR4:

```
// xTR4s LISP map-cache

xTR4#show ip lisp map-cache 1.1.1.1
LISP IPv4 Mapping Cache for EID-table default (IID 0), 4 entries

1.1.1.1/32, uptime: 00:00:37, expires: 23:59:22, via map-reply, complete
  Sources: map-reply
  State: complete, last modified: 00:00:37, map-source: 2.2.2.2
  Active, Packets out: 0(0 bytes)
  Encapsulating dynamic-EID traffic
  Locator  Uptime    State      Pri/Wgt
  2.2.2.2  00:00:37  up           0/0  
    Last up-down state change:         00:00:37, state change count: 1
    Last route reachability change:    00:00:37, state change count: 1
    Last priority / weight change:     never/never
    RLOC-probing loc-status algorithm:
      Last RLOC-probe sent:            never

// xTR4s CEF table

xTR4#show ip cef 1.1.1.1 detail 
1.1.1.1/32, epoch 0, flags [subtree context, check lisp eligibility]
  SC owned,sourced: LISP remote EID - locator status bits 0x00000001
  LISP remote EID: 19 packets 1900 bytes fwd action encap, dynamic EID need encap
  SC inherited: LISP cfg dyn-EID - LISP configured dynamic-EID
  LISP EID attributes: localEID No, c-dynEID Yes, d-dynEID No
  LISP source path list
    nexthop 2.2.2.2 LISP0
  2 IPL sources [no flags]
  nexthop 2.2.2.2 LISP0
```

## Solicit Map Requests

So, how does the LISP infrastructure inform xTR4 that the EID 1.1.1.1/24 has actually moved and it no longer exists behind xTR2? This is where SMRs (solicit map requests) come into play. 


After the host mobility event, consider the next packet that hits xTR4, destined for 1.1.1.1. Based on the CEF table seen above, xTR4 encapsulates this and sends it to xTR2. 

![static1](/static/images/cisco/sda_6/smr_2.jpg)

When xTR2 receives this, it checks its LISP database, where it finds no entry for this EID. It generates a map request (with the 'S' bit set) and sends that to the source RLOC, xTR4, in this case. This map request is called a SMR (solicit map request). 

![static1](/static/images/cisco/sda_6/smr_3.jpg)

Essentially, what xTR2 is trying to convey is that the destination EID is no longer behind it and the originating RLOC should ask the MS/MR again for information on where this EID is now. 

The SMR packet looks like this (notice that the 'S' bit is set):

![static1](/static/images/cisco/sda_6/smr_4.jpg)

When xTR4 gets this SMR, it generates an encapsulated map request (with the 's' bit set) and sends that to the MS/MR. The MS/MR forwards this to xTR6, since that is the RLOC for 1.1.1.1/32 as registered in its site database. xTR6 sends a Map Reply back to xTR4.

![static1](/static/images/cisco/sda_6/smr_5.jpg)

The Map Request generated on receipt of a SMR looks like this (notice the 's' bit is set; the purpose of the 's' bit is simply to signify that this map request was invoked by the receipt of a SMR):

![static1](/static/images/cisco/sda_6/smr_6.jpg)

Once this map reply is received by xTR4, it now updates its LISP map-cache table (and the corresponding entry in the CEF table as well) to point to xTR6 as the RLOC for this EID. 

```
// xTR4s LISP map-cache

xTR4#show ip lisp map-cache 
LISP IPv4 Mapping Cache for EID-table default (IID 0), 4 entries

0.0.0.0/0, uptime: 00:52:46, expires: never, via static send map-request
  Negative cache entry, action: send-map-request
1.1.1.0/24, uptime: 00:46:52, expires: never, via dynamic-EID, send-map-request
  Negative cache entry, action: send-map-request
1.1.1.1/32, uptime: 00:35:38, expires: 23:58:39, via map-reply, complete
  Locator  Uptime    State      Pri/Wgt
  6.6.6.6  00:01:20  up           0/0  
5.5.5.0/24, uptime: 00:52:46, expires: never, via dynamic-EID, send-map-request

// xTR4s CEF table

xTR4#show ip cef 1.1.1.1 detail
1.1.1.1/32, epoch 0, flags [subtree context, check lisp eligibility]
  SC owned,sourced: LISP remote EID - locator status bits 0x00000001
  LISP remote EID: 147331 packets 14733100 bytes fwd action encap, dynamic EID need encap
  SC inherited: LISP cfg dyn-EID - LISP configured dynamic-EID
  LISP EID attributes: localEID No, c-dynEID Yes, d-dynEID No
  LISP source path list
    nexthop 6.6.6.6 LISP0
  2 IPL sources [no flags]
  nexthop 6.6.6.6 LISP0
```

Post this, traffic destined to 1.1.1.1 now gets directed to xTR6.