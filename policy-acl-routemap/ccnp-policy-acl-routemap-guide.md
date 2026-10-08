# CCNP Enterprise Policy Lab — ACLs & Route-Maps

A large (~22-node) lab built to exercise **ACLs and route-maps** end to end. The topology
deliberately runs two IGPs (OSPF on the campus side, EIGRP on the data-center/branch side)
that meet at redistribution boundaries, a BGP edge with two ISPs and a partner, and a
policy-based-routing path with dual upstreams — so every kind of policy tool has a real
surface to act on.

> Interface names match the imported YAML. `Gi0/0` is the management slot, unused for data.
> IOSvL2 switches (DIST-A/B, ACC, DC-SW, BR-SW) number interfaces `Gi0/0–0/3` then
> `Gi1/0–1/3`; routed /31s on a switch need `no switchport` first. Verify the mapping in
> CML before configuring.

Work the sections in order. Sections 1 (IGPs) and 4 (BGP) build the routing substrate;
Sections 2, 3, and 5 are the policy focus (PBR, redistribution route-maps, security ACLs).
The final section is an unguided challenge.

---

## What to do where — quick map

At-a-glance summary of the task and the routers it touches per section. Full config is in
each section below.

**Section 1 — Routing substrate**
- OSPF area 0, per-interface, p2p /31s: **EDGE1/2, CORE1/2, DIST-A/B** (`no switchport` on DIST uplinks).
- EIGRP AS 100, named mode: **AGG1/2, DC1, BR1**.
- AGG1 sits in *both* IGPs (it's the redistribution boundary for Section 3).

**Section 2 — PBR**
- All work is on **BR1**: extended ACL matching branch users, route-map setting AGG2 next-hop, `ip policy` inbound on the LAN interface, plus IP SLA + track for fail-back.

**Section 3 — Redistribution route-maps**
- **AGG1**: EIGRP <-> OSPF redistribution both directions, with tags for loop prevention and a prefix-list limiting the EIGRP -> OSPF direction to DC + branch prefixes.
- **CORE1**: OSPF -> BGP redistribution, setting community 65000:100 on internal prefixes.

**Section 4 — BGP path control**
- **EDGE1**: eBGP to ISP1 and PARTNER, iBGP to EDGE2, local-pref in from ISP1, partner in/out community + AS-path policy.
- **EDGE2**: eBGP to ISP2, iBGP to EDGE1, AS-path prepend out toward ISP2.
- **ISP1 / ISP2 / PARTNER**: peer-side eBGP + originate their test prefixes.

**Section 5 — Security ACLs**
- **DIST-A**: extended named ACL on VLAN10 (user -> server rules).
- **SRV-PUB**: time-based DMZ ACL (maintenance-window SSH).
- **DC1**: reflexive ACL pair for internal-services return traffic.
- **EDGE1**: infrastructure ACL, VTY access-class, and CoPP.

---

## Addressing plan

Loopbacks are router-IDs and policy anchors.

| Device | Loopback0 | Domain / role |
|---|---|---|
| EDGE1 | 10.0.0.1/32 | BGP edge, PBR enforce, iACL/CoPP |
| EDGE2 | 10.0.0.2/32 | BGP edge (2nd path) |
| CORE1 | 10.0.0.3/32 | OSPF core, OSPF -> BGP |
| CORE2 | 10.0.0.4/32 | OSPF core, OSPF -> EIGRP |
| DIST-A | 10.0.0.5/32 | OSPF dist, user-VLAN ACLs |
| DIST-B | 10.0.0.6/32 | OSPF dist, user-VLAN ACLs |
| AGG1 | 10.0.0.8/32 | EIGRP agg, EIGRP <-> OSPF route-map filter |
| AGG2 | 10.0.0.9/32 | EIGRP agg, PBR next-hop path |
| DC1 | 10.0.0.10/32 | Data-center router, server ACLs |
| BR1 | 10.0.0.12/32 | Branch (EIGRP), PBR source site |
| ISP1 | 172.16.1.1/32 | eBGP AS 65101 |
| ISP2 | 172.16.2.1/32 | eBGP AS 65102 |
| PARTNER | 172.16.20.1/32 | eBGP AS 65200 |

Selected transit /31s (mask `255.255.255.254`):

| Link | Subnet | Left / Right |
|---|---|---|
| EDGE1–CORE1 | 10.1.0.0/31 | .0 / .1 |
| EDGE2–CORE2 | 10.1.0.2/31 | .2 / .3 |
| EDGE1–CORE2 | 10.1.0.4/31 | .4 / .5 |
| EDGE2–CORE1 | 10.1.0.6/31 | .6 / .7 |
| CORE1–CORE2 | 10.1.0.8/31 | .8 / .9 |
| CORE1–DIST-A | 10.1.0.10/31 | .10 / .11 |
| CORE2–DIST-B | 10.1.0.12/31 | .12 / .13 |
| DIST-A–DIST-B | 10.1.0.14/31 | .14 / .15 |
| CORE2–AGG1 | 10.1.0.16/31 | .16 / .17 |
| CORE1–AGG1 | 10.1.0.18/31 | .18 / .19 |
| AGG1–AGG2 | 10.1.0.20/31 | .20 / .21 |
| AGG1–DC1 | 10.1.0.22/31 | .22 / .23 |
| AGG2–DC1 | 10.1.0.24/31 | .24 / .25 |
| BR1–AGG1 | 10.1.0.26/31 | .26 / .27 |
| BR1–AGG2 | 10.1.0.28/31 | .28 / .29 |

Edge/BGP peering fabric (INET-SW, 100.64.0.0/24): ISP1 .101, ISP2 .102, PARTNER .200,
EDGE1 .1, EDGE2 .2.

Segments:

| Segment | Subnet | Notes |
|---|---|---|
| User VLAN10 | 192.168.10.0/24 | ACC-A1, ACL target (user -> server rules) |
| User VLAN20 | 192.168.20.0/24 | ACC-A2 |
| DMZ / public | 172.31.0.0/24 | SRV-PUB (.10), time-based + iACL targets |
| Internal svcs | 10.20.0.0/24 | SRV-INT (.10), reflexive/established targets |
| DC servers | 10.30.0.0/24 | behind DC-SW |
| Branch LAN | 10.40.0.0/24 | behind BR1, PBR source |

---

## Section 1 — Routing substrate (OSPF + EIGRP)

Goal: stand up the two IGP domains so later policy has routes to act on. This section is
deliberately light — the interesting policy work is Sections 2, 3, 5.

### 1.1 OSPF domain (campus/left)

EDGE1/EDGE2, CORE1/CORE2, DIST-A/DIST-B in OSPF area 0, enabled **per interface**. On the
IOSvL2 distribution switches the routed uplinks need `no switchport`.

```
! CORE1 (representative)
interface Loopback0
 ip address 10.0.0.3 255.255.255.255
 ip ospf 1 area 0
interface GigabitEthernet0/1
 ip address 10.1.0.1 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
router ospf 1
 router-id 10.0.0.3
```

### 1.2 EIGRP domain (DC/branch/right)

AGG1/AGG2, DC1, BR1 in EIGRP AS 100 (named mode). AGG1 also has OSPF interfaces toward the
core — it's the redistribution boundary (Section 3).

```
! AGG2 (representative)
router eigrp DC
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 10.0.0.9
  network 10.0.0.9 0.0.0.0
  network 10.1.0.20 0.0.0.1
  network 10.1.0.24 0.0.0.1
  network 10.1.0.28 0.0.0.1
```

### Verify Section 1
```
show ip ospf neighbor
show ip eigrp neighbors
show ip route
```
OSPF full on the campus side, EIGRP adjacencies on the DC/branch side. No redistribution
yet, so the two domains can't see each other — that's Section 3.

### Full config for this section

_Only this section's commands, per device it touches. Representative devices shown; mirror on the symmetric peer (e.g. CORE2, DIST-B, EDGE2/ISP2) using the addressing plan._

**CORE1**
```
hostname CORE1
interface Loopback0
 ip address 10.0.0.3 255.255.255.255
 ip ospf 1 area 0
interface GigabitEthernet0/1
 description to EDGE1
 ip address 10.1.0.1 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet0/2
 description to EDGE2
 ip address 10.1.0.7 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet0/3
 description to CORE2
 ip address 10.1.0.8 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet0/4
 description to DIST-A
 ip address 10.1.0.10 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
router ospf 1
 router-id 10.0.0.3
```
**DIST-A**
```
hostname DIST-A
ip routing
interface Loopback0
 ip address 10.0.0.5 255.255.255.255
 ip ospf 1 area 0
interface GigabitEthernet0/1
 description to CORE1
 no switchport
 ip address 10.1.0.11 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet0/2
 description to DIST-B (peer)
 no switchport
 ip address 10.1.0.14 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
router ospf 1
 router-id 10.0.0.5
```
**AGG1**
```
hostname AGG1
interface Loopback0
 ip address 10.0.0.8 255.255.255.255
interface GigabitEthernet0/1
 description to CORE2 (OSPF side)
 ip address 10.1.0.17 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet0/2
 description to CORE1 (OSPF side)
 ip address 10.1.0.19 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet0/3
 description to AGG2
 ip address 10.1.0.20 255.255.255.254
 no shutdown
interface GigabitEthernet0/4
 description to DC1
 ip address 10.1.0.22 255.255.255.254
 no shutdown
interface GigabitEthernet0/5
 description to BR1
 ip address 10.1.0.26 255.255.255.254
 no shutdown
router ospf 1
 router-id 10.0.0.8
router eigrp DC
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 10.0.0.8
  network 10.1.0.20 0.0.0.1
  network 10.1.0.22 0.0.0.1
  network 10.1.0.26 0.0.0.1
```
**AGG2**
```
hostname AGG2
interface Loopback0
 ip address 10.0.0.9 255.255.255.255
interface GigabitEthernet0/1
 description to AGG1
 ip address 10.1.0.21 255.255.255.254
 no shutdown
interface GigabitEthernet0/2
 description to DC1
 ip address 10.1.0.24 255.255.255.254
 no shutdown
interface GigabitEthernet0/3
 description to BR1 (PBR path)
 ip address 10.1.0.29 255.255.255.254
 no shutdown
router eigrp DC
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 10.0.0.9
  network 10.0.0.9 0.0.0.0
  network 10.1.0.20 0.0.0.1
  network 10.1.0.24 0.0.0.1
  network 10.1.0.28 0.0.0.1
```
**BR1**
```
hostname BR1
interface Loopback0
 ip address 10.0.0.12 255.255.255.255
interface GigabitEthernet0/1
 description to AGG1 (primary)
 ip address 10.1.0.27 255.255.255.254
 no shutdown
interface GigabitEthernet0/2
 description to AGG2 (PBR path)
 ip address 10.1.0.28 255.255.255.254
 no shutdown
interface GigabitEthernet0/3
 description to BR-SW (branch LAN)
 ip address 10.40.0.1 255.255.255.0
 no shutdown
router eigrp DC
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 10.0.0.12
  network 10.1.0.26 0.0.0.1
  network 10.1.0.28 0.0.0.1
  network 10.40.0.0 0.0.0.255
```


---

## Section 2 — Policy-Based Routing (PBR)

Goal: override the routing table for selected traffic. BR1 has two upstreams — AGG1
(primary, the routing table's choice) and AGG2 (policy path). Force branch **user** traffic
(and only that) out AGG2 while leaving everything else on the normal path, then make the
PBR resilient with IP SLA.

### 2.1 Match interesting traffic with an ACL

The route-map matches on an ACL — the first interlock between the two topics.

```
ip access-list extended PBR-USERS
 permit ip 10.40.0.0 0.0.0.255 any
```

### 2.2 Route-map that sets the next hop

```
route-map PBR-BRANCH permit 10
 match ip address PBR-USERS
 set ip next-hop 10.1.0.29
route-map PBR-BRANCH permit 20
 ! everything else: no set, falls through to normal routing
```

### 2.3 Apply inbound on the source interface

PBR acts on traffic **arriving** on the interface facing the hosts.

```
interface GigabitEthernet0/3
 description to BR-SW (branch LAN)
 ip policy route-map PBR-BRANCH
```

### 2.4 Make PBR resilient (IP SLA + verify-availability)

If the AGG2 path dies, PBR would black-hole. Track it and only set the next hop when it's
reachable; otherwise traffic falls back to the routing table.

```
ip sla 10
 icmp-echo 10.1.0.29 source-interface GigabitEthernet0/2
 frequency 5
ip sla schedule 10 life forever start-time now
track 10 ip sla 10 reachability
route-map PBR-BRANCH permit 10
 match ip address PBR-USERS
 set ip next-hop verify-availability 10.1.0.29 1 track 10
```

### Verify Section 2
```
show route-map PBR-BRANCH
show ip policy
traceroute 10.30.0.10 source 10.40.0.10   ! from a branch user - should exit via AGG2
show track 10
```
Branch user traffic takes the AGG2 path; other branch traffic uses AGG1. Shut the AGG2 link
and confirm PBR fails back to normal routing.

### Full config for this section

_Only this section's commands, per device it touches. Representative devices shown; mirror on the symmetric peer (e.g. CORE2, DIST-B, EDGE2/ISP2) using the addressing plan._

**BR1**
```
hostname BR1
!
ip access-list extended PBR-USERS
 permit ip 10.40.0.0 0.0.0.255 any
!
ip sla 10
 icmp-echo 10.1.0.29 source-interface GigabitEthernet0/2
 frequency 5
ip sla schedule 10 life forever start-time now
!
track 10 ip sla 10 reachability
!
route-map PBR-BRANCH permit 10
 match ip address PBR-USERS
 set ip next-hop verify-availability 10.1.0.29 1 track 10
route-map PBR-BRANCH permit 20
!
interface GigabitEthernet0/3
 ip policy route-map PBR-BRANCH
```


---

## Section 3 — Redistribution filtering with route-maps

Goal: connect the OSPF and EIGRP domains at AGG1 (and BGP at the core), controlling exactly
what crosses with route-maps — tagging to prevent loops, prefix selection, and metric
setting.

### 3.1 OSPF -> EIGRP redistribution at AGG1 (tagged, filtered)

Redistribution runs both ways at AGG1, but each direction is a separate policy: EIGRP -> OSPF
and OSPF -> EIGRP. Tag routes as they cross in each direction; deny the return of anything
carrying the far domain's tag (loop prevention). Also filter: only allow the DC and branch
prefixes in the EIGRP -> OSPF direction.

```
ip prefix-list DC-BRANCH seq 5 permit 10.30.0.0/24
ip prefix-list DC-BRANCH seq 10 permit 10.40.0.0/24
!
route-map EIGRP-INTO-OSPF permit 10
 match ip address prefix-list DC-BRANCH
 set tag 100
route-map EIGRP-INTO-OSPF deny 20
 ! implicitly drop everything else crossing into OSPF
!
route-map OSPF-INTO-EIGRP deny 10
 match tag 100
route-map OSPF-INTO-EIGRP permit 20
 set tag 200
!
router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP-INTO-OSPF
router eigrp DC
 address-family ipv4 unicast autonomous-system 100
  topology base
   redistribute ospf 1 metric 1000000 100 255 1 1500 route-map OSPF-INTO-EIGRP
  exit-af-topology
```

### 3.2 OSPF -> BGP at the core

Bring internal prefixes into BGP selectively and set a community as they go (used in
Section 4's policy).

```
ip prefix-list INTERNAL seq 5 permit 192.168.10.0/24
ip prefix-list INTERNAL seq 10 permit 192.168.20.0/24
route-map OSPF-INTO-BGP permit 10
 match ip address prefix-list INTERNAL
 set community 65000:100
router bgp 65000
 address-family ipv4
  redistribute ospf 1 route-map OSPF-INTO-BGP
```

### Verify Section 3
```
show ip route ospf        ! DC/branch prefixes as E2 with tag 100
show ip route eigrp       ! campus prefixes as D EX, tag 200
show ip bgp               ! internal prefixes with community 65000:100
show route-map
```
Only the allowed prefixes cross; no route carries back into the domain it came from.

### Full config for this section

_Only this section's commands, per device it touches. Representative devices shown; mirror on the symmetric peer (e.g. CORE2, DIST-B, EDGE2/ISP2) using the addressing plan._

**AGG1**
```
hostname AGG1
!
ip prefix-list DC-BRANCH seq 5 permit 10.30.0.0/24
ip prefix-list DC-BRANCH seq 10 permit 10.40.0.0/24
!
route-map EIGRP-INTO-OSPF permit 10
 match ip address prefix-list DC-BRANCH
 set tag 100
route-map EIGRP-INTO-OSPF deny 20
!
route-map OSPF-INTO-EIGRP deny 10
 match tag 100
route-map OSPF-INTO-EIGRP permit 20
 set tag 200
!
router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP-INTO-OSPF
!
router eigrp DC
 address-family ipv4 unicast autonomous-system 100
  topology base
   redistribute ospf 1 metric 1000000 100 255 1 1500 route-map OSPF-INTO-EIGRP
  exit-af-topology
```
**CORE1**
```
hostname CORE1
!
ip prefix-list INTERNAL seq 5 permit 192.168.10.0/24
ip prefix-list INTERNAL seq 10 permit 192.168.20.0/24
!
route-map OSPF-INTO-BGP permit 10
 match ip address prefix-list INTERNAL
 set community 65000:100
!
router bgp 65000
 bgp router-id 10.0.0.3
 address-family ipv4
  redistribute ospf 1 route-map OSPF-INTO-BGP
```


---

## Section 4 — BGP path control with route-maps

Goal: two ISPs plus a partner, with route-maps steering inbound and outbound path selection
and using communities. ISP1 is primary, ISP2 backup; the partner gets a filtered, tagged
view.

### 4.1 Sessions

```
! EDGE1
router bgp 65000
 bgp router-id 10.0.0.1
 neighbor 100.64.0.101 remote-as 65101      ! ISP1
 neighbor 100.64.0.200 remote-as 65200      ! PARTNER
 neighbor 10.0.0.2 remote-as 65000           ! iBGP to EDGE2
 neighbor 10.0.0.2 update-source Loopback0
```

### 4.2 Steer our outbound traffic via ISP1 (local-preference)

Local-pref is set on routes coming *in* from ISP1 (route-map applied `in`), but what it
controls is which exit *our* AS prefers — i.e. our **outbound** traffic. Higher local-pref
wins, so 200 makes ISP1 the preferred exit. Applied on **EDGE1**.

```
route-map FROM-ISP1 permit 10
 set local-preference 200
router bgp 65000
 address-family ipv4
  neighbor 100.64.0.101 route-map FROM-ISP1 in
```

### 4.3 Steer inbound traffic to arrive via ISP1 (AS-path prepend toward ISP2)

To influence how the outside world reaches us (**inbound** traffic), we make the ISP2 path
look worse by prepending our AS on advertisements sent *out* to ISP2. This config lives on
**EDGE2** (the router peering with ISP2), not EDGE1.

```
route-map TO-ISP2 permit 10
 set as-path prepend 65000 65000 65000
router bgp 65000
 address-family ipv4
  neighbor 100.64.0.102 route-map TO-ISP2 out   ! on EDGE2
```

### 4.4 Partner policy — filter with an ACL/prefix-list and match community

Only advertise internal (community 65000:100) prefixes to the partner, and only accept the
partner's own routes back.

```
ip as-path access-list 10 permit ^65200$
route-map TO-PARTNER permit 10
 match community 1
 set community no-export additive
ip community-list 1 permit 65000:100
route-map FROM-PARTNER permit 10
 match as-path 10
router bgp 65000
 address-family ipv4
  neighbor 100.64.0.200 route-map TO-PARTNER out
  neighbor 100.64.0.200 route-map FROM-PARTNER in
```

### Verify Section 4
```
show ip bgp summary
show ip bgp
show ip bgp neighbors 100.64.0.101 advertised-routes
show ip bgp community 65000:100
```
ISP1 preferred both directions; partner sees only tagged internal prefixes marked
no-export; only partner-origin routes accepted inbound.

### Full config for this section

_Only this section's commands, per device it touches. Representative devices shown; mirror on the symmetric peer (e.g. CORE2, DIST-B, EDGE2/ISP2) using the addressing plan._

**EDGE1**
```
hostname EDGE1
!
interface GigabitEthernet0/1
 description to INET-SW (peering fabric)
 ip address 100.64.0.1 255.255.255.0
 no shutdown
!
ip community-list 1 permit 65000:100
ip as-path access-list 10 permit ^65200$
!
route-map FROM-ISP1 permit 10
 set local-preference 200
route-map TO-PARTNER permit 10
 match community 1
 set community no-export additive
route-map FROM-PARTNER permit 10
 match as-path 10
!
router bgp 65000
 bgp router-id 10.0.0.1
 neighbor 100.64.0.101 remote-as 65101
 neighbor 100.64.0.200 remote-as 65200
 neighbor 10.0.0.2 remote-as 65000
 neighbor 10.0.0.2 update-source Loopback0
 address-family ipv4
  neighbor 100.64.0.101 activate
  neighbor 100.64.0.101 route-map FROM-ISP1 in
  neighbor 100.64.0.200 activate
  neighbor 100.64.0.200 route-map TO-PARTNER out
  neighbor 100.64.0.200 route-map FROM-PARTNER in
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 next-hop-self
  neighbor 10.0.0.2 send-community
```
**EDGE2**
```
hostname EDGE2
!
interface GigabitEthernet0/1
 description to INET-SW (peering fabric)
 ip address 100.64.0.2 255.255.255.0
 no shutdown
!
route-map TO-ISP2 permit 10
 set as-path prepend 65000 65000 65000
!
router bgp 65000
 bgp router-id 10.0.0.2
 neighbor 100.64.0.102 remote-as 65102
 neighbor 10.0.0.1 remote-as 65000
 neighbor 10.0.0.1 update-source Loopback0
 address-family ipv4
  neighbor 100.64.0.102 activate
  neighbor 100.64.0.102 route-map TO-ISP2 out
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.1 next-hop-self
  neighbor 10.0.0.1 send-community
```
**ISP1**
```
hostname ISP1
interface GigabitEthernet0/1
 ip address 100.64.0.101 255.255.255.0
 no shutdown
interface Loopback1
 ip address 203.0.113.1 255.255.255.0
router bgp 65101
 bgp router-id 172.16.1.1
 neighbor 100.64.0.1 remote-as 65000
 address-family ipv4
  neighbor 100.64.0.1 activate
  network 203.0.113.0 mask 255.255.255.0
```
**PARTNER**
```
hostname PARTNER
interface GigabitEthernet0/1
 ip address 100.64.0.200 255.255.255.0
 no shutdown
interface Loopback1
 ip address 198.51.100.1 255.255.255.0
router bgp 65200
 bgp router-id 172.16.20.1
 neighbor 100.64.0.1 remote-as 65000
 address-family ipv4
  neighbor 100.64.0.1 activate
  network 198.51.100.0 mask 255.255.255.0
```


---

## Section 5 — Security ACLs

Goal: the full ACL toolkit against real segments — standard, extended, named, time-based,
reflexive/established, an infrastructure ACL, and CoPP. Several reference or are referenced
by the policy built earlier.

### 5.1 Extended named ACL: user -> server rules (on DIST-A, VLAN10 inbound)

```
ip access-list extended VLAN10-IN
 permit tcp 192.168.10.0 0.0.0.255 10.30.0.0 0.0.0.255 eq 443
 permit tcp 192.168.10.0 0.0.0.255 10.30.0.0 0.0.0.255 eq 22
 permit udp 192.168.10.0 0.0.0.255 host 10.20.0.10 eq 53
 deny   ip  192.168.10.0 0.0.0.255 10.20.0.0 0.0.0.255
 permit ip  192.168.10.0 0.0.0.255 any
interface Vlan10
 ip access-group VLAN10-IN in
```

### 5.2 Time-based ACL: DMZ maintenance window (SRV-PUB)

```
time-range MAINT
 periodic daily 22:00 to 23:59
ip access-list extended DMZ-IN
 permit tcp any host 172.31.0.10 eq 443
 permit tcp any host 172.31.0.10 eq 22 time-range MAINT
 deny   ip any any log
interface GigabitEthernet0/4
 ip access-group DMZ-IN in
```

### 5.3 Established / reflexive: internal services return traffic (DC1)

```
ip access-list extended INT-OUT
 permit tcp 10.30.0.0 0.0.0.255 any reflect MIRROR
ip access-list extended INT-IN
 evaluate MIRROR
 deny ip any any
interface GigabitEthernet0/4
 ip access-group INT-OUT out
 ip access-group INT-IN in
```

### 5.4 Infrastructure ACL + VTY protection (EDGE1)

```
ip access-list extended iACL-IN
 deny ip any 10.0.0.0 0.0.0.255              ! protect loopback/infra space from outside
 permit ip any any
interface GigabitEthernet0/1
 ip access-group iACL-IN in
!
ip access-list standard MGMT
 permit 10.20.0.0 0.0.0.255
line vty 0 4
 access-class MGMT in
 transport input ssh
```

### 5.5 CoPP (EDGE1)

```
ip access-list extended CoPP-ICMP
 permit icmp any any
class-map match-all CM-ICMP
 match access-group name CoPP-ICMP
policy-map CoPP
 class CM-ICMP
  police 8000 conform-action transmit exceed-action drop
control-plane
 service-policy input CoPP
```

### Verify Section 5
```
show ip access-lists
show access-lists MIRROR          ! reflexive entries appear as sessions open
show time-range
show policy-map control-plane
```
VLAN10 users reach servers only on allowed ports; DMZ SSH only in the window; internal
return traffic permitted only for flows it initiated; infra space unreachable from outside;
control-plane ICMP policed.

### Full config for this section

_Only this section's commands, per device it touches. Representative devices shown; mirror on the symmetric peer (e.g. CORE2, DIST-B, EDGE2/ISP2) using the addressing plan._

**DIST-A**
```
hostname DIST-A
!
ip access-list extended VLAN10-IN
 permit tcp 192.168.10.0 0.0.0.255 10.30.0.0 0.0.0.255 eq 443
 permit tcp 192.168.10.0 0.0.0.255 10.30.0.0 0.0.0.255 eq 22
 permit udp 192.168.10.0 0.0.0.255 host 10.20.0.10 eq 53
 deny   ip  192.168.10.0 0.0.0.255 10.20.0.0 0.0.0.255
 permit ip  192.168.10.0 0.0.0.255 any
!
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
 ip access-group VLAN10-IN in
```
**SRV-PUB**
```
hostname SRV-PUB
!
time-range MAINT
 periodic daily 22:00 to 23:59
!
ip access-list extended DMZ-IN
 permit tcp any host 172.31.0.10 eq 443
 permit tcp any host 172.31.0.10 eq 22 time-range MAINT
 deny   ip any any log
!
interface GigabitEthernet0/1
 description DMZ segment
 ip access-group DMZ-IN in
```
**DC1**
```
hostname DC1
!
ip access-list extended INT-OUT
 permit tcp 10.30.0.0 0.0.0.255 any reflect MIRROR
ip access-list extended INT-IN
 evaluate MIRROR
 deny ip any any
!
interface GigabitEthernet0/4
 description to SRV-INT
 ip access-group INT-OUT out
 ip access-group INT-IN in
```
**EDGE1**
```
hostname EDGE1
!
ip access-list extended iACL-IN
 deny ip any 10.0.0.0 0.0.0.255
 permit ip any any
!
ip access-list standard MGMT
 permit 10.20.0.0 0.0.0.255
!
ip access-list extended CoPP-ICMP
 permit icmp any any
class-map match-all CM-ICMP
 match access-group name CoPP-ICMP
policy-map CoPP
 class CM-ICMP
  police 8000 conform-action transmit exceed-action drop
!
interface GigabitEthernet0/1
 ip access-group iACL-IN in
!
line vty 0 4
 access-class MGMT in
 transport input ssh
!
control-plane
 service-policy input CoPP
```


---

## Final integration check

- Branch user (10.40.0.10) -> DC server: exits via the AGG2 PBR path; other branch traffic via AGG1.
- Campus and DC/branch reach each other only for the prefixes Section 3 allowed.
- ISP1 is the active path both directions; failing it moves traffic to ISP2.
- Partner sees only tagged internal prefixes (no-export); its inbound is AS-path filtered.
- VLAN10 host can reach servers only on 443/22 and DNS; blocked from internal svcs subnet.
- DMZ SSH works only during the maintenance window; infra ACL blocks the loopback range from the edge.

---

# CHALLENGE — steps only, no guide

Rebuild from a blank topology. No command hints. Verify each step before moving on.

1. Stand up OSPF area 0 across EDGE1/2, CORE1/2, DIST-A/B (per-interface, /31, p2p); `no switchport` on the DIST routed uplinks.
2. Stand up EIGRP AS 100 (named mode) across AGG1/2, DC1, BR1.
3. PBR: match branch user subnet with an extended ACL and force it out the AGG2 next-hop via a route-map applied inbound on BR1's LAN interface.
4. Add IP SLA + `verify-availability` so the PBR next-hop is used only when reachable; prove fail-back.
5. Redistribute EIGRP <-> OSPF at AGG1 (both directions) with tags preventing loopback and a prefix-list allowing only DC + branch prefixes in the EIGRP -> OSPF direction.
6. Redistribute OSPF -> BGP at the core, setting community 65000:100 on internal prefixes via route-map.
7. Bring up eBGP to ISP1, ISP2, and the PARTNER, plus iBGP EDGE1 <-> EDGE2.
8. Prefer ISP1 inbound (local-preference) and outbound (AS-path prepend toward ISP2).
9. Advertise only community-65000:100 prefixes to the partner, tagged no-export; accept only partner-origin (AS-path filtered) routes inbound.
10. Extended named ACL on VLAN10 allowing only 443/22 to DC servers and DNS to internal svcs, denying the internal-svcs subnet otherwise.
11. Time-based ACL permitting DMZ SSH only in a nightly window.
12. Reflexive ACL pair so internal-services return traffic is allowed only for flows it initiated.
13. Infrastructure ACL on the edge denying outside access to the 10.0.0.0/24 loopback space; VTY access-class limited to the mgmt subnet, SSH only.
14. CoPP policing ICMP to the control plane on EDGE1.
15. Prove every check in the integration list passes.
16. Bonus: convert one numbered ACL to named and add `log-input`; observe hits with `show access-lists`.
17. Bonus: add a second PBR clause that sets a different next-hop by DSCP marking, matched with an extended ACL.
