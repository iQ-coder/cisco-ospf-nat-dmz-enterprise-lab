# cisco-ospf-nat-dmz-enterprise-lab

An enterprise network lab built on top of the security architecture from [network-security-dmz-architecture-lab](https://github.com/iQ-coder/network-security-dmz-architecture-lab), extending it with multi-area OSPF dynamic routing and NAT/PAT for internet access.

This lab demonstrates how a real enterprise network evolves — starting from a secure static-routed architecture and graduating to dynamic routing with proper address translation at the edge.

---

## What Was Added

| Feature | Description |
|---------|-------------|
| Multi-area OSPF | Area 0 backbone + Area 1 for internal LAN, R-INSIDE acts as ABR |
| NAT/PAT | All internal traffic translated to single public IP on R-EDGE |
| OSPF neighbor relationships | R-EDGE ↔ R-INSIDE (Area 0), R-INSIDE ↔ MLS (Area 1) |
| Static route cleanup | Replaced internal static routes with OSPF, kept default routes at edge |

---

## Topology

![Topology](https://raw.githubusercontent.com/iQ-coder/cisco-ospf-nat-dmz-enterprise-lab/main/Screenshot%202026-05-30%20032559.png)

---

## OSPF Design

### Area Design

| Area | Devices | Networks |
|------|---------|---------|
| Area 0 (Backbone) | R-EDGE, R-INSIDE | 10.0.0.0/30, 172.16.1.0/24 |
| Area 1 (LAN) | R-INSIDE, MLS | 192.168.0.0/16 (all VLANs) |

### Why Multi-Area?

A single Area 0 would work for this lab, but multi-area OSPF was chosen intentionally:

- Keeps the LSDB lighter in the backbone as the network grows
- Limits LSA flooding to within each area
- R-INSIDE acts as an ABR (Area Border Router), summarizing Area 1 routes into Area 0
- Reflects real enterprise design where the LAN is separated from the WAN-facing backbone

### Router IDs

| Device | Router ID |
|--------|-----------|
| R-EDGE | 1.1.1.1 |
| R-INSIDE | 2.2.2.2 |
| MLS | 3.3.3.3 |

### OSPF Configuration Per Device

**R-EDGE**
```
router ospf 1
 router-id 1.1.1.1
 network 10.0.0.0 0.0.0.3 area 0
 network 172.16.1.0 0.0.0.255 area 0
```

**R-INSIDE**
```
router ospf 1
 router-id 2.2.2.2
 network 10.0.0.0 0.0.0.3 area 0
 network 192.168.1.0 0.0.0.3 area 1
```

**MLS**
```
router ospf 1
 router-id 3.3.3.3
 network 192.168.0.0 0.0.255.255 area 1
```

---

## NAT/PAT Configuration

PAT (Port Address Translation) overload is used on R-EDGE — all internal devices share the single public IP `203.0.113.2`, differentiated by port numbers.

### Why PAT over Dynamic NAT?

Dynamic NAT requires a pool of public IPs — one per active session. PAT uses port numbers to multiplex thousands of sessions through a single public IP, which is how real enterprise edge routers and home routers work.

### Configuration

```
access-list 1 permit 192.168.0.0 0.0.255.255
access-list 1 permit 172.16.1.0 0.0.0.255

ip nat inside source list 1 interface serial0/3/0 overload

interface serial0/3/0
 ip nat outside

interface gig0/0
 ip nat inside

interface gig0/1
 ip nat inside
```

### Interface Roles

| Interface | NAT Role |
|-----------|----------|
| R-EDGE Se0/3/0 | Outside |
| R-EDGE Gig0/0 | Inside (DMZ) |
| R-EDGE Gig0/1 | Inside (toward R-INSIDE) |

---

## Routing Table (Final State)

| Device | Type | Network | Next Hop | Note |
|--------|------|---------|----------|------|
| MLS | Static | 0.0.0.0/0 | 192.168.1.1 | Default route to R-INSIDE |
| R-INSIDE | OSPF | 172.16.1.0/24 | 10.0.0.1 | DMZ learned via OSPF |
| R-INSIDE | OSPF | 192.168.x.0/24 | 192.168.1.2 | VLANs learned via OSPF |
| R-INSIDE | Static | 0.0.0.0/0 | 10.0.0.1 | Default route to R-EDGE |
| R-EDGE | OSPF | 192.168.x.0/24 | 10.0.0.2 | Internal subnets via OSPF |
| R-EDGE | Static | 0.0.0.0/0 | 203.0.113.1 | Default route to ISP |
| R-ISP | Static | 0.0.0.0/0 | 203.0.113.2 | Default route to R-EDGE |

---

## Test Results

| Test | From | To | Expected | Result |
|------|------|----|----------|--------|
| Internet access | PC-IT | PC-OUTSIDE (8.8.8.10) | ✓ Allowed via NAT | ✓ Pass |
| Internet access | PC-HR | PC-OUTSIDE (8.8.8.10) | ✓ Allowed via NAT | ✓ Pass |
| Internet access | PC-SALES | PC-OUTSIDE (8.8.8.10) | ✓ Allowed via NAT | ✓ Pass |
| OSPF convergence | R-INSIDE | All internal subnets | ✓ O routes in table | ✓ Pass |
| NAT translation | R-EDGE | show ip nat translations | ✓ 192.168.x.x → 203.0.113.2 | ✓ Pass |
| DMZ still blocked | WEB-Server | PC-IT | ✗ Blocked | ✓ Pass |
| HTTP from outside | PC-OUTSIDE | WEB-Server | ✓ Allowed | ✓ Pass |

---

## Key Concepts Demonstrated

- **Multi-area OSPF** — Area 0 backbone and Area 1 LAN with ABR at R-INSIDE
- **PAT overload** — many-to-one address translation using port multiplexing
- **Route redistribution awareness** — default routes kept static, internal routes dynamic
- **OSPF neighbor relationships** — hello packets, DR/BDR election on broadcast segments
- **ACL + NAT interaction** — inbound ACL on Se0/3/0 must permit established TCP and ICMP echo-replies for return traffic to flow

---

## Related Projects

This lab is part of a progressive Cisco networking series:

| Repo | Concepts |
|------|---------|
| [network-security-dmz-architecture-lab](https://github.com/iQ-coder/network-security-dmz-architecture-lab) | DMZ architecture, VLANs, ACLs, static routing |
| **cisco-ospf-nat-dmz-enterprise-lab** (this repo) | Multi-area OSPF, NAT/PAT, dynamic routing |

---

## Tools Used

- Cisco Packet Tracer 8.x
- Cisco 2911 Routers
- Cisco 3560-24PS Multilayer Switch
- Cisco 2960-24TT Layer 2 Switches
