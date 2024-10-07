---
date:
  created: 2021-12-29
  updated: 2021-12-29
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
# Cisco SDA Part V - LISP mobility - roaming hosts
In this post, we look at an actual LISP host mobility event and what happens behind the scenes to make this work.

## Introduction and topology

Continuing on from the previous post, we take a look at actual host mobility events and how the LISP infrastructure facilitates this. Our goal for this post is to have the simulated host (1.1.1.1) move from behind xTR2 to behind xTR4 (simulated via R10). A working assumption used in the post is that there is no active traffic destined for the host that is moving (we will look at this in the SMR post).

The topology is a slightly modified version of what we used in the last post:

![static1](/static/images/cisco/sda_5/lisp_mobility_1.jpg)

<!-- more -->

There are certain baselines we need to establish from the get go - 1.1.1.1/32 has been learnt dynamically and installed in the site database of the MS/MR, with a RLOC of xTR2 (2.2.2.2). Let's confirm this:

```
MS_MR#show lisp site 1.1.1.1
LISP Site Registration Information

Site name: SITE_A
Allowed configured locators: any
Requested EID-prefix:
  EID-prefix: 1.1.1.1/32 
    First registered:     00:19:38
    Last registered:      00:00:56
    Routing table tag:    0
    Origin:               Dynamic, more specific of 1.1.1.0/24
    Merge active:         No
    Proxy reply:          No
    TTL:                  1d00h
    State:                complete
    Registration errors:  
      Authentication failures:   0
      Allowed locators mismatch: 0
    ETR 2.2.2.2, last registered 00:00:56, no proxy-reply, map-notify
                 TTL 1d00h, no merge, hash-function sha1, nonce 0xC6805C87-0xF287CBA7
                 state complete, no security-capability
                 xTR-ID 0x8527C2E3-0xB036F499-0x0C2893BB-0x56D43540
                 site-ID unspecified
      Locator  Local  State      Pri/Wgt  Scope
      2.2.2.2  yes    up           0/0    IPv4 none
```

On xTR2, this entry is also present in its LISP database:

```
xTR2#show ip lisp database 
LISP ETR IPv4 Mapping Database for EID-table default (IID 0), LSBs: 0x1, 1 entries

1.1.1.1/32, dynamic-eid 1.1.1.0/24_EID, locator-set xTR2
  Locator  Pri/Wgt  Source     State
  2.2.2.2    0/0    cfg-intf   site-self, reachable
```

Additionally, xTR4 needs to be provisioned for 1.1.1.0/24 as a dynamic EID:

```
xTR4(config)#router lisp
xTR4(config-router-lisp)#eid-table default instance-id 0
xTR4(config-router-lisp-eid-table)#dynamic-eid 1.1.1.0/24_EID
xTR4(config-router-lisp-eid-table-dynamic-eid)#database-mapping 1.1.1.0/24 locator-set xTR4
```

## LISP mobility event

So, now that we have these baselines established, let's move 1.1.1.1 to R10. 

![static1](/static/images/cisco/sda_5/lisp_mobility_2.jpg)

When this happens, some traffic (can be control-plane or data-plane) is generated from 1.1.1.1 and it hits xTR4. xTR4 dynamically learns this, installs it in its LISP database and sends a Map Register to the MS/MR. The MS/MR accepts this and installs it in its site database. 

![static1](/static/images/cisco/sda_5/lisp_mobility_3.jpg)

At the same time, the MS/MR knew that there was a previously registered RLOC against this specific /32 entry. It informs the original RLOC (xTR2, in this case) of this new RLOC learn by sending it a Map Notify as well. 

![static1](/static/images/cisco/sda_5/lisp_mobility_4.jpg)

This Map Notify is as follows:

![static1](/static/images/cisco/sda_5/lisp_mobility_5.jpg)

When xTR2 receives this Map Notify, it realizes that the host has "moved away". It deletes the /32 entry from its LISP database and sends a Map Register back to the MS/MR with an action of 'Drop'. 

![static1](/static/images/cisco/sda_5/lisp_mobility_6.jpg)

This packet looks like so:

![static1](/static/images/cisco/sda_5/lisp_mobility_7.jpg)

On receiving this Map Register, the MS/MR removes xTR2 as a RLOC for 1.1.1.1/32. This action completes the host move. 