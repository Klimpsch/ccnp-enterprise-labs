# CCNP Enterprise OSPF-Core Lab — Configuration Guide

A start-to-finish build guide for the `CCNP-Enterprise-OSPF-Core` topology. Work the
sections in order — each one assumes the previous is up. The final section is an
unguided challenge: steps only, no walkthrough.

> **OSPF is enabled per interface** in this guide (`ip ospf <process> area <area>`),
> not with `network` statements under the routing process. This is the modern, explicit
> way: the area is declared on the interface it applies to, there's no wildcard-mask
> arithmetic, and adding or renumbering a link never means editing the router stanza.

> Interface names below match the imported YAML exactly. `Gi0/0` is the management
> slot and is left unused for data. Verify the mapping in CML once before you start —
> if you added or reordered links, your slots will differ.

---

## What to do where — quick map

At-a-glance summary of the task and the devices it touches per section. Full config is in
each section below.

**Section 1 — OSPF core**
- Per-interface OSPF area 0, p2p /31s, SHA-256 auth, matching reference-bandwidth: **CORE1/2, DIST1/2** (`no switchport` + `ip routing` on DIST uplinks).
- NSSA Area 20 ABR: **WAN1** (Area 0 to core, Area 20 to branch); NSSA internal: **BRANCH-EDGE**.
- Stub Area 10 toward services: **DIST1** services SVI (or a dedicated SRV-CORE router).

**Section 2 — Layer 2 access**
- VLANs 10/20/99, trunks, port hardening (portfast, BPDU guard, port-security, DHCP snooping, DAI): **ACC1/2/3**.
- LACP bundle from **ACC1** to **DIST1** (two links) + independent backup trunk to **DIST2**.
- RSTP root placement: **DIST1** (root VLAN10), **DIST2** (root VLAN20).

**Section 3 — FHRP + IP SLA**
- HSRPv2 on SVIs, IP SLA probe + tracked object decrementing priority on upstream failure: **DIST1** (active VLAN10) and **DIST2** (active VLAN20).

**Section 4 — BGP**
- eBGP to ISP + BGP origination + local-pref in: **CORE1** (to ISP1), **CORE2** (to ISP2, AS-path prepend out).
- iBGP over loopbacks with next-hop-self: **CORE1 <-> CORE2**.
- Peer-side eBGP + originate test prefixes: **ISP1, ISP2**.

**Section 5 — Redistribution**
- EIGRP AS 100 in the branch: **BRANCH-EDGE, BRANCH-R1**.
- Mutual OSPF <-> EIGRP redistribution with tag-based loop prevention: **BRANCH-EDGE** (the NSSA/EIGRP boundary).

**Section 6 — DMVPN overlay**
- Underlay addressing + static default, hub tunnel, EIGRP over tunnel: **WAN1** (hub).
- Underlay + spoke tunnel (NHS/NHRP maps to hub), EIGRP over tunnel: **BRANCH-EDGE** (spoke).

---

## Addressing plan

Loopbacks double as router-IDs and BGP/OSPF anchors.

| Device | Loopback0 | Role |
|---|---|---|
| CORE1 | 10.0.0.1/32 | OSPF Area 0, iBGP, eBGP to ISP1 |
| CORE2 | 10.0.0.2/32 | OSPF Area 0, iBGP, eBGP to ISP2 |
| DIST1 | 10.0.0.3/32 | OSPF Area 0, HSRP active VLAN10 |
| DIST2 | 10.0.0.4/32 | OSPF Area 0, HSRP active VLAN20 |
| WAN1  | 10.0.0.7/32 | OSPF NSSA ABR, DMVPN hub |
| BRANCH-EDGE | 10.0.0.8/32 | NSSA/EIGRP boundary, DMVPN spoke |
| BRANCH-R1 | 10.0.0.9/32 | EIGRP interior |
| ISP1 | 172.16.0.1/32 | eBGP AS 65001 |
| ISP2 | 172.16.0.2/32 | eBGP AS 65002 |

Point-to-point transit links (all **/31**, RFC 3021 — both addresses usable on a 2-host link):

| Link | Subnet | Left / Right |
|---|---|---|
| CORE1–CORE2 | 10.1.0.0/31 | .0 / .1 |
| CORE1–DIST1 | 10.1.0.2/31 | .2 / .3 |
| CORE1–DIST2 | 10.1.0.4/31 | .4 / .5 |
| CORE2–DIST1 | 10.1.0.6/31 | .6 / .7 |
| CORE2–DIST2 | 10.1.0.8/31 | .8 / .9 |
| DIST1–DIST2 | 10.1.0.10/31 | .10 / .11 |
| CORE2–WAN1 | 10.1.0.12/31 | .12 / .13 |
| WAN1–BRANCH-EDGE | 10.1.0.14/31 | .14 / .15 |
| BRANCH-EDGE–BRANCH-R1 | 10.2.0.0/31 | .0 / .1 |

> On IOS a /31 is configured with mask `255.255.255.254`. The router warns
> `% Warning: use /31 mask on non point-to-point interface is not recommended` on
> multi-access ports — it's fine on these routed p2p links. On Ethernet, OSPF still
> defaults to the **broadcast** network type regardless of the /31, so it will hold a
> pointless DR/BDR election on each two-router link. Setting `ip ospf network
> point-to-point` on every transit interface (done throughout Section 1) fixes that:
> no DR election, faster adjacency, and a cleaner LSDB.

Access / services:

| Segment | Subnet | Notes |
|---|---|---|
| VLAN10 (data) | 192.168.10.0/24 | GW .1 HSRP VIP; DIST1 .2, DIST2 .3 |
| VLAN20 (voice/data) | 192.168.20.0/24 | GW .1 HSRP VIP; DIST2 .2, DIST1 .3 |
| VLAN99 (mgmt) | 192.168.99.0/24 | switch SVIs |
| Services (SRV-CORE) | 192.168.50.0/24 | DHCP/NTP/syslog host |
| Branch LAN | 10.2.10.0/24 | behind BRANCH-R1 |

Internet underlay (DMVPN):

| Segment | Subnet | Notes |
|---|---|---|
| INET cloud | 100.64.0.0/24 | ISP1 .1, ISP2 .2, WAN1 .7, BRANCH-EDGE .8 |
| eBGP CORE1–ISP1 | 203.0.113.0/31 | .0 CORE1 / .1 ISP1 |
| eBGP CORE2–ISP2 | 203.0.113.2/31 | .2 CORE2 / .3 ISP2 |

Tunnel (DMVPN): `172.20.0.0/24` — WAN1 hub .1, BRANCH-EDGE spoke .8.

---

## Section 1 — OSPF core (the required backbone)

Goal: a clean Area 0 across CORE1/CORE2/DIST1/DIST2, an NSSA out to the branch, and a
stub toward the services segment. OSPF is turned on **at each interface** with
`ip ospf 1 area <n>`; the router process only carries global settings (router-id,
reference-bandwidth, area types).

### 1.1 Interfaces, loopbacks, and per-interface OSPF (example: CORE1)

Address the interface, set the OSPF network type to point-to-point, and enable OSPF in
the area — all in the interface stanza.

```
enable
configure terminal
hostname CORE1
interface Loopback0
 ip address 10.0.0.1 255.255.255.255
 ip ospf 1 area 0
interface GigabitEthernet0/1
 description to CORE2
 ip address 10.1.0.0 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet0/2
 description to DIST1
 ip address 10.1.0.2 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet0/3
 description to DIST2
 ip address 10.1.0.4 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
```

Repeat per device using its own addresses. Every routed link gets `no shutdown`,
`ip ospf 1 area 0`, and `ip ospf network point-to-point`.

> The loopback carries `ip ospf 1 area 0` too, so the router-ID prefix is advertised.
> A loopback is a stub host by default (advertised as a /32) — no network-type change
> needed there.

> **DIST1/DIST2 are multilayer switches (IOSvL2), not routers.** CORE1/CORE2/WAN1 are
> IOSv routers where every interface is already L3, but on the distribution switches a
> physical port defaults to a **switchport**. Any port carrying a routed /31 (the uplinks
> to the core and the DIST1 <-> DIST2 link) must be converted to a routed port with
> `no switchport` *before* you add the IP address and OSPF. Enable `ip routing` globally
> on both (it's in their startup config). The DIST transit-interface pattern is:
>
> ```
> ip routing
> interface GigabitEthernet0/1
>  description to CORE1
>  no switchport
>  ip address 10.1.0.3 255.255.255.254
>  ip ospf 1 area 0
>  ip ospf network point-to-point
>  no shutdown
> ```
>
> The ACC-facing ports on DIST (the EtherChannel members and the trunk in Section 2.2)
> stay as switchports — do **not** put `no switchport` on those. Only the core/peer /31s
> and the inter-VLAN SVIs are L3.

### 1.2 OSPF process (global settings only)

With interfaces doing the enabling, the process stanza is short — just the router-ID and
the reference-bandwidth. No `network` statements at all.

```
router ospf 1
 router-id 10.0.0.1
 auto-cost reference-bandwidth 100000
```

Do the same on CORE2, DIST1, DIST2 with their own router-IDs. The reference-bandwidth
must match on every router — set it everywhere.

> **Passive interfaces:** with interface-mode OSPF you still control adjacency formation
> from the process with `passive-interface`. Since the only OSPF-enabled data links are
> the transit interfaces (and the loopback, which is passive-safe), you generally don't
> need `passive-interface default` here. If you later enable OSPF on an access-facing SVI,
> mark it passive under `router ospf 1` so it advertises the subnet without forming
> neighbors on the user segment.

### 1.3 Authentication (SHA key chain on transit)

Key-chain authentication is applied on the interface, which pairs naturally with the
interface-mode style.

```
key chain OSPF-KC
 key 1
  key-string CorePass123
  cryptographic-algorithm hmac-sha-256
interface GigabitEthernet0/1
 ip ospf authentication key-chain OSPF-KC
```

Apply the same key-chain reference on both ends of every Area 0 link.

### 1.4 DR/BDR control

Because every transit interface is set to `ip ospf network point-to-point` (Section 1.1),
there is **no DR/BDR election** on these links — which is exactly what you want on a
two-router segment. If you ever add a genuine multi-access OSPF segment (e.g. three+
routers on a shared VLAN), leave it as the default broadcast type and pin the DR by
raising priority on the preferred router and zeroing it elsewhere:

```
interface GigabitEthernet0/x
 ip ospf priority 255      ! preferred DR
 ! ip ospf priority 0      ! never DR
```

### 1.5 NSSA toward the branch

WAN1 is the ABR between Area 0 and the NSSA (Area 20). Declare the area type in the
process, and enable OSPF per interface with the correct area on each link.

On WAN1:
```
router ospf 1
 router-id 10.0.0.7
 auto-cost reference-bandwidth 100000
 area 20 nssa
interface GigabitEthernet0/1
 description to CORE2
 ip address 10.1.0.13 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet0/2
 description to BRANCH-EDGE
 ip address 10.1.0.14 255.255.255.254
 ip ospf 1 area 20
 ip ospf network point-to-point
 no shutdown
```

On BRANCH-EDGE:
```
router ospf 1
 router-id 10.0.0.8
 auto-cost reference-bandwidth 100000
 area 20 nssa
interface GigabitEthernet0/1
 description to WAN1
 ip address 10.1.0.15 255.255.255.254
 ip ospf 1 area 20
 ip ospf network point-to-point
 no shutdown
```

> The `area 20 nssa` statement lives in the process on both the ABR (WAN1) and the
> internal router (BRANCH-EDGE) — area *type* is a per-area property, so it stays in the
> router stanza. Only the *enablement* moved to the interface.

### 1.6 Stub toward services

Put the services segment in a stub area (Area 10) so the services host sees only a default
route. Declare the stub in the process; enable OSPF on the SVI/interface facing the
services segment and mark it passive so it advertises the subnet without forming a
neighbor on the host segment.

```
router ospf 1
 area 10 stub
 passive-interface Vlan50
interface Vlan50
 ip address 192.168.50.1 255.255.255.0
 ip ospf 1 area 10
```

> For a *totally stubby* area (challenge step 5), configure `area 10 stub no-summary` on
> the ABR only. The host then sees a single O*IA default instead of inter-area prefixes.

### Verify Section 1
```
show ip ospf neighbor
show ip ospf interface brief
show ip ospf interface GigabitEthernet0/1   ! confirm "Network Type POINT_TO_POINT"
show ip route ospf
show ip ospf database
```
All core/dist neighbors FULL; transit interfaces show network type POINT_TO_POINT with no
DR/BDR; NSSA type-7 LSAs visible; stub area shows O*IA default.

### Full config for this section

_Only this section's commands, per device it touches. Paste into each device in config mode._

**CORE1**
```
hostname CORE1
!
interface Loopback0
 ip address 10.0.0.1 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/1
 description to CORE2
 ip address 10.1.0.0 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
interface GigabitEthernet0/2
 description to DIST1
 ip address 10.1.0.2 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
interface GigabitEthernet0/3
 description to DIST2
 ip address 10.1.0.4 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
!
key chain OSPF-KC
 key 1
  key-string CorePass123
  cryptographic-algorithm hmac-sha-256
!
router ospf 1
 router-id 10.0.0.1
 auto-cost reference-bandwidth 100000
```
**CORE2**
```
hostname CORE2
!
interface Loopback0
 ip address 10.0.0.2 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/1
 description to CORE1
 ip address 10.1.0.1 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
interface GigabitEthernet0/2
 description to DIST1
 ip address 10.1.0.6 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
interface GigabitEthernet0/3
 description to DIST2
 ip address 10.1.0.8 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
interface GigabitEthernet0/4
 description to WAN1
 ip address 10.1.0.12 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
!
key chain OSPF-KC
 key 1
  key-string CorePass123
  cryptographic-algorithm hmac-sha-256
!
router ospf 1
 router-id 10.0.0.2
 auto-cost reference-bandwidth 100000
```
**DIST1**
```
hostname DIST1
ip routing
!
interface Loopback0
 ip address 10.0.0.3 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/1
 description to CORE1
 no switchport
 ip address 10.1.0.3 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
interface GigabitEthernet0/2
 description to CORE2
 no switchport
 ip address 10.1.0.7 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
interface GigabitEthernet0/3
 description to DIST2 (peer link)
 no switchport
 ip address 10.1.0.10 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
!
key chain OSPF-KC
 key 1
  key-string CorePass123
  cryptographic-algorithm hmac-sha-256
!
router ospf 1
 router-id 10.0.0.3
 auto-cost reference-bandwidth 100000
```
**DIST2**
```
hostname DIST2
ip routing
!
interface Loopback0
 ip address 10.0.0.4 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/1
 description to CORE1
 no switchport
 ip address 10.1.0.5 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
interface GigabitEthernet0/2
 description to CORE2
 no switchport
 ip address 10.1.0.9 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
interface GigabitEthernet0/3
 description to DIST1 (peer link)
 no switchport
 ip address 10.1.0.11 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf authentication key-chain OSPF-KC
 no shutdown
!
key chain OSPF-KC
 key 1
  key-string CorePass123
  cryptographic-algorithm hmac-sha-256
!
router ospf 1
 router-id 10.0.0.4
 auto-cost reference-bandwidth 100000
```
**WAN1**
```
hostname WAN1
!
interface Loopback0
 ip address 10.0.0.7 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/1
 description to CORE2
 ip address 10.1.0.13 255.255.255.254
 ip ospf 1 area 0
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet0/2
 description to BRANCH-EDGE
 ip address 10.1.0.14 255.255.255.254
 ip ospf 1 area 20
 ip ospf network point-to-point
 no shutdown
!
router ospf 1
 router-id 10.0.0.7
 auto-cost reference-bandwidth 100000
 area 20 nssa
```
**BRANCH-EDGE**
```
hostname BRANCH-EDGE
!
interface Loopback0
 ip address 10.0.0.8 255.255.255.255
 ip ospf 1 area 20
!
interface GigabitEthernet0/1
 description to WAN1
 ip address 10.1.0.15 255.255.255.254
 ip ospf 1 area 20
 ip ospf network point-to-point
 no shutdown
!
router ospf 1
 router-id 10.0.0.8
 auto-cost reference-bandwidth 100000
 area 20 nssa
```
**SRV-CORE (or DIST1 services SVI)**
```
! Services stub segment (Area 10). If a dedicated router fronts SRV-CORE,
! apply there; otherwise this is DIST1's services SVI.
!
interface Vlan50
 ip address 192.168.50.1 255.255.255.0
 ip ospf 1 area 10
!
router ospf 1
 area 10 stub
 passive-interface Vlan50
```


---

## Section 2 — Layer 2 access

Goal: VLANs, trunks, RSTP root placement, LACP EtherChannel to the distribution pair,
and access-port hardening. Configure on ACC1/ACC2/ACC3 (IOSvL2).

### 2.1 VLANs and access ports (ACC1)

```
vlan 10
 name DATA
vlan 20
 name VOICE
vlan 99
 name MGMT
interface range GigabitEthernet0/3 - 3
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### 2.2 Trunks / EtherChannel to distribution

There's no VSS/StackWise here, so a single EtherChannel can't span the two distribution
chassis. The working design instead is: **one real LACP bundle from ACC1 to a single
distribution switch (DIST1)**, plus **one independent backup trunk to DIST2** that STP
keeps in a blocking state until the primary path fails.

To make the bundle possible the topology gives ACC1 two physical links to DIST1
(`Gi0/1` + `Gi0/3` -> DIST1 `Gi1/0` + `Gi1/3`) and one link to DIST2 (`Gi0/2` -> DIST2
`Gi1/0`).

**On ACC1 — bundle the two DIST1 links, trunk the port-channel:**
```
interface range GigabitEthernet0/1, GigabitEthernet0/3
 channel-group 1 mode active
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
```

**On ACC1 — the single DIST2 link as an independent backup trunk (no channel-group):**
```
interface GigabitEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
```

**On DIST1 — matching port-channel on the two ACC1-facing ports:**
```
interface range GigabitEthernet1/0, GigabitEthernet1/3
 channel-group 1 mode active
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
```

**On DIST2 — the single ACC1-facing port as a plain trunk:**
```
interface GigabitEthernet1/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
```

Because DIST1 is STP root for VLAN 10 (Section 2.3), the bundled Po1 path wins and the
DIST2 backup trunk lands in blocking — you'll see it go forwarding only when you shut the
bundle. That's the redundancy demonstration: a real, forming LACP channel *and* an
STP-managed standby, without pretending a cross-chassis bundle works.

ACC2 (single link to DIST1 `Gi1/1`) and ACC3 (single link to DIST2 `Gi1/1`) have only one
uplink each, so they're plain access-switch trunks — no EtherChannel:
```
! ACC2 Gi0/1  and  ACC3 Gi0/1
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
```

### 2.3 STP root placement (RSTP)

Make the distribution switches the root and backup for the access VLANs. On DIST1:

```
spanning-tree mode rapid-pvst
spanning-tree vlan 10 root primary
spanning-tree vlan 20 root secondary
```

Swap primary/secondary for VLAN 20 on DIST2 so the two VLANs use different roots.

### 2.4 Access-port security

```
interface GigabitEthernet0/3
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
```

Enable DHCP snooping and dynamic ARP inspection on the access switches:

Trust every uplink toward distribution — the bundled `Port-channel1` **and** the backup
trunk `Gi0/2` — so DHCP/ARP still pass when STP moves traffic to the backup path:
```
ip dhcp snooping
ip dhcp snooping vlan 10,20
interface Port-channel1
 ip dhcp snooping trust
 ip arp inspection trust
interface GigabitEthernet0/2
 ip dhcp snooping trust
 ip arp inspection trust
ip arp inspection vlan 10,20
```

### Verify Section 2
```
show vlan brief
show etherchannel summary          ! Po1 = P (in port-channel), both member ports bundled
show spanning-tree vlan 10         ! Po1 forwarding (root DIST1); Gi0/2 backup blocking
show port-security
show ip dhcp snooping
```
Failover check: `shutdown` the two Po1 member ports on ACC1 -> the DIST2 backup trunk
(`Gi0/2`) transitions to forwarding and VLAN 10 stays reachable; `no shutdown` restores
the bundle and the backup returns to blocking.

### Full config for this section

_Only this section's commands, per device it touches. Paste into each device in config mode._

**ACC1**
```
hostname ACC1
!
vlan 10
 name DATA
vlan 20
 name VOICE
vlan 99
 name MGMT
!
interface range GigabitEthernet0/1, GigabitEthernet0/3
 channel-group 1 mode active
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 ip dhcp snooping trust
 ip arp inspection trust
!
interface GigabitEthernet0/2
 description backup trunk to DIST2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 ip dhcp snooping trust
 ip arp inspection trust
!
spanning-tree mode rapid-pvst
ip dhcp snooping
ip dhcp snooping vlan 10,20
ip arp inspection vlan 10,20
```
**ACC2**
```
hostname ACC2
!
vlan 10
 name DATA
vlan 20
 name VOICE
vlan 99
 name MGMT
!
interface GigabitEthernet0/1
 description uplink to DIST1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 ip dhcp snooping trust
 ip arp inspection trust
!
interface GigabitEthernet0/2
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
!
spanning-tree mode rapid-pvst
ip dhcp snooping
ip dhcp snooping vlan 10,20
ip arp inspection vlan 10,20
```
**ACC3**
```
hostname ACC3
!
vlan 10
 name DATA
vlan 20
 name VOICE
vlan 99
 name MGMT
!
interface GigabitEthernet0/1
 description uplink to DIST2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 ip dhcp snooping trust
 ip arp inspection trust
!
spanning-tree mode rapid-pvst
ip dhcp snooping
ip dhcp snooping vlan 10,20
ip arp inspection vlan 10,20
```
**DIST1**
```
hostname DIST1
!
vlan 10
 name DATA
vlan 20
 name VOICE
vlan 99
 name MGMT
!
! ACC1 EtherChannel (root for VLAN 10)
interface range GigabitEthernet1/0, GigabitEthernet1/3
 channel-group 1 mode active
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
!
! ACC2 single-link trunk
interface GigabitEthernet1/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
!
spanning-tree mode rapid-pvst
spanning-tree vlan 10 root primary
spanning-tree vlan 20 root secondary
```
**DIST2**
```
hostname DIST2
!
vlan 10
 name DATA
vlan 20
 name VOICE
vlan 99
 name MGMT
!
! ACC1 backup trunk (single link, STP-blocked for VLAN 10)
interface GigabitEthernet1/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
!
! ACC3 single-link trunk
interface GigabitEthernet1/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
!
spanning-tree mode rapid-pvst
spanning-tree vlan 20 root primary
spanning-tree vlan 10 root secondary
```


---

## Section 3 — FHRP + IP SLA tracking

Goal: HSRP on the distribution SVIs with an IP SLA + tracked object that fails the active
role over when the upstream path dies. DIST1 active for VLAN10, DIST2 active for VLAN20.

### 3.1 SVIs and HSRP (DIST1)

```
interface Vlan10
 ip address 192.168.10.2 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.1
 standby 10 priority 110
 standby 10 preempt
interface Vlan20
 ip address 192.168.20.3 255.255.255.0
 standby version 2
 standby 20 ip 192.168.20.1
 standby 20 priority 90
 standby 20 preempt
```

On DIST2 mirror it: VLAN10 priority 90, VLAN20 priority 110, same VIPs and `.` host
addresses swapped per the plan.

> If you advertise the VLAN10/20 subnets into OSPF, enable OSPF on these SVIs with
> `ip ospf 1 area 0` and mark them `passive-interface` under `router ospf 1` — the subnet
> gets advertised but no neighbor forms toward the hosts.

### 3.2 IP SLA probe (DIST1 -> upstream)

Probe an always-on upstream target (e.g. CORE2's loopback reachable via the core).

```
ip sla 1
 icmp-echo 10.0.0.2 source-interface GigabitEthernet0/1
 frequency 5
ip sla schedule 1 life forever start-time now
```

### 3.3 Track the SLA and decrement HSRP

```
track 1 ip sla 1 reachability
interface Vlan10
 standby 10 track 1 decrement 30
```

When the probe fails, priority drops 110 -> 80 and (with preempt on DIST2) VLAN10 fails over.
Add a second SLA/track on DIST2 for VLAN20 so each has its own upstream dependency.

### Verify Section 3
```
show standby brief
show track
show ip sla statistics
```
Shut DIST1 Gi0/1 and confirm VLAN10 active moves to DIST2, then recovers on `no shut`.

### Full config for this section

_Only this section's commands, per device it touches. Paste into each device in config mode._

**DIST1**
```
hostname DIST1
!
interface Vlan10
 ip address 192.168.10.2 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.1
 standby 10 priority 110
 standby 10 preempt
 standby 10 track 1 decrement 30
interface Vlan20
 ip address 192.168.20.3 255.255.255.0
 standby version 2
 standby 20 ip 192.168.20.1
 standby 20 priority 90
 standby 20 preempt
!
ip sla 1
 icmp-echo 10.0.0.2 source-interface GigabitEthernet0/1
 frequency 5
ip sla schedule 1 life forever start-time now
!
track 1 ip sla 1 reachability
```
**DIST2**
```
hostname DIST2
!
interface Vlan10
 ip address 192.168.10.3 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.1
 standby 10 priority 90
 standby 10 preempt
interface Vlan20
 ip address 192.168.20.2 255.255.255.0
 standby version 2
 standby 20 ip 192.168.20.1
 standby 20 priority 110
 standby 20 preempt
 standby 20 track 2 decrement 30
!
ip sla 2
 icmp-echo 10.0.0.1 source-interface GigabitEthernet0/2
 frequency 5
ip sla schedule 2 life forever start-time now
!
track 2 ip sla 2 reachability
```


---

## Section 4 — BGP (eBGP to ISP, iBGP across core)

Goal: eBGP from each core to its ISP, iBGP between the cores, and clean next-hop handling.
Local AS is 65000; ISP1 is 65001, ISP2 is 65002.

### 4.1 eBGP edges

On CORE1 (Gi0/4 -> ISP1, 203.0.113.0/31):
```
interface GigabitEthernet0/4
 ip address 203.0.113.0 255.255.255.254
 no shutdown
router bgp 65000
 bgp router-id 10.0.0.1
 neighbor 203.0.113.1 remote-as 65001
 address-family ipv4
  neighbor 203.0.113.1 activate
  network 10.0.0.0 mask 255.255.0.0
```

> Leave the eBGP interface **out** of OSPF (no `ip ospf` line on Gi0/4) — the ISP link is
> a BGP-only edge. The `network 10.0.0.0 mask 255.255.0.0` here is a *BGP* origination
> statement, unrelated to OSPF enablement.

On ISP1 (mirror at 203.0.113.1, remote-as 65000, advertise a test prefix like
8.8.8.0/24). Repeat CORE2 <-> ISP2 on 203.0.113.2/31 (CORE2 .2, ISP2 .3) with AS 65002.

### 4.2 iBGP across the core

Peer the cores over loopbacks with next-hop-self so eBGP routes stay reachable.

```
router bgp 65000
 neighbor 10.0.0.2 remote-as 65000
 neighbor 10.0.0.2 update-source Loopback0
 address-family ipv4
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 next-hop-self
```

Mirror on CORE2 toward 10.0.0.1. (The loopbacks are reachable because they're advertised
in OSPF via `ip ospf 1 area 0` on Loopback0.)

### 4.3 Path manipulation (practice)

- Steer *our outbound* traffic toward ISP1 with local-preference. It's set on routes coming *in* from ISP1 (route-map applied `in` on CORE1), but higher local-pref makes ISP1 our preferred *exit*.
- Steer *inbound* traffic to arrive via ISP1 by prepending our AS on advertisements sent *out* to ISP2 (on CORE2), making the ISP2 path look worse to the outside world.

```
route-map LP-IN permit 10
 set local-preference 200
router bgp 65000
 address-family ipv4
  neighbor 203.0.113.1 route-map LP-IN in
```

### Verify Section 4
```
show ip bgp summary
show ip bgp
show ip route bgp
```
eBGP + iBGP sessions Established; ISP test prefixes present with expected best-path.

### Full config for this section

_Only this section's commands, per device it touches. Paste into each device in config mode._

**CORE1**
```
hostname CORE1
!
interface GigabitEthernet0/4
 description to ISP1 (eBGP, no OSPF)
 ip address 203.0.113.0 255.255.255.254
 no shutdown
!
route-map LP-IN permit 10
 set local-preference 200
!
router bgp 65000
 bgp router-id 10.0.0.1
 neighbor 203.0.113.1 remote-as 65001
 neighbor 10.0.0.2 remote-as 65000
 neighbor 10.0.0.2 update-source Loopback0
 address-family ipv4
  neighbor 203.0.113.1 activate
  neighbor 203.0.113.1 route-map LP-IN in
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 next-hop-self
  network 10.0.0.0 mask 255.255.0.0
```
**CORE2**
```
hostname CORE2
!
interface GigabitEthernet0/5
 description to ISP2 (eBGP, no OSPF)
 ip address 203.0.113.2 255.255.255.254
 no shutdown
!
router bgp 65000
 bgp router-id 10.0.0.2
 neighbor 203.0.113.3 remote-as 65002
 neighbor 10.0.0.1 remote-as 65000
 neighbor 10.0.0.1 update-source Loopback0
 address-family ipv4
  neighbor 203.0.113.3 activate
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.1 next-hop-self
  network 10.0.0.0 mask 255.255.0.0
```
**ISP1**
```
hostname ISP1
!
interface GigabitEthernet0/1
 description to CORE1
 ip address 203.0.113.1 255.255.255.254
 no shutdown
!
interface Loopback8
 ip address 8.8.8.8 255.255.255.0
!
router bgp 65001
 bgp router-id 172.16.0.1
 neighbor 203.0.113.0 remote-as 65000
 address-family ipv4
  neighbor 203.0.113.0 activate
  network 8.8.8.0 mask 255.255.255.0
```
**ISP2**
```
hostname ISP2
!
interface GigabitEthernet0/1
 description to CORE2
 ip address 203.0.113.3 255.255.255.254
 no shutdown
!
interface Loopback9
 ip address 9.9.9.9 255.255.255.0
!
router bgp 65002
 bgp router-id 172.16.0.2
 neighbor 203.0.113.2 remote-as 65000
 address-family ipv4
  neighbor 203.0.113.2 activate
  network 9.9.9.0 mask 255.255.255.0
```


---

## Section 5 — Redistribution (OSPF <-> EIGRP)

Goal: run EIGRP in the branch, make BRANCH-EDGE the OSPF-NSSA/EIGRP boundary, and do
mutual redistribution safely with tags and filtering.

### 5.1 EIGRP in the branch

On BRANCH-R1 and BRANCH-EDGE (AS 100):
```
router eigrp 100
 network 10.2.0.0 0.0.0.1
 network 10.2.10.0 0.0.0.255
 no auto-summary
```

> Enable EIGRP only on the branch-interior links here — not on the WAN1-facing interface,
> which stays in OSPF Area 20 (`ip ospf 1 area 20`, Section 1.5). BRANCH-EDGE runs both
> protocols, and the boundary between them is what Section 5.2 redistributes.

### 5.2 Mutual redistribution on BRANCH-EDGE

Tag routes as they cross so they can be filtered on the return trip (loop prevention).

```
route-map OSPF-TO-EIGRP permit 10
 set tag 200
route-map EIGRP-TO-OSPF deny 10
 match tag 200
route-map EIGRP-TO-OSPF permit 20
 set tag 100
router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
router eigrp 100
 redistribute ospf 1 metric 100000 100 255 1 1500 route-map OSPF-TO-EIGRP
```

The `deny` for tag 200 stops OSPF-origin routes that were redistributed into EIGRP from
being pushed back into OSPF.

### Verify Section 5
```
show ip route eigrp
show ip route ospf
show ip ospf database nssa-external
show route-map
```
Branch LAN appears in the core as an external; no route flaps or duplicate paths.

### Full config for this section

_Only this section's commands, per device it touches. Paste into each device in config mode._

**BRANCH-EDGE**
```
hostname BRANCH-EDGE
!
interface GigabitEthernet0/2
 description to BRANCH-R1 (EIGRP)
 ip address 10.2.0.0 255.255.255.254
 no shutdown
!
route-map OSPF-TO-EIGRP permit 10
 set tag 200
route-map EIGRP-TO-OSPF deny 10
 match tag 200
route-map EIGRP-TO-OSPF permit 20
 set tag 100
!
router eigrp 100
 network 10.2.0.0 0.0.0.1
 no auto-summary
 redistribute ospf 1 metric 100000 100 255 1 1500 route-map OSPF-TO-EIGRP
!
router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
```
**BRANCH-R1**
```
hostname BRANCH-R1
!
interface GigabitEthernet0/1
 description to BRANCH-EDGE
 ip address 10.2.0.1 255.255.255.254
 no shutdown
interface GigabitEthernet0/2
 description to BRANCH-SW (branch LAN)
 ip address 10.2.10.1 255.255.255.0
 no shutdown
!
router eigrp 100
 network 10.2.0.0 0.0.0.1
 network 10.2.10.0 0.0.0.255
 no auto-summary
```


---

## Section 6 — DMVPN overlay

Goal: a Phase 1 hub-and-spoke tunnel with WAN1 as hub and BRANCH-EDGE as spoke, over the
INET underlay (100.64.0.0/24). Extend to Phase 3 later.

### 6.1 Underlay

Give the internet-facing interfaces their 100.64.0.x addresses and a default route toward
the ISP cloud so the tunnel endpoints can reach each other. These interfaces stay **out**
of OSPF — the underlay is reached by static default, and the overlay runs its own routing.

```
! WAN1
interface GigabitEthernet0/3
 ip address 100.64.0.7 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 100.64.0.1
```
Do the equivalent on BRANCH-EDGE (100.64.0.8).

### 6.2 Hub tunnel (WAN1)

```
interface Tunnel0
 ip address 172.20.0.1 255.255.255.0
 tunnel source GigabitEthernet0/3
 tunnel mode gre multipoint
 ip nhrp network-id 1
 ip nhrp map multicast dynamic
 no ip split-horizon eigrp 100
```

### 6.3 Spoke tunnel (BRANCH-EDGE)

```
interface Tunnel0
 ip address 172.20.0.8 255.255.255.0
 tunnel source GigabitEthernet0/3
 tunnel mode gre multipoint
 ip nhrp network-id 1
 ip nhrp nhs 172.20.0.1
 ip nhrp map 172.20.0.1 100.64.0.7
 ip nhrp map multicast 100.64.0.7
```

### 6.4 Run a routing protocol over the tunnel

Extend EIGRP AS 100 across the tunnel subnet so branch prefixes reach the hub via the
overlay:
```
router eigrp 100
 network 172.20.0.0 0.0.0.255
```

> If you'd rather run OSPF over the tunnel instead of EIGRP, enable it per interface with
> `ip ospf 1 area 20` on Tunnel0 at both ends and set `ip ospf network point-to-multipoint`
> on the hub (broadcast/NBMA types get messy over DMVPN). Keep it EIGRP for this lab so
> Section 5's redistribution story stays intact.

### Verify Section 6
```
show dmvpn
show ip nhrp
show ip eigrp neighbors
ping 172.20.0.1 source 172.20.0.8
```
NHRP registration up; EIGRP neighbor over Tunnel0; branch reachable across the overlay.

### Full config for this section

_Only this section's commands, per device it touches. Paste into each device in config mode._

**WAN1**
```
hostname WAN1
!
interface GigabitEthernet0/3
 description INET underlay
 ip address 100.64.0.7 255.255.255.0
 no shutdown
!
interface Tunnel0
 ip address 172.20.0.1 255.255.255.0
 tunnel source GigabitEthernet0/3
 tunnel mode gre multipoint
 ip nhrp network-id 1
 ip nhrp map multicast dynamic
 no ip split-horizon eigrp 100
!
ip route 0.0.0.0 0.0.0.0 100.64.0.1
!
router eigrp 100
 network 172.20.0.0 0.0.0.255
```
**BRANCH-EDGE**
```
hostname BRANCH-EDGE
!
interface GigabitEthernet0/3
 description INET underlay
 ip address 100.64.0.8 255.255.255.0
 no shutdown
!
interface Tunnel0
 ip address 172.20.0.8 255.255.255.0
 tunnel source GigabitEthernet0/3
 tunnel mode gre multipoint
 ip nhrp network-id 1
 ip nhrp nhs 172.20.0.1
 ip nhrp map 172.20.0.1 100.64.0.7
 ip nhrp map multicast 100.64.0.7
!
ip route 0.0.0.0 0.0.0.0 100.64.0.1
!
router eigrp 100
 network 172.20.0.0 0.0.0.255
```


---

## Final integration check

- Access host in VLAN10 pings the services host, the branch LAN, and an ISP test prefix.
- Failover: shut DIST1 upstream -> VLAN10 HSRP moves to DIST2 -> traffic still flows.
- Shut CORE1–ISP1 -> BGP best path shifts to ISP2 -> internet prefixes stay reachable.
- Shut the WAN1–BRANCH-EDGE Area 20 link -> branch still reachable via the DMVPN overlay.

---

# CHALLENGE — steps only, no guide

Rebuild the following from a blank topology. No command hints. Verify each before moving on.

1. Bring up all loopbacks and transit /31s per the addressing plan.
2. Enable OSPF **per interface** (`ip ospf 1 area 0`) across CORE1, CORE2, DIST1, DIST2 — no `network` statements — with matching reference-bandwidth.
3. Set every transit interface to `ip ospf network point-to-point` and confirm no DR/BDR is elected.
4. Add SHA-256 authentication on every Area 0 adjacency.
5. Make WAN1 an ABR: interfaces in Area 0 to the core, Area 20 to BRANCH-EDGE, with `area 20 nssa` in the process.
6. Put the services segment in a totally stubby area (`area 10 stub no-summary` on the ABR); confirm the host sees only a default route.
7. Create VLANs 10/20/99 on the access switches with correct trunks to distribution.
8. Build a real LACP EtherChannel from ACC1 to DIST1 (two links, `Gi0/1`+`Gi0/3`), and configure the single ACC1 -> DIST2 link as an independent backup trunk; confirm the bundle forms and the backup blocks.
9. Set RSTP so VLAN10 and VLAN20 root on different distribution switches.
10. Harden all access ports: portfast, BPDU guard, port-security, DHCP snooping, DAI.
11. Configure HSRPv2: DIST1 active VLAN10, DIST2 active VLAN20, both with preempt.
12. Add an IP SLA + tracked object on each active router that decrements HSRP priority on upstream failure; prove failover and recovery.
13. Bring up eBGP CORE1 <-> ISP1 (65000/65001) and CORE2 <-> ISP2 (65000/65002) — keep the ISP links out of OSPF.
14. Bring up iBGP CORE1 <-> CORE2 over loopbacks with next-hop-self.
15. Make ISP1 the primary inbound and outbound path using local-preference and AS-path prepending.
16. Run EIGRP AS 100 across the branch (BRANCH-EDGE, BRANCH-R1, branch LAN).
17. Mutually redistribute OSPF and EIGRP on BRANCH-EDGE with route tags that prevent a redistribution loop.
18. Build a DMVPN Phase 1 tunnel: WAN1 hub, BRANCH-EDGE spoke, over the 100.64.0.0/24 underlay.
19. Carry branch prefixes to the hub over the tunnel with EIGRP.
20. Prove all four failure scenarios in the integration check pass.
21. Bonus: dual-stack the core with OSPFv3 — enable it per interface with `ipv6 ospf 1 area 0` — and re-verify Area 0.
22. Bonus: convert DMVPN to Phase 3 and confirm spoke-to-spoke shortcut (add a second spoke).
