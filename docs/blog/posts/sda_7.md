---
date:
  created: 2022-01-01
  updated: 2022-01-01
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
# Cisco SDA Part VII - multi-instance LISP
In this post, we look at multi-instance LISP, which is another core construct for Cisco's SD-Access.

## Introduction and topology

We're slowly getting closer to the true implementation of LISP in Cisco's SD-Access. LISP has the capability of being VRF-aware - this is achieved via multi-instance LISP. 

The idea is fairly simple - you have multiple instances of LISP (mapped to corresponding VRFs) - all your LISP tables are now maintained per instance. 

We will be using the following topology for this:

![static1](/static/images/cisco/sda_7/multi_instance_lisp_1.jpg)

<!-- more -->

## VRF aware configuration

We have moved R1 and R5 into a VRF called 'TEST'. This is done by assigning the port facing R1 on xTR2 to this VRF:

```
xTR2#show run int gig0/0
Building configuration...

Current configuration : 246 bytes
!
interface GigabitEthernet0/0
 vrf forwarding TEST
 ip address 10.1.12.2 255.255.255.0
 no ip redirects
 ip ospf network point-to-point
 duplex auto
 speed auto
 media-type rj45
 no lisp mobility liveness test
 lisp mobility 1.1.1.0/24_EID
end
```

Similar configuration is done on xTR4 as well. To make LISP VRF aware, you need to create instances of LISP and map the instance to a particular VRF. All your database mappings will now come under this instance-ID.

For this example, we will create an instance-ID of 100 and map the VRF 'TEST' to this instance-ID.

```
xTR2(config)#router lisp
xTR2(config-router-lisp)#eid-table vrf TEST instance-id 100
xTR2(config-router-lisp-eid-table)#database-mapping 1.1.1.0/24 locator-set xTR2
```

Similar configuration is done on xTR4 as well:

```
xTR4(config)#router lisp
xTR4(config-router-lisp)# eid-table vrf TEST instance-id 100
xTR4(config-router-lisp-eid-table)#database-mapping 5.5.5.0/24 locator-set xTR4
```

All of these database mappings will now be per instance-ID. They will no longer show up in the global LISP database.

```
// no entries found in xTR2s global LISP database

xTR2#show ip lisp database 
% Could not find EID table in configuration.

// xTR2s LISP database for instance-ID 100

xTR2#show ip lisp instance-id 100 database 
LISP ETR IPv4 Mapping Database for EID-table vrf TEST (IID 100), LSBs: 0x1, 1 entries

1.1.1.0/24, locator-set xTR2
  Locator  Pri/Wgt  Source     State
  2.2.2.2    0/0    cfg-intf   site-self, reachable

// no entries found in xTR4s global LISP database

xTR4#show ip lisp database 
% Could not find EID table in configuration.

// xTR4s LISP database for instance-ID 100

xTR4#show ip lisp instance-id 100 database 
LISP ETR IPv4 Mapping Database for EID-table vrf TEST (IID 100), LSBs: 0x1, 1 entries

5.5.5.0/24, locator-set xTR4
  Locator  Pri/Wgt  Source     State
  4.4.4.4    0/0    cfg-intf   site-self, reachable
```

From the MS/MR perspective, we simply map the instance-IDs to the EIDs within the sites:

```
MS_MR#show run | sec router lisp
router lisp
 eid-table default instance-id 0
  exit
 !
 site SITE_A
  authentication-key cisco
  eid-prefix instance-id 100 1.1.1.0/24
  exit
 !
 site SITE_B
  authentication-key cisco
  eid-prefix instance-id 100 5.5.5.0/24
  exit
 !
 ipv4 map-server
 ipv4 map-resolver
 ipv4 map-request-source 3.3.3.3
 exit
```

## Packet walks

From this point on, the control-plane and data-plane packet walks are the same - the only difference is that the lookups happen against the specific instance-IDs that are defined. Let's consider both possible scenarios in our above topology. 

If the packet comes from R1, it comes in the VRF called TEST. This can be visualized like so:

![static1](/static/images/cisco/sda_7/multi_instance_lisp_2.jpg)

If the packet comes from R8, it comes in another VRF we created called TEST_2. This can be visualized like so:

![static1](/static/images/cisco/sda_7/multi_instance_lisp_3.jpg)

How does the LISP process know which instance-ID to consider? This is populated in the LISP packets themselves. For example, consider the following 'Encapsulated Map Request' packet:

![static1](/static/images/cisco/sda_7/multi_instance_lisp_4.jpg)

The LISP packet format has the provision to carry the instance-ID as an attribute within the source EID (as you can see from the above packet). This determines which instance-ID the lookups are done in. The same instance-ID is carried in actual data-plane packets as well. Consider the following ICMP packet as an example:

![static1](/static/images/cisco/sda_7/multi_instance_lisp_5.jpg)

This is the truest form of LISP implementation in SDA and this is exactly how SDA achieves macro segmentation. The terminology is 'Virtual Network' or VN in the SDA world however this is nothing but VRFs. 