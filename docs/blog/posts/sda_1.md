---
date:
  created: 2021-12-25
  updated: 2021-12-25
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
# Cisco SDA Part I - Introduction to LISP and its basic terminology
This is a new series that will cover Cisco's Software Defined Access architecture/solution over time. There are several moving pieces to this - in this post, we're going to start with a key component, which is LISP.

## Traditional architecture vs LISP architecture

This is a new series that will cover Cisco's Software Defined Access architecture/solution over time. There are several moving pieces to this - we're going to start with a key component, which is LISP. 

LISP stands for Locator/ID Separation protocol. Let's quickly revisit how endpoints are/were identified - with a simple IP address (IPv4/IPv6, what have you). The IP address was both the location and the identity of the endpoint. LISP (which serves as a routing architecture), aims to decouple the identity of an endpoint from its location. 

The IP address continues to be the identity of the endpoint however, its location is now advertised as a separate entity (or address space) as well. 

A simple visual comparison helps understand this better:

![static1](/static/images/cisco/sda_1/lisp1.jpg)

<!-- more -->

The location is also known as a routing locator or RLOC in the LISP world. An example of an RLOC would be an edge node that the endpoint connects to. The identity is known as endpoint ID or EID. These two terms (EIDs and RLOCs) are crucial to understanding LISP. 

LISP, as an architecture, must facilitate bi-directional conversation between EIDs (for example, 10.1.1.1 and 20.1.1.1 in my example above). It does so by exhibiting tunneling like behavior - think of it as a stateless tunnel, whose purpose is purely to get packets from one RLOC to another. To do this, it encapsulates the packet (similar to VXLAN), slaps on headers on top that can carry the packet through the routed core to the destination RLOC.

This, in turn, implies that there needs to be a mechanism that allows for RLOCs to advertise EIDs they are locally attached to, so that other RLOCs are aware of the location of remote EIDs. For this, LISP utilizes a mapping system that facilitates the propagation of EID to RLOC mappings. 

This is similar to how a routing protocol like OSPF advertises networks to its neighbors with one key difference - routing protocols (OSPF/EIGRP/BGP ...) are all based on a push model while LISP is based on a pull model (very important, considering the original goal and design of LISP was to solve the Internet's scalability problems). 

Think of this visually:


![static1](/static/images/cisco/sda_1/lisp2.jpg)


With your traditional routing models like OSPF, information was always pushed to the neighbors. This forced the peers to evaluate the prefixes, pushed them into RIB/FIB even if there were no active conversations against that prefix. 

LISP changes this dynamic completely by introducing a pull model. It works on an architecture where the RLOCs advertise their locally attached EIDs to a central system (SW1, in the topological example above) that stores it in some database. It DOES NOT advertise these EIDs to other RLOCs. If a RLOC wants to talk to a remote EID, it must query for information about this EID to the central system. As you can see, there is a quite the parallel here between LISP and DNS. 

## LISP terminology

Now that we have a basic understanding of this new routing architecture paradigm that LISP introduces, let's talk about some common terminology that you should know:

*EIDs* - endpoint IDs. These can be local (directly attached to a RLOC) or remote (not locally attached and learnt via the LISP mapping process). 
*RLOCs* - routing locators. This is how the location of an endpoint is identified. Typically the first routed hop (the default gateway for your endpoints). 
*Map Server and Map Resolver (MS/MR)* - the LISP mapping system is distributed across the Map Server and the Map Resolver. RLOCs send information about locally attached EIDs to the Map Server to store in its database while the Map Resolver is used to forward queries for EIDs to the Map Server so that it can reply back. Typically, one device acts as both the Map Server and the Map Resolver, especially in the SDA world. 
*eTR* - egress tunnel router. This device registers its locally attached EIDs with the Map Server. It also decapsulates the encapsulated packets it receives to send natively towards the destination. 
*iTR* - ingress tunnel router. This device sends requests to the Map Resolver about EIDs it wants to talk to. It also encapsulates a native packet with the appropriate LISP headers and forwards it on, post encapsulation.
*xTR* - typically, one device will perform both eTR and iTR functions when you consider bi-directional conversation. Such a device is termed as 'xTR'. 

The following visualization should help you understand the above terminology better:

![static1](/static/images/cisco/sda_1/lisp3.jpg)

In the next post, we'll get to the meat of LISP - its basic configuration and operation.