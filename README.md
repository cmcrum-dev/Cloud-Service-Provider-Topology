# Topology

![alt text](https://github.com/cmcrum-dev/Cloud-Service-Provider-Topology/blob/main/Topology.png "Topology")

## Introduction
 
This topology provides the fundamental layout/idea for a server hosting/service provider. In the topology I employed a few features that were not covered in the CCNA 200-301 exam, so this gave me an opportunity to grow and learn

This section of the lab models a cloud service provider (AS 100), something like a small scale AWS or like Linode but smaller. Having the ability to host multiple isolated tenants or multi-tenant layouts on shared infrastructure. the CSP is peered upstream to an ISP (AS 1). Customer sites reach their private cloud resources over tunnels that terminate directly to the CSP edge with their own tenant based routing table made possible by VRFlite.

## Topology Overview
 
### Cloud Service Provider (AS 100, 45.10.10.0/24)
 
- **Access layer (C1-A1, C1-A2):** L2 switches responsible for carrying tenant VLANs to hosts.
- **Distribution layer (C1-DS1, C1-DS2):** Multi-layer switches acting as tenant gateways, with VRF-aware SVIs with GLBP to provide High availability and Redundancy.
- **Edge routers (Cloud1-1, Cloud1-2):** Per-VRF transport to distribution layer, tunnel termination for customers, and BGP peering to the ISP (203.0.113.6/30, 203.0.113.134/30).
- At every point redundancy was ensured. If on link fails at any point network operations continue largely unaffected aside from ECMP and GLBP load balancing.

### Upstream ISP (AS 1)

- ISP1-1, ISP1-2, and ISP1-3 form a triangle core (203.0.113.0/26, 203.0.113.64/26, 203.0.113.128/26). The ISP in this topology was put in place mainly to illustrate there is almost always multiple hops from A - B when accessing anything over the internet a singular router not only would illustrate this, but also not have the port-density required for it.
- Customer sites VRF-Test (203.0.113.10/30) and VRF-Test2 (203.0.113.14/30) connect through ISP1-1. These were test users for testing single and multi-tenancy to ensure the tunnels,VRFs, and VLANs work

## Tenant Segmentation Overview

I aimed to maximize segmentation at all levels to ensure traffic doesn't leak or traverse into adjacent tenants

Implementing VRFs allowed for independent routing tables per tenant as well as multiples of the same IP addressing schemes (though this wasnt done for clarity), VLANs were implemented for L2 segmentation, and OSPF was enabled per VRF for redundancy and isolation of routing instances
 
| VRF Name | Subnet | VLAN | Gateway (GLBP VIP) | DS1 / DS2 SVI | OSPF Process + GLBP Group |
|---|---|---|---|---|---|
| testing | 10.100.254.0/24 | 999 | 10.100.254.1 | .2 / .3 | 999 |
| tenant_1 | 10.100.1.0/24 | 101 | 10.100.1.1 | .2 / .3 | 101 |
| tenant_2 | 10.100.20.0/24 | 102 | 10.100.20.1 | .2 / .3 | 102 |

Additionally each tenant's VLAN, OSPF process ID, and GLBP group share the same number, so any tenant can be identified from a single value during troubleshooting. 

## Main Design Decisions
 
**Gateway at the distribution layer, not the routers.** The edge routers that I am using in the lab are running IOSv and don't support LAG. When trying to place a tenant VLAN on two router interfaces simultaneously it fails, because IOS won't allow two interfaces in the same subnet which is expected. I then moved the tenant gateways and FHRP down to the distribution switches which aligns with proper design. The distribution to cloud edge links became routed /30  links, and there can be multiple VRFs per multiple links, carried on dot1q subinterfaces and trunk SVIs.
 
**ECMP instead of LAG.** With routed links and OSPF per VRF, each distribution switch learns customer routes from both edge routers, and traffic load-shares across equal-cost paths. This helps replaces the redundancy a LAG would have provided, with independent link failure detection.
 
**GLBP for first-hop load balancing.** Both distribution switches actively forward tenant traffic instead of one sitting in standby, as it would with HSRP or VRRP. This increases efficiency and makes sure there are no IDLE nodes
 
**Tunnels into tenant VRFs.** Private tenant addresses can't be routed across the internet. GRE tunnels carry them over the public underlay, and the tunnel interface's VRF determines which tenant the traffic belongs to.

## Beyond CCNA
 
- VRF-lite with per-VRF OSPF processes
- Route distinguishers ... Even though I'm not fully familiar what RDs are other than their similar to a VLAN tag for VRFs
- GLBP load balancing types, and AVF/AVG purposes
- GRE tunnel configs with VRFs 
- Per-VRF design with dot1q subinterfaces
- eBGP between provider and other autonomous systems

## Lessons Learned
 
- When applying vrf forwarding to an interface erases its IP, assigning the VRF first, then re-add the address is best practice.
- OSPF router IDs must be unique across all processes on a router, even across VRFs. This was jarring initially until I thought about it as a process on a computer. You cant have 2 processes with the same PID like Chrome(1) and Chrome(2) with the Same PID of 1492 or something along those lines. 
- Plain ping and show ip route only use the global table, so after alot of brow raising, and tracing configs just to find everything correct. I found that VRF troubleshooting needs to include the vrf keyword.

## Next Steps and Circling back
 
- Encrypt tunnels with IPsec. Maximize security per tenant
- Move away from static and OSPF tunnel routing to BGP tunnel routing. Closer to AWS based solutions (AWS VPN GW, AWS Customer GW)
- Add VRF-aware NAT for public-facing tenant services. Mimic AWS public IP assignment
- Long term: Clean up deployment, hammer in the knowledge, consider cloud-cloud setups
