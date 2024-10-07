---
date:
  created: 2022-01-03
  updated: 2022-01-03
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
# Cisco SDA Part IX - need for duplicate IPs on fabric borders
In this post, we look at why SD-Access borders have the anycast IP addresses configured as loopback addresses.

## Introduction and topology

Looking at the some of the configuration that is automatically pushed from DNAC, you should spot some very interesting things in there. This post aims to demystify these and help the reader understand why these were needed in the first place, hopefully giving you a better understanding of how the SDA fabric is built. 

Let's consider the following topology for this:

![static1](/static/images/cisco/sda_9/borders_1.jpg)

<!-- more -->

Since a SDA fabric implements an anycast gateway type SVI across all edges, you would see the same IP address to SVI mappings created on all your edges in the fabric. Interestingly enough, the same IP address is also mapped to a loopback interface on the borders of the fabric (one loopback on the border for every SVI created on the edge). Look at the following configuration snippets from an edge and a border:

```
// Border1
    
Border1#show run int lo1022
Building configuration...

Current configuration : 123 bytes
!
interface Loopback1022
 description Loopback Border
 vrf forwarding Guest_VN
 ip address 192.2.21.1 255.255.255.255
end

// Border2

Border2#show run int lo1022
Building configuration...

Current configuration : 123 bytes
!
interface Loopback1022
 description Loopback Border
 vrf forwarding Guest_VN
 ip address 192.2.21.1 255.255.255.255
end

// Edge1

Edge1#show run int vlan 1022
Building configuration...

Current configuration : 310 bytes
!
interface Vlan1022
 description Configured from Cisco DNA-Center
 mac-address 0000.0c9f.f45d
 vrf forwarding Guest_VN
 ip address 192.2.21.1 255.255.255.0
 ip helper-address 192.2.201.224
 no ip redirects
 ip route-cache same-interface
 no lisp mobility liveness test
 lisp mobility 192_2_21_0-Guest_VN
end

// Edge2

Edge2#show run int vlan 1022
Building configuration...

Current configuration : 310 bytes
!
interface Vlan1022
 description Configured from Cisco DNA-Center
 mac-address 0000.0c9f.f45d
 vrf forwarding Guest_VN
 ip address 192.2.21.1 255.255.255.0
 ip helper-address 192.2.201.224
 no ip redirects
 ip route-cache same-interface
 no lisp mobility liveness test
 lisp mobility 192_2_21_0-Guest_VN
end
```

## Need for loopback IPs on the border

Why do we do this? Remember when we talked about how DHCP works with SDA and the need for option82? Well, this ties directly into that. In that post, I stated:

> And here's the second major problem that DHCP needs to deal with in  SD-Access - this 'giaddr' address is essentially the interface VLANs IP address, which (as you might have guessed it) is the anycast gateway IP address across all Edge's. Essentially, the same VLAN and IP address  mapping will exist across all Edge's. How can the DHCP offer be directed to the Edge that actually sourced the DHCP discover?

The answer to this was given slightly later in the same post. For ease of reading, here is a direct quote from the post regarding how this works:

> So, when this offer hits the Border, it is leaked to the CPU and this RLOC + VNI information is extracted from option82.

However, I did not explain the little configuration "hack" that was done to make it leak to the CPU. When the DHCP offer comes back from the DHCP server, the destination IP address is the anycast gateway IP address. If this is not created on the border, how else would it leak the packet to the CPU and allow for DHCP snooping to process this and interpret the option82 data? 

So, there you have it - the reason DNAC pushes /32 loopbacks to the border with the same IP address as your anycast gateway IP addresses is to allow for the DHCP packets to leak to the CPU for processing of option82. The packet is then rebuilt and sent towards the appropriate edge switch. 

A second, equally interesting bit of configuration lies within the LISP and BGP sections of the border.

On the border, apart from configuring these /32 loopbacks with the same IPs as the anycast gateway IPs, we also advertise this into BGP:

```
// Border1

Border1#show run | sec router bgp
router bgp 65003
 bgp router-id interface Loopback0
 bgp log-neighbor-changes
 bgp graceful-restart
 
 *snip*

 address-family ipv4 vrf Guest_VN
  bgp aggregate-timer 0
  network 192.2.21.1 mask 255.255.255.255
  aggregate-address 192.2.21.0 255.255.255.0 summary-only
  redistribute lisp metric 10
  neighbor 192.2.100.6 remote-as 65002
  neighbor 192.2.100.6 update-source Vlan3004
  neighbor 192.2.100.6 activate
  neighbor 192.2.100.6 weight 65535
 exit-address-family

// Border2

Border2#show run | sec router bgp
router bgp 65003
 bgp router-id interface Loopback0
 bgp log-neighbor-changes
 bgp graceful-restart

 *snip*

 address-family ipv4 vrf Guest_VN
  bgp aggregate-timer 0
  network 192.2.21.1 mask 255.255.255.255
  aggregate-address 192.2.21.0 255.255.255.0 summary-only
  redistribute lisp metric 10
  neighbor 192.2.100.18 remote-as 65002
  neighbor 192.2.100.18 update-source Vlan3012
  neighbor 192.2.100.18 activate
  neighbor 192.2.100.18 weight 65535
 exit-address-family 
```

What is odd about this is that we're redistributing LISP into BGP already. This implies that my /32 LISP host entries in the server cache will be pushed into the BGP table and aggregated out to the peer anyway. So, what purpose does this very specific network statement serve? 

Well, consider a scenario where there are no hosts in the fabric at all. The first host connects to the network and tries to get an IP address via DHCP. At this point in time, will there be any entry in the LISP database? No - because the host doesn't even have a valid IP address yet. 

Naturally, the redistribution of LISP into BGP is useless at this point in time - there is nothing in the LISP server cache to redistribute. Because of this, the packets that come back from the DHCP server will get blackholed eventually - the upstream router (from the perspective of the border) will have no route to the anycast gateway IP address. 

This is why a manual network statement is needed to advertise this anycast gateway IP to the outside world. 

Interesting little tricks, innit? 