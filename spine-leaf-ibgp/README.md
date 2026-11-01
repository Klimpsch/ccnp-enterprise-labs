# DC Spine-Leaf with iBGP: Config Guide

Companion to `dc-spine-leaf-ibgp.yaml` (CML 2.10, IOSv).

## Addressing Plan

**BGP AS:** 65000  **OSPF:** process 1, area 0

| Device | Loopback0 |
|---|---|
| SPINE1 | 10.0.0.1/32 |
| SPINE2 | 10.0.0.2/32 |
| LEAF1 | 10.0.0.11/32 |
| LEAF2 | 10.0.0.12/32 |
| LEAF3 | 10.0.0.13/32 |
| LEAF4 | 10.0.0.14/32 |

| Spine interface | Spine IP | Leaf interface | Leaf IP |
|---|---|---|---|
| SPINE1 Gi0/1 | 10.1.1.0/31 | LEAF1 Gi0/1 | 10.1.1.1/31 |
| SPINE1 Gi0/2 | 10.1.1.2/31 | LEAF2 Gi0/1 | 10.1.1.3/31 |
| SPINE1 Gi0/3 | 10.1.1.4/31 | LEAF3 Gi0/1 | 10.1.1.5/31 |
| SPINE1 Gi0/4 | 10.1.1.6/31 | LEAF4 Gi0/1 | 10.1.1.7/31 |
| SPINE2 Gi0/1 | 10.1.2.0/31 | LEAF1 Gi0/2 | 10.1.2.1/31 |
| SPINE2 Gi0/2 | 10.1.2.2/31 | LEAF2 Gi0/2 | 10.1.2.3/31 |
| SPINE2 Gi0/3 | 10.1.2.4/31 | LEAF3 Gi0/2 | 10.1.2.5/31 |
| SPINE2 Gi0/4 | 10.1.2.6/31 | LEAF4 Gi0/2 | 10.1.2.7/31 |

| Leaf | Gi0/3 (gateway) | Host | Host IP (Gi0/1) |
|---|---|---|---|
| LEAF1 | 172.16.1.1/24 | HOST1 | 172.16.1.10/24 |
| LEAF2 | 172.16.2.1/24 | HOST2 | 172.16.2.10/24 |
| LEAF3 | 172.16.3.1/24 | HOST3 | 172.16.3.10/24 |
| LEAF4 | 172.16.4.1/24 | HOST4 | 172.16.4.10/24 |

---

## 1. Interfaces

**SPINE1**
```
enable
configure terminal
no ip domain lookup
ip routing
ip cef
!
interface Loopback0
 description ROUTER-ID / BGP PEERING
 ip address 10.0.0.1 255.255.255.255
!
interface GigabitEthernet0/1
 description FABRIC to LEAF1 Gi0/1
 ip address 10.1.1.0 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/2
 description FABRIC to LEAF2 Gi0/1
 ip address 10.1.1.2 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/3
 description FABRIC to LEAF3 Gi0/1
 ip address 10.1.1.4 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/4
 description FABRIC to LEAF4 Gi0/1
 ip address 10.1.1.6 255.255.255.254
 no shutdown
end
```

**SPINE2**
```
enable
configure terminal
no ip domain lookup
ip routing
ip cef
!
interface Loopback0
 description ROUTER-ID / BGP PEERING
 ip address 10.0.0.2 255.255.255.255
!
interface GigabitEthernet0/1
 description FABRIC to LEAF1 Gi0/2
 ip address 10.1.2.0 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/2
 description FABRIC to LEAF2 Gi0/2
 ip address 10.1.2.2 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/3
 description FABRIC to LEAF3 Gi0/2
 ip address 10.1.2.4 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/4
 description FABRIC to LEAF4 Gi0/2
 ip address 10.1.2.6 255.255.255.254
 no shutdown
end
```

**LEAF1**
```
enable
configure terminal
no ip domain lookup
ip routing
ip cef
!
interface Loopback0
 description ROUTER-ID / BGP PEERING
 ip address 10.0.0.11 255.255.255.255
!
interface GigabitEthernet0/1
 description FABRIC to SPINE1 Gi0/1
 ip address 10.1.1.1 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/2
 description FABRIC to SPINE2 Gi0/1
 ip address 10.1.2.1 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/3
 description SERVER LAN - HOST1
 ip address 172.16.1.1 255.255.255.0
 no shutdown
end
```

**LEAF2**
```
enable
configure terminal
no ip domain lookup
ip routing
ip cef
!
interface Loopback0
 description ROUTER-ID / BGP PEERING
 ip address 10.0.0.12 255.255.255.255
!
interface GigabitEthernet0/1
 description FABRIC to SPINE1 Gi0/2
 ip address 10.1.1.3 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/2
 description FABRIC to SPINE2 Gi0/2
 ip address 10.1.2.3 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/3
 description SERVER LAN - HOST2
 ip address 172.16.2.1 255.255.255.0
 no shutdown
end
```

**LEAF3**
```
enable
configure terminal
no ip domain lookup
ip routing
ip cef
!
interface Loopback0
 description ROUTER-ID / BGP PEERING
 ip address 10.0.0.13 255.255.255.255
!
interface GigabitEthernet0/1
 description FABRIC to SPINE1 Gi0/3
 ip address 10.1.1.5 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/2
 description FABRIC to SPINE2 Gi0/3
 ip address 10.1.2.5 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/3
 description SERVER LAN - HOST3
 ip address 172.16.3.1 255.255.255.0
 no shutdown
end
```

**LEAF4**
```
enable
configure terminal
no ip domain lookup
ip routing
ip cef
!
interface Loopback0
 description ROUTER-ID / BGP PEERING
 ip address 10.0.0.14 255.255.255.255
!
interface GigabitEthernet0/1
 description FABRIC to SPINE1 Gi0/4
 ip address 10.1.1.7 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/2
 description FABRIC to SPINE2 Gi0/4
 ip address 10.1.2.7 255.255.255.254
 no shutdown
!
interface GigabitEthernet0/3
 description SERVER LAN - HOST4
 ip address 172.16.4.1 255.255.255.0
 no shutdown
end
```

---

## 2. OSPF Underlay

**SPINE1**
```
configure terminal
router ospf 1
 router-id 10.0.0.1
 passive-interface default
 no passive-interface GigabitEthernet0/1
 no passive-interface GigabitEthernet0/2
 no passive-interface GigabitEthernet0/3
 no passive-interface GigabitEthernet0/4
!
interface Loopback0
 ip ospf 1 area 0
!
interface range GigabitEthernet0/1 - 4
 ip ospf network point-to-point
 ip ospf 1 area 0
end
```

**SPINE2**
```
configure terminal
router ospf 1
 router-id 10.0.0.2
 passive-interface default
 no passive-interface GigabitEthernet0/1
 no passive-interface GigabitEthernet0/2
 no passive-interface GigabitEthernet0/3
 no passive-interface GigabitEthernet0/4
!
interface Loopback0
 ip ospf 1 area 0
!
interface range GigabitEthernet0/1 - 4
 ip ospf network point-to-point
 ip ospf 1 area 0
end
```

**LEAF1**
```
configure terminal
router ospf 1
 router-id 10.0.0.11
 passive-interface default
 no passive-interface GigabitEthernet0/1
 no passive-interface GigabitEthernet0/2
!
interface Loopback0
 ip ospf 1 area 0
!
interface range GigabitEthernet0/1 - 2
 ip ospf network point-to-point
 ip ospf 1 area 0
end
```

**LEAF2**
```
configure terminal
router ospf 1
 router-id 10.0.0.12
 passive-interface default
 no passive-interface GigabitEthernet0/1
 no passive-interface GigabitEthernet0/2
!
interface Loopback0
 ip ospf 1 area 0
!
interface range GigabitEthernet0/1 - 2
 ip ospf network point-to-point
 ip ospf 1 area 0
end
```

**LEAF3**
```
configure terminal
router ospf 1
 router-id 10.0.0.13
 passive-interface default
 no passive-interface GigabitEthernet0/1
 no passive-interface GigabitEthernet0/2
!
interface Loopback0
 ip ospf 1 area 0
!
interface range GigabitEthernet0/1 - 2
 ip ospf network point-to-point
 ip ospf 1 area 0
end
```

**LEAF4**
```
configure terminal
router ospf 1
 router-id 10.0.0.14
 passive-interface default
 no passive-interface GigabitEthernet0/1
 no passive-interface GigabitEthernet0/2
!
interface Loopback0
 ip ospf 1 area 0
!
interface range GigabitEthernet0/1 - 2
 ip ospf network point-to-point
 ip ospf 1 area 0
end
```

---

## 3. iBGP Overlay

**SPINE1**
```
configure terminal
router bgp 65000
 bgp router-id 10.0.0.1
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor LEAFS peer-group
 neighbor LEAFS remote-as 65000
 neighbor LEAFS update-source Loopback0
 neighbor 10.0.0.11 peer-group LEAFS
 neighbor 10.0.0.12 peer-group LEAFS
 neighbor 10.0.0.13 peer-group LEAFS
 neighbor 10.0.0.14 peer-group LEAFS
 !
 address-family ipv4
  neighbor LEAFS route-reflector-client
  neighbor 10.0.0.11 activate
  neighbor 10.0.0.12 activate
  neighbor 10.0.0.13 activate
  neighbor 10.0.0.14 activate
 exit-address-family
end
```

**SPINE2**
```
configure terminal
router bgp 65000
 bgp router-id 10.0.0.2
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor LEAFS peer-group
 neighbor LEAFS remote-as 65000
 neighbor LEAFS update-source Loopback0
 neighbor 10.0.0.11 peer-group LEAFS
 neighbor 10.0.0.12 peer-group LEAFS
 neighbor 10.0.0.13 peer-group LEAFS
 neighbor 10.0.0.14 peer-group LEAFS
 !
 address-family ipv4
  neighbor LEAFS route-reflector-client
  neighbor 10.0.0.11 activate
  neighbor 10.0.0.12 activate
  neighbor 10.0.0.13 activate
  neighbor 10.0.0.14 activate
 exit-address-family
end
```

**LEAF1**
```
configure terminal
router bgp 65000
 bgp router-id 10.0.0.11
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor SPINES peer-group
 neighbor SPINES remote-as 65000
 neighbor SPINES update-source Loopback0
 neighbor 10.0.0.1 peer-group SPINES
 neighbor 10.0.0.2 peer-group SPINES
 !
 address-family ipv4
  network 172.16.1.0 mask 255.255.255.0
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.2 activate
 exit-address-family
end
```

**LEAF2**
```
configure terminal
router bgp 65000
 bgp router-id 10.0.0.12
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor SPINES peer-group
 neighbor SPINES remote-as 65000
 neighbor SPINES update-source Loopback0
 neighbor 10.0.0.1 peer-group SPINES
 neighbor 10.0.0.2 peer-group SPINES
 !
 address-family ipv4
  network 172.16.2.0 mask 255.255.255.0
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.2 activate
 exit-address-family
end
```

**LEAF3**
```
configure terminal
router bgp 65000
 bgp router-id 10.0.0.13
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor SPINES peer-group
 neighbor SPINES remote-as 65000
 neighbor SPINES update-source Loopback0
 neighbor 10.0.0.1 peer-group SPINES
 neighbor 10.0.0.2 peer-group SPINES
 !
 address-family ipv4
  network 172.16.3.0 mask 255.255.255.0
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.2 activate
 exit-address-family
end
```

**LEAF4**
```
configure terminal
router bgp 65000
 bgp router-id 10.0.0.14
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor SPINES peer-group
 neighbor SPINES remote-as 65000
 neighbor SPINES update-source Loopback0
 neighbor 10.0.0.1 peer-group SPINES
 neighbor 10.0.0.2 peer-group SPINES
 !
 address-family ipv4
  network 172.16.4.0 mask 255.255.255.0
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.2 activate
 exit-address-family
end
```

---

## 4. Hosts

**HOST1**
```
enable
configure terminal
no ip domain lookup
interface GigabitEthernet0/1
 description SERVER NIC to LEAF1
 ip address 172.16.1.10 255.255.255.0
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 172.16.1.1 name LEAF-GATEWAY
end
```

**HOST2**
```
enable
configure terminal
no ip domain lookup
interface GigabitEthernet0/1
 description SERVER NIC to LEAF2
 ip address 172.16.2.10 255.255.255.0
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 172.16.2.1 name LEAF-GATEWAY
end
```

**HOST3**
```
enable
configure terminal
no ip domain lookup
interface GigabitEthernet0/1
 description SERVER NIC to LEAF3
 ip address 172.16.3.10 255.255.255.0
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 172.16.3.1 name LEAF-GATEWAY
end
```

**HOST4**
```
enable
configure terminal
no ip domain lookup
interface GigabitEthernet0/1
 description SERVER NIC to LEAF4
 ip address 172.16.4.10 255.255.255.0
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 172.16.4.1 name LEAF-GATEWAY
end
```
