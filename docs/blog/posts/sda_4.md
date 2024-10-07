---
date:
  created: 2021-12-28
  updated: 2021-12-28
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
# Cisco SDA Part IV - LISP mobility - dynamic EIDs
In this post, we look at LISP dynamic EID - a core construct of LISP host mobility.

## Introduction and topology

One of the most important characteristics of LISP is the mobility it offers - the next few posts aim at helping understand how this functionality is achieved, starting with dynamic EIDs.

We will continue using the same topology as before, with some minor changes to the xTRs. xTR6 is now another xTR and not a PxTR.

![static1](/static/images/cisco/sda_4/dynamic_eid_1.jpg)

<!-- more -->

The goal is this post is to understand how dynamic EIDs are configured and how does it really function behind the scenes. Our final test is to have complete connectivity between 1.1.1.1 and 5.5.5.5. Our initial (and relevant LISP configuration) is pretty standard:

```
// xTR2

xTR2#show run | sec router lisp
router lisp
 eid-table default instance-id 0
  exit
 !
 ipv4 itr map-resolver 3.3.3.3
 ipv4 itr
 ipv4 etr map-server 3.3.3.3 key cisco
 ipv4 etr
 exit

// xTR4 

xTR4#show run | sec router lisp
router lisp
 eid-table default instance-id 0
  exit
 !
 ipv4 itr map-resolver 3.3.3.3
 ipv4 itr
 ipv4 etr map-server 3.3.3.3 key cisco
 ipv4 etr
 exit

// xTR6

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

## Dynamic EIDs

### Configuring dynamic EIDs


With dynamic EIDs, the first thing you realize is that you have to use locator sets. Think of locator sets as a parallel to peer groups in BGP - you can use one locator set, that uniquely identifies a RLOC and its attributes, and apply that to several instances of LISP. 

```
xTR2(config-router-lisp)#locator-set xTR2 
xTR2(config-router-lisp-locator-set)#ipv4-interface lo0 priority 1 weight 50
xTR2(config-router-lisp-locator-set)#end
```




Before we create the dynamic EID, take a quick look at the LISP map-cache:

```
xTR2#show ip lisp map-cache
LISP IPv4 Mapping Cache for EID-table default (IID 0), 1 entries

0.0.0.0/0, uptime: 00:05:24, expires: never, via static send map-request
  Negative cache entry, action: send-map-request
```




As expected, there's just a single, default entry in the LISP cache. Let's configure a dynamic EID now:

```
xTR2(config-router-lisp)#eid-table default instance-id 0
xTR2(config-router-lisp-eid-table)#dynamic-eid 1.1.1.0/24_EID
xTR2(config-router-lisp-eid-table-dynamic-eid)#database-mapping 1.1.1.0/24 locator-set xTR2 
xTR2(config-router-lisp-eid-table-dynamic-eid)#end
```




You create a dynamic EID using the 'dynamic-eid' CLI and give it a name that can be referenced later. Within this dynamic EID, you need to create your database mappings - this dictates the prefix range that is assumed to be dynamic (or 'roaming') in nature.


This creates a new entry in the LISP map-cache:

```
xTR2#show ip lisp map-cache
LISP IPv4 Mapping Cache for EID-table default (IID 0), 2 entries

0.0.0.0/0, uptime: 00:09:04, expires: never, via static send map-request
  Negative cache entry, action: send-map-request
1.1.1.0/24, uptime: 00:01:32, expires: never, via dynamic-EID, send-map-request
  Negative cache entry, action: send-map-request
```




But notice, as of now, the MS/MR has no registrations against this EID:

```
MS_MR#show lisp site 1.1.1.0/24
LISP Site Registration Information

Site name: SITE_A
Allowed configured locators: any
Requested EID-prefix:
  EID-prefix: 1.1.1.0/24 
    First registered:     00:55:01
    Last registered:      00:10:22
    Routing table tag:    0
    Origin:               Configuration
    Merge active:         No
    Proxy reply:          No
    TTL:                  00:00:00
    State:                unknown
    Registration errors:  
      Authentication failures:   0
      Allowed locators mismatch: 0
    No registrations.
```




At this point, if I try to ping from R1 to xTR2 (sourcing an IP address of 1.1.1.1), nothing works and we get this error message:

```
LISP: Processing data signal for EID prefix IID 0 1.1.1.1/32
LISP-0: Remote EID IID 0 prefix 1.1.1.0/24, Process data signal, matching remote dynamic EID entry (sources: <dynEID>, state: send-map-request, rlocs: 0).
LISP-0: Remote EID IID 0 prefix 1.1.1.1/32, Change state to incomplete (sources: <signal>, state: unknown, rlocs: 0).
LISP-0: Remote EID IID 0 prefix 1.1.1.1/32, [incomplete] Scheduling map requests delay 00:00:00 min_elapsed 00:00:01 (sources: <signal>, state: incomplete, rlocs: 0).
LISP-0: IID 0 No ITR RLOCs available, do not process remote EID prefix map requests.
```




So, what is missing? With dynamic EIDs, you need to configure the host facing L3 interfaces as LISP mobility interfaces as well for this to work:

```
xTR2(config)#int gig0/0
xTR2(config-if)#lisp mobility 1.1.1.0/24_EID
```




This is where you call the dynamic EID that was defined at the beginning. And that's all there is to it! That is the complete configuration you need to enable dynamic EIDs in LISP on xTRs. There is a small catch though, which we'll get to in a bit. 

### Learning EIDs dynamically

Now that we understand how to configure dynamic EIDs, the big question remains - how are EIDs learnt dynamically? The answer is simple - via traffic. This can either be control-plane traffic (like ARP) or data-plane traffic (like an ICMP request). 


When said traffic hits a xTR that is enabled for dynamic EIDs and it falls within that prefix range, the dynamic registration process starts. This includes sending a Map Register to the MS/MR as well as adding an exact match entry in the LISP database and RIB/FIB.

![static1](/static/images/cisco/sda_4/dynamic_eid_2.jpg)





The Map Register process is the same as before - nothing special here. xTR2 generates a Map Register packet and sends that to the configured Map Server (MS/MR, in our case). The MS/MR looks at its site database and confirms that an EID exists in its database that matches the EID in the register and then sends a Map Notify back. 



Interestingly enough though, we see no Map Notify coming back from the MS/MR right now:

![static1](/static/images/cisco/sda_4/dynamic_eid_3.jpg)




Any guesses on why? Let's walk through the entire flow and understand why this happened. xTR2 gets a data packet (or an ARP) from R1 which has a source IP address of 1.1.1.1 and a destination IP address of 10.1.12.2. This triggers the LISP dynamic EID learn, which invokes an exact route installation in the LISP database and RIB/FIB. 


We can confirm that both of these occurred correctly:

```
// RIB table

xTR2#show ip route 1.1.1.1
Routing entry for 1.1.1.1/32
  Known via "lisp", distance 10, metric 1, type unknown
  Last update from 1.1.1.1 on GigabitEthernet0/0, 00:09:36 ago
  Routing Descriptor Blocks:
  * 1.1.1.1, from 0.0.0.0, 00:09:36 ago, via GigabitEthernet0/0
      Route metric is 1, traffic share count is 1

// LISP database

xTR2#show ip lisp database 
LISP ETR IPv4 Mapping Database for EID-table default (IID 0), LSBs: 0x1, 1 entries

1.1.1.1/32, dynamic-eid 1.1.1.0/24_EID, locator-set xTR2
  Locator  Pri/Wgt  Source     State
  2.2.2.2    0/0    cfg-intf   site-self, reachable 
```




Additionally, a Map Register is sent to the MS/MR. Does the MS/MR have a match for this EID in its site database?

```
MS_MR#show lisp site 1.1.1.1   
LISP Site Registration Information

Site name: SITE_A
Allowed configured locators: any
Requested EID-prefix:
  EID-prefix: 1.1.1.0/24 
    First registered:     01:19:16
    Last registered:      00:34:37
    Routing table tag:    0
    Origin:               Configuration
    Merge active:         No
    Proxy reply:          No
    TTL:                  00:00:00
    State:                unknown
    Registration errors:  
      Authentication failures:   0
      Allowed locators mismatch: 0
    No registrations.
```





Yes, it does. However, pay close attention to the output - we are hitting the 1.1.1.0/24 entry and naturally, there are no registrations against this EID so it cannot send a Map Notify back. 



This is one of the biggest gotchas with dynamic EIDs which leads me to one of the most important changes needed to make dynamic EIDs work - you must allow for more specific prefixes to be accepted on the MS/MR. 


It is a very simple, but crucial change:

```
MS_MR(config)#router lisp
MS_MR(config-router-lisp)#site SITE_A
MS_MR(config-router-lisp-site)#eid-prefix 1.1.1.0/24 accept-more-specifics
MS_MR(config-router-lisp-site)#end
```




Now, when the Map Register is sent again, the MS/MR is willing to accept a more specific prefix under the range of 1.1.1.0/24 and it sends the corresponding Map Notify back:

![static1](/static/images/cisco/sda_4/dynamic_eid_4.jpg)




The MS/MR also has this entry in its site database now:

```
MS_MR#show lisp site 1.1.1.1
LISP Site Registration Information

Site name: SITE_A
Allowed configured locators: any
Requested EID-prefix:
  EID-prefix: 1.1.1.1/32 
    First registered:     00:04:23
    Last registered:      00:00:24
    Routing table tag:    0
    Origin:               Dynamic, more specific of 1.1.1.0/24
    Merge active:         No
    Proxy reply:          No
    TTL:                  1d00h
    State:                complete
    Registration errors:  
      Authentication failures:   0
      Allowed locators mismatch: 0
    ETR 2.2.2.2, last registered 00:00:24, no proxy-reply, map-notify
                 TTL 1d00h, no merge, hash-function sha1, nonce 0x3737C5D8-0x6F651C53
                 state complete, no security-capability
                 xTR-ID 0xA55F2BD8-0xA072A7CF-0xDCD31112-0xC826C03F
                 site-ID unspecified
      Locator  Local  State      Pri/Wgt  Scope
      2.2.2.2  yes    up           0/0    IPv4 none
```




As you can see, this is an entirely new entry that was dynamically added to the site database of the MS/MR because we allowed it to learn more specific prefixes within the scope of 1.1.1.0/24. 

### Completing our configuration

Let's complete our configuration for the 5.5.5.0/24 EID space as well. This requires three changes - first, on xTR4, configure this EID as a dynamic EID range. Second, on the MS/MR, allow for more specific prefixes to be learnt for this EID space. Lastly, on the interface facing the host, enable LISP mobility against the configured dynamic EID. 

```
// configure a locator-set for xTR4

xTR4(config-router-lisp)#locator-set xTR4
xTR4(config-router-lisp-locator-set)#ipv4-interface lo0

// configure dynamic EID for 5.5.5.0/24

xTR4(config-router-lisp)#eid-table default instance-id 0
xTR4(config-router-lisp-eid-table)#dynamic-eid 5.5.5.0/24_EID
xTR4(config-router-lisp-eid-table-dynamic-eid)#database-mapping 5.5.5.0/24 locator-set xTR4

// configure interface gig0/0 for LISP mobility

xTR4(config-router-lisp-eid-table)#int gig0/0
xTR4(config-if)#lisp mobility 5.5.5.0/24_EID  

// configure MS/MR to allow more specifics for 5.5.5.0/24 range

MS_MR(config)#router lisp
MS_MR(config-router-lisp)#site SITE_B
MS_MR(config-router-lisp-site)#eid-prefix 5.5.5.0/24 accept-more-specifics
MS_MR(config-router-lisp-site)#end
```




Let's trigger a dynamic EID learn for the 5.5.5.0/24 range and ensure that an entry gets added into the MS/MR site database. 

```
R5#ping 10.1.45.4 source 5.5.5.5
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.1.45.4, timeout is 2 seconds:
Packet sent with a source address of 5.5.5.5 
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 4/4/5 ms
```




A look at xTR4s LISP database and the MS/MRs site database to confirm the expected entries were added:

```
// xTR4s LISP database

xTR4#show ip lisp database  
LISP ETR IPv4 Mapping Database for EID-table default (IID 0), LSBs: 0x1, 1 entries

5.5.5.5/32, dynamic-eid 5.5.5.0/24_EID, locator-set xTR4
  Locator  Pri/Wgt  Source     State
  4.4.4.4    0/0    cfg-intf   site-self, reachable

// MS/MRs site database  

MS_MR#show lisp site 5.5.5.5
LISP Site Registration Information

Site name: SITE_B
Allowed configured locators: any
Requested EID-prefix:
  EID-prefix: 5.5.5.5/32 
    First registered:     00:00:24
    Last registered:      00:00:24
    Routing table tag:    0
    Origin:               Dynamic, more specific of 5.5.5.0/24
    Merge active:         No
    Proxy reply:          No
    TTL:                  1d00h
    State:                complete
    Registration errors:  
      Authentication failures:   0
      Allowed locators mismatch: 0
    ETR 4.4.4.4, last registered 00:00:24, no proxy-reply, map-notify
                 TTL 1d00h, no merge, hash-function sha1, nonce 0xC5BB43A9-0x4B85B722
                 state complete, no security-capability
                 xTR-ID 0xF1E43ADA-0x5BB5F6AE-0xAE8DC820-0x4AE98A3E
                 site-ID unspecified
      Locator  Local  State      Pri/Wgt  Scope
      4.4.4.4  yes    up           0/0    IPv4 none
```



At this point, we should have complete reachability from 1.1.1.1 to 5.5.5.5, which we do. The data-plane process is the same as normal EIDs - nothing new there.

```
R1#ping 5.5.5.5 source 1.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 5.5.5.5, timeout is 2 seconds:
Packet sent with a source address of 1.1.1.1 
..!!!
Success rate is 60 percent (3/5), round-trip min/avg/max = 7/10/14 ms
```


This completes our section of understanding dynamic EIDs. In the next post, we will look at host mobility and what happens when a host moves from one RLOC to another. See you in the next one!