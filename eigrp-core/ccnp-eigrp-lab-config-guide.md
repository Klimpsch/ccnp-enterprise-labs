# CCNP Enterprise EIGRP-Core Lab — Configuration Guide

A start-to-finish build guide for the `CCNP-Enterprise-EIGRP-Core` topology. EIGRP is
the core routing protocol; work the sections in order, each assuming the previous is up.
The final section is an unguided challenge — steps only.

> Interface names match the imported YAML. `Gi0/0` is the management slot, unused for
> data. Verify the mapping in CML once before starting.

---

## Addressing plan

Loopbacks are router-IDs and EIGRP/BGP anchors.

| Device | Loopback0 | Role |
|---|---|---|
| CORE1 | 10.0.0.1/32 | EIGRP AS100, iBGP, eBGP to ISP1 |
| CORE2 | 10.0.0.2/32 | EIGRP AS100, iBGP, eBGP to ISP2 |
| DIST1 | 10.0.0.3/32 | EIGRP AS100, HSRP active VLAN10 |
| DIST2 | 10.0.0.4/32 | EIGRP AS100, HSRP active VLAN20 |
| WAN1  | 10.0.0.7/32 | EIGRP/OSPF boundary side, DMVPN hub |
| BRANCH-EDGE | 10.0.0.8/32 | EIGRP<->OSPF boundary, DMVPN spoke |
| BRANCH-R1 | 10.0.0.9/32 | OSPF interior (branch) |
| ISP1 | 172.16.0.1/32 | eBGP AS 65001 |
| ISP2 | 172.16.0.2/32 | eBGP AS 65002 |

Point-to-point transit links (all **/31**, RFC 3021; mask `255.255.255.254`):

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

Access / services:

| Segment | Subnet | Notes |
|---|---|---|
| VLAN10 (data) | 192.168.10.0/24 | GW .1 HSRP VIP; DIST1 .2, DIST2 .3 |
| VLAN20 (voice/data) | 192.168.20.0/24 | GW .1 HSRP VIP; DIST2 .2, DIST1 .3 |
| VLAN99 (mgmt) | 192.168.99.0/24 | switch SVIs |
| Services (SRV-CORE) | 192.168.50.0/24 | DHCP/NTP/syslog host, EIGRP stub segment |
| Branch LAN | 10.2.10.0/24 | behind BRANCH-R1 (OSPF) |

Internet underlay (DMVPN):

| Segment | Subnet | Notes |
|---|---|---|
| INET cloud | 100.64.0.0/24 | ISP1 .1, ISP2 .2, WAN1 .7, BRANCH-EDGE .8 |
| eBGP CORE1–ISP1 | 203.0.113.0/31 | .0 CORE1 / .1 ISP1 |
| eBGP CORE2–ISP2 | 203.0.113.2/31 | .2 CORE2 / .3 ISP2 |

Tunnel (DMVPN): `172.20.0.0/24` — WAN1 hub .1, BRANCH-EDGE spoke .8.

---

## Section 1 — EIGRP core (the required backbone)

Goal: a clean EIGRP AS 100 across CORE1/CORE2/DIST1/DIST2 using **named mode**, with
authentication, summarization toward the edges, a stub on the services segment, and
unequal-cost load balancing via variance. Bring up interfaces and loopbacks first.

### 1.1 Interfaces and loopbacks (example: CORE1)

```
enable
configure terminal
hostname CORE1
interface Loopback0
 ip address 10.0.0.1 255.255.255.255
interface GigabitEthernet0/1
 description to CORE2
 ip address 10.1.0.0 255.255.255.254
 no shutdown
interface GigabitEthernet0/2
 description to DIST1
 ip address 10.1.0.2 255.255.255.254
 no shutdown
interface GigabitEthernet0/3
 description to DIST2
 ip address 10.1.0.4 255.255.255.254
 no shutdown
```

Repeat per device using the addressing plan. Every routed link gets `no shutdown`.

> **DIST1/DIST2 are multilayer switches (IOSvL2), not routers.** On the distribution
> switches a physical port defaults to a **switchport**, so any port carrying a routed
> /31 (the core uplinks `Gi0/1`/`Gi0/2` and the DIST1↔DIST2 peer link `Gi0/3`) must be
> made a routed port with `no switchport` before you add the IP address and EIGRP. Enable
> `ip routing` globally on both. Example DIST transit interface:
>
> ```
> ip routing
> interface GigabitEthernet0/1
>  description to CORE1
>  no switchport
>  ip address 10.1.0.3 255.255.255.254
>  no shutdown
> ```
>
> The ACC-facing ports on DIST (EtherChannel members / trunk in Section 2.2) stay as
> switchports — do **not** put `no switchport` on those. Only the core/peer /31s and the
> inter-VLAN SVIs are L3.

### 1.2 EIGRP named mode (CORE1)

Named mode keeps the whole EIGRP config under one `router eigrp` block with an
address-family. Use it on all four core/dist routers.

```
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 10.0.0.1
  network 10.0.0.1 0.0.0.0
  network 10.1.0.0 0.0.0.1
  network 10.1.0.2 0.0.0.1
  network 10.1.0.4 0.0.0.1
  af-interface default
   passive-interface
  exit-af-interface
  af-interface GigabitEthernet0/1
   no passive-interface
  exit-af-interface
  af-interface GigabitEthernet0/2
   no passive-interface
  exit-af-interface
  af-interface GigabitEthernet0/3
   no passive-interface
  exit-af-interface
```

Mirror on CORE2, DIST1, DIST2 with their own router-IDs and transit networks.

### 1.3 Authentication (SHA-256 in named mode)

Named mode supports SHA-256 keychains directly on the address-family interfaces.

```
key chain EIGRP-KC
 key 1
  key-string CampusEIGRP123
  cryptographic-algorithm hmac-sha-256
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  af-interface GigabitEthernet0/1
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
```

Apply on every transit interface on both ends.

### 1.4 Summarization

Summarize the access/services space toward the core so the core sees one prefix. On
DIST1/DIST2 toward the core interfaces:

```
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  af-interface GigabitEthernet0/1
   summary-address 192.168.0.0 255.255.0.0
  exit-af-interface
```

### 1.5 EIGRP stub on the services segment

Make the router facing the services host a stub so it only receives a summary/default and
isn't used as transit. (In this lab DIST1 fronts SRV-CORE; if you dedicate a router to the
services segment, make it the stub.)

```
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  eigrp stub connected summary
```

### 1.6 Unequal-cost load balancing (variance)

With two paths of differing metric between the cores and distribution, allow EIGRP to use
both:

```
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  topology base
   variance 2
   maximum-paths 4
  exit-af-topology
```

### Verify Section 1
```
show ip eigrp neighbors
show ip eigrp interfaces detail
show ip route eigrp
show ip protocols
show ip eigrp topology
```
All core/dist neighbors present; SHA auth up; summary visible in the core; variance paths
show as multiple successors.

### Full config for this section

_Only this section's commands, per device it touches. Paste into each device in config mode._

**CORE1**
```
hostname CORE1
!
key chain EIGRP-KC
 key 1
  key-string CampusEIGRP123
  cryptographic-algorithm hmac-sha-256
!
interface Loopback0
 ip address 10.0.0.1 255.255.255.255
interface GigabitEthernet0/1
 description to CORE2
 ip address 10.1.0.0 255.255.255.254
 no shutdown
interface GigabitEthernet0/2
 description to DIST1
 ip address 10.1.0.2 255.255.255.254
 no shutdown
interface GigabitEthernet0/3
 description to DIST2
 ip address 10.1.0.4 255.255.255.254
 no shutdown
!
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 10.0.0.1
  af-interface default
   passive-interface
  exit-af-interface
  af-interface GigabitEthernet0/1
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  af-interface GigabitEthernet0/2
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  af-interface GigabitEthernet0/3
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  topology base
   variance 2
   maximum-paths 4
  exit-af-topology
  network 10.0.0.1 0.0.0.0
  network 10.1.0.0 0.0.0.1
  network 10.1.0.2 0.0.0.1
  network 10.1.0.4 0.0.0.1
```
**CORE2**
```
hostname CORE2
!
key chain EIGRP-KC
 key 1
  key-string CampusEIGRP123
  cryptographic-algorithm hmac-sha-256
!
interface Loopback0
 ip address 10.0.0.2 255.255.255.255
interface GigabitEthernet0/1
 description to CORE1
 ip address 10.1.0.1 255.255.255.254
 no shutdown
interface GigabitEthernet0/2
 description to DIST1
 ip address 10.1.0.6 255.255.255.254
 no shutdown
interface GigabitEthernet0/3
 description to DIST2
 ip address 10.1.0.8 255.255.255.254
 no shutdown
interface GigabitEthernet0/4
 description to WAN1
 ip address 10.1.0.12 255.255.255.254
 no shutdown
!
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 10.0.0.2
  af-interface default
   passive-interface
  exit-af-interface
  af-interface GigabitEthernet0/1
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  af-interface GigabitEthernet0/2
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  af-interface GigabitEthernet0/3
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  af-interface GigabitEthernet0/4
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  topology base
   variance 2
   maximum-paths 4
  exit-af-topology
  network 10.0.0.2 0.0.0.0
  network 10.1.0.0 0.0.0.1
  network 10.1.0.6 0.0.0.1
  network 10.1.0.8 0.0.0.1
  network 10.1.0.12 0.0.0.1
```
**DIST1**
```
hostname DIST1
ip routing
!
key chain EIGRP-KC
 key 1
  key-string CampusEIGRP123
  cryptographic-algorithm hmac-sha-256
!
interface Loopback0
 ip address 10.0.0.3 255.255.255.255
interface GigabitEthernet0/1
 description to CORE1
 no switchport
 ip address 10.1.0.3 255.255.255.254
 no shutdown
interface GigabitEthernet0/2
 description to CORE2
 no switchport
 ip address 10.1.0.7 255.255.255.254
 no shutdown
interface GigabitEthernet0/3
 description to DIST2 (peer link)
 no switchport
 ip address 10.1.0.10 255.255.255.254
 no shutdown
!
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 10.0.0.3
  af-interface default
   passive-interface
  exit-af-interface
  af-interface GigabitEthernet0/1
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  af-interface GigabitEthernet0/2
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  af-interface GigabitEthernet0/3
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  topology base
   variance 2
   maximum-paths 4
  exit-af-topology
  af-interface GigabitEthernet0/1
   summary-address 192.168.0.0 255.255.0.0
  exit-af-interface
  af-interface GigabitEthernet0/2
   summary-address 192.168.0.0 255.255.0.0
  exit-af-interface
  network 10.0.0.3 0.0.0.0
  network 10.1.0.2 0.0.0.1
  network 10.1.0.6 0.0.0.1
  network 10.1.0.10 0.0.0.1
```
**DIST2**
```
hostname DIST2
ip routing
!
key chain EIGRP-KC
 key 1
  key-string CampusEIGRP123
  cryptographic-algorithm hmac-sha-256
!
interface Loopback0
 ip address 10.0.0.4 255.255.255.255
interface GigabitEthernet0/1
 description to CORE1
 no switchport
 ip address 10.1.0.5 255.255.255.254
 no shutdown
interface GigabitEthernet0/2
 description to CORE2
 no switchport
 ip address 10.1.0.9 255.255.255.254
 no shutdown
interface GigabitEthernet0/3
 description to DIST1 (peer link)
 no switchport
 ip address 10.1.0.11 255.255.255.254
 no shutdown
!
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 10.0.0.4
  af-interface default
   passive-interface
  exit-af-interface
  af-interface GigabitEthernet0/1
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  af-interface GigabitEthernet0/2
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  af-interface GigabitEthernet0/3
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  topology base
   variance 2
   maximum-paths 4
  exit-af-topology
  af-interface GigabitEthernet0/1
   summary-address 192.168.0.0 255.255.0.0
  exit-af-interface
  af-interface GigabitEthernet0/2
   summary-address 192.168.0.0 255.255.0.0
  exit-af-interface
  network 10.0.0.4 0.0.0.0
  network 10.1.0.4 0.0.0.1
  network 10.1.0.8 0.0.0.1
  network 10.1.0.10 0.0.0.1
```
**WAN1**
```
hostname WAN1
!
key chain EIGRP-KC
 key 1
  key-string CampusEIGRP123
  cryptographic-algorithm hmac-sha-256
!
interface Loopback0
 ip address 10.0.0.7 255.255.255.255
interface GigabitEthernet0/1
 description to CORE2
 ip address 10.1.0.13 255.255.255.254
 no shutdown
!
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 10.0.0.7
  af-interface default
   passive-interface
  exit-af-interface
  af-interface GigabitEthernet0/1
   no passive-interface
   authentication mode hmac-sha-256 CampusEIGRP123
  exit-af-interface
  topology base
   variance 2
   maximum-paths 4
  exit-af-topology
  network 10.0.0.7 0.0.0.0
  network 10.1.0.12 0.0.0.1
```
**SRV-CORE segment (EIGRP stub router)**
```
! Services stub. Apply on whichever router fronts SRV-CORE.
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  eigrp stub connected summary
```


---

## Section 2 — Layer 2 access

Protocol-agnostic; identical to the campus L2 build. Configure on ACC1/ACC2/ACC3.

### 2.1 VLANs and access ports (ACC2)

Configure VLANs on every access switch. Use ACC2 for the access-port example — on ACC1,
`Gi0/1`/`Gi0/3` are the EtherChannel to DIST1 and `Gi0/2` is the backup trunk (Section
2.2), so its host ports would be higher-numbered slots. ACC2 has a single uplink on
`Gi0/1`, leaving `Gi0/2`+ free for hosts.
```
vlan 10
 name DATA
vlan 20
 name VOICE
vlan 99
 name MGMT
interface GigabitEthernet0/2
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
```

### 2.2 Trunks / EtherChannel to distribution

No VSS/StackWise, so a single EtherChannel can't span the two distribution chassis. The
working design: **one real LACP bundle from ACC1 to a single distribution switch (DIST1)**
plus **one independent backup trunk to DIST2** that STP keeps blocking until the primary
fails. The topology gives ACC1 two links to DIST1 (`Gi0/1`+`Gi0/3` → DIST1 `Gi1/0`+`Gi1/3`)
and one to DIST2 (`Gi0/2` → DIST2 `Gi1/0`).

**On ACC1 — bundle the two DIST1 links:**
```
interface range GigabitEthernet0/1, GigabitEthernet0/3
 channel-group 1 mode active
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
```

**On ACC1 — the DIST2 link as an independent backup trunk (no channel-group):**
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

With DIST1 as STP root for VLAN 10 (Section 2.3), the bundled Po1 path wins and the DIST2
backup trunk sits in blocking until you shut the bundle — a real forming LACP channel plus
an STP-managed standby. ACC2 (`Gi0/1`→DIST1) and ACC3 (`Gi0/1`→DIST2) have a single uplink
each, so they're plain trunks, no EtherChannel.

### 2.3 STP root placement (RSTP)
```
spanning-tree mode rapid-pvst
spanning-tree vlan 10 root primary
spanning-tree vlan 20 root secondary
```
Swap primary/secondary for VLAN 20 on DIST2.

### 2.4 Access-port security
```
interface GigabitEthernet0/2
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
ip dhcp snooping
ip dhcp snooping vlan 10,20
ip arp inspection vlan 10,20
interface Port-channel1
 ip dhcp snooping trust
 ip arp inspection trust
interface GigabitEthernet0/2
 ip dhcp snooping trust
 ip arp inspection trust
```
> Trust the bundled `Port-channel1` **and** the backup trunk (ACC1 `Gi0/2`) so DHCP/ARP
> still pass when STP moves traffic to the backup path. The port-security example above is
> the ACC2 host port; snooping-trust the DIST-facing uplinks, never the host ports.

### Verify Section 2
```
show vlan brief
show etherchannel summary          ! Po1 = P, both members bundled
show spanning-tree vlan 10         ! Po1 forwarding (root DIST1); ACC1 Gi0/2 backup blocking
show port-security
```
Failover check: shut ACC1's Po1 members → the DIST2 backup trunk (`Gi0/2`) goes
forwarding and VLAN 10 stays up; `no shutdown` restores the bundle and the backup blocks
again.

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
interface range GigabitEthernet1/0, GigabitEthernet1/3
 channel-group 1 mode active
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
!
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
interface GigabitEthernet1/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
!
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

HSRP on the distribution SVIs with an IP SLA + tracked object that fails the active role
over on upstream failure. DIST1 active VLAN10, DIST2 active VLAN20.

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
Mirror on DIST2 (VLAN10 priority 90, VLAN20 priority 110).

### 3.2 IP SLA + track (DIST1)
```
ip sla 1
 icmp-echo 10.0.0.2 source-interface GigabitEthernet0/1
 frequency 5
ip sla schedule 1 life forever start-time now
track 1 ip sla 1 reachability
interface Vlan10
 standby 10 track 1 decrement 30
```
Add a mirrored SLA/track on DIST2 for VLAN20.

### Verify Section 3
```
show standby brief
show track
show ip sla statistics
```
Shut DIST1 Gi0/1 → VLAN10 active moves to DIST2 → recovers on `no shut`.

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

Local AS 65000; ISP1 65001, ISP2 65002. Redistribute EIGRP into BGP (or originate with
`network`) so internal prefixes are advertised.

### 4.1 eBGP edges (CORE1 → ISP1, 203.0.113.0/31)
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
Mirror CORE2↔ISP2 on 203.0.113.2/31 (CORE2 .2, ISP2 .3), AS 65002. On each ISP, advertise
a test prefix (e.g. 8.8.8.0/24).

### 4.2 iBGP across the core
```
router bgp 65000
 neighbor 10.0.0.2 remote-as 65000
 neighbor 10.0.0.2 update-source Loopback0
 address-family ipv4
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 next-hop-self
```
Mirror on CORE2.

### 4.3 Redistribute EIGRP into BGP (optional to advertising specific networks)
```
router bgp 65000
 address-family ipv4
  redistribute eigrp 100
```

### Verify Section 4
```
show ip bgp summary
show ip bgp
show ip route bgp
```

### Full config for this section

_Only this section's commands, per device it touches. Paste into each device in config mode._

**CORE1**
```
hostname CORE1
!
interface GigabitEthernet0/4
 description to ISP1 (eBGP)
 ip address 203.0.113.0 255.255.255.254
 no shutdown
!
router bgp 65000
 bgp router-id 10.0.0.1
 neighbor 203.0.113.1 remote-as 65001
 neighbor 10.0.0.2 remote-as 65000
 neighbor 10.0.0.2 update-source Loopback0
 address-family ipv4
  neighbor 203.0.113.1 activate
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 next-hop-self
  network 10.0.0.0 mask 255.255.0.0
  redistribute eigrp 100
```
**CORE2**
```
hostname CORE2
!
interface GigabitEthernet0/5
 description to ISP2 (eBGP)
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
  redistribute eigrp 100
```
**ISP1**
```
hostname ISP1
!
interface GigabitEthernet0/1
 description to CORE1
 ip address 203.0.113.1 255.255.255.254
 no shutdown
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

## Section 5 — Redistribution (EIGRP ↔ OSPF)

Run OSPF in the branch, make BRANCH-EDGE the EIGRP/OSPF boundary, and do mutual
redistribution with tags to prevent loops. (This is the mirror of the OSPF-core lab.)

### 5.1 OSPF in the branch
On BRANCH-R1 and BRANCH-EDGE (process 1, area 0 at the branch):
```
router ospf 1
 router-id 10.0.0.9
 network 10.2.0.0 0.0.0.1 area 0
 network 10.2.10.0 0.0.0.255 area 0
```
(Use 10.0.0.8 as the router-id on BRANCH-EDGE.)

### 5.2 Extend EIGRP to BRANCH-EDGE
BRANCH-EDGE runs both: EIGRP AS100 toward WAN1 and OSPF toward BRANCH-R1.
```
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  network 10.1.0.14 0.0.0.1
```

### 5.3 Mutual redistribution on BRANCH-EDGE (tagged)
```
route-map EIGRP-TO-OSPF permit 10
 set tag 100
route-map OSPF-TO-EIGRP deny 10
 match tag 100
route-map OSPF-TO-EIGRP permit 20
 set tag 200
router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  topology base
   redistribute ospf 1 metric 100000 100 255 1 1500 route-map OSPF-TO-EIGRP
  exit-af-topology
```
The `deny tag 100` stops EIGRP-origin routes (redistributed into OSPF) from being pushed
back into EIGRP.

### Verify Section 5
```
show ip route eigrp
show ip route ospf
show ip eigrp topology
show route-map
```
Branch LAN appears in the core as an EIGRP external (D EX); no loops or flaps.

### Full config for this section

_Only this section's commands, per device it touches. Paste into each device in config mode._

**BRANCH-EDGE**
```
hostname BRANCH-EDGE
!
interface GigabitEthernet0/1
 description to WAN1 (EIGRP)
 ip address 10.1.0.15 255.255.255.254
 no shutdown
interface GigabitEthernet0/2
 description to BRANCH-R1 (OSPF)
 ip address 10.2.0.0 255.255.255.254
 no shutdown
!
route-map EIGRP-TO-OSPF permit 10
 set tag 100
route-map OSPF-TO-EIGRP deny 10
 match tag 100
route-map OSPF-TO-EIGRP permit 20
 set tag 200
!
router ospf 1
 router-id 10.0.0.8
 network 10.2.0.0 0.0.0.1 area 0
 redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
!
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  network 10.1.0.14 0.0.0.1
  topology base
   redistribute ospf 1 metric 100000 100 255 1 1500 route-map OSPF-TO-EIGRP
  exit-af-topology
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
router ospf 1
 router-id 10.0.0.9
 network 10.2.0.0 0.0.0.1 area 0
 network 10.2.10.0 0.0.0.255 area 0
```


---

## Section 6 — DMVPN overlay (EIGRP over the tunnel)

Phase 1 hub-and-spoke: WAN1 hub, BRANCH-EDGE spoke, over the INET underlay
(100.64.0.0/24). EIGRP runs across the tunnel.

### 6.1 Underlay
```
! WAN1
interface GigabitEthernet0/3
 ip address 100.64.0.7 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 100.64.0.1
```
Equivalent on BRANCH-EDGE (100.64.0.8).

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

### 6.4 EIGRP over the tunnel
```
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  network 172.20.0.0 0.0.0.255
```
`no ip split-horizon eigrp 100` on the hub tunnel is what lets spokes learn each other's
routes — essential for the Phase 3 extension later.

### Verify Section 6
```
show dmvpn
show ip nhrp
show ip eigrp neighbors
ping 172.20.0.1 source 172.20.0.8
```

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
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
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
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  network 172.20.0.0 0.0.0.255
```


---

## Final integration check

- VLAN10 host reaches the services host, the branch LAN, and an ISP test prefix.
- Shut DIST1 upstream → VLAN10 HSRP moves to DIST2 → traffic still flows.
- Shut CORE1–ISP1 → BGP best path shifts to ISP2.
- Shut WAN1–BRANCH-EDGE → branch still reachable via the DMVPN overlay.
- Break one core–dist link → variance/feasible-successor keeps the path up with no reconvergence delay.

---

# CHALLENGE — steps only, no guide

Rebuild from a blank topology. No command hints. Verify each step before moving on.

1. Bring up all loopbacks and transit /31s per the addressing plan.
2. Configure EIGRP AS 100 in **named mode** across CORE1, CORE2, DIST1, DIST2.
3. Set explicit EIGRP router-IDs and make all non-transit interfaces passive.
4. Add SHA-256 authentication on every EIGRP adjacency.
5. Summarize the 192.168.0.0/16 access space toward the core.
6. Configure an EIGRP stub on the services-facing router; confirm it isn't used as transit.
7. Tune variance and maximum-paths so an unequal-cost path is installed; prove it in the routing table.
8. Create VLANs 10/20/99 on the access switches with correct trunks to distribution.
9. Build a real LACP EtherChannel from ACC1 to DIST1 (two links, `Gi0/1`+`Gi0/3`), and configure the single ACC1→DIST2 link as an independent backup trunk; confirm the bundle forms and the backup blocks.
10. Set RSTP so VLAN10 and VLAN20 root on different distribution switches.
11. Harden all access ports: portfast, BPDU guard, port-security, DHCP snooping, DAI.
12. Configure HSRPv2: DIST1 active VLAN10, DIST2 active VLAN20, both with preempt.
13. Add an IP SLA + tracked object on each active router that decrements HSRP on upstream failure; prove failover and recovery.
14. Bring up eBGP CORE1↔ISP1 (65000/65001) and CORE2↔ISP2 (65000/65002).
15. Bring up iBGP CORE1↔CORE2 over loopbacks with next-hop-self.
16. Advertise internal prefixes into BGP (network statements or redistribute EIGRP).
17. Run OSPF area 0 across the branch (BRANCH-EDGE, BRANCH-R1, branch LAN).
18. Mutually redistribute EIGRP and OSPF on BRANCH-EDGE with tags that prevent a redistribution loop.
19. Build a DMVPN Phase 1 tunnel: WAN1 hub, BRANCH-EDGE spoke, over 100.64.0.0/24; disable split-horizon on the hub.
20. Extend EIGRP AS 100 across the tunnel and confirm the neighbor and branch reachability.
21. Prove all failure scenarios in the integration check pass.
22. Bonus: convert the EIGRP core to classic mode and back; note what changes.
23. Bonus: extend DMVPN to Phase 3 and confirm spoke-to-spoke shortcut with a second spoke.
