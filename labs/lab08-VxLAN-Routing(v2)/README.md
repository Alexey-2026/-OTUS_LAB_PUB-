# Настройка VxLAN Routing

### Цель работы
Реализовать передачу суммарных префиксов через EVPN route-type 5.

<img src="https://github.com/user-attachments/assets/f443ca68-4994-4d7d-9c21-e7a2690e739c" width="500" style="max-width: 100%; height: auto;">




## План адресации

### Underlay сеть (линки /31)

| Линк | Spine интерфейс | IP Spine | Leaf интерфейс | IP Leaf |
|------|----------------|----------|----------------|---------|
| Spine1-Leaf1 | Eth1/1 | 172.16.10.1 | Eth1/6 | 172.16.10.0 |
| Spine1-Leaf2 | Eth1/2 | 172.16.10.3 | Eth1/6 | 172.16.10.2 |
| Spine1-Leaf3 | Eth1/3 | 172.16.10.5 | Eth1/6 | 172.16.10.4 |
| Spine1-Leaf4 | Eth1/4 | 172.16.10.15 | Eth1/6 | 172.16.10.14 |
| Spine2-Leaf1 | Eth1/1 | 172.16.10.7 | Eth1/7 | 172.16.10.6 |
| Spine2-Leaf2 | Eth1/2 | 172.16.10.9 | Eth1/7 | 172.16.10.8 |
| Spine2-Leaf3 | Eth1/3 | 172.16.10.11 | Eth1/7 | 172.16.10.10 |
| Spine2-Leaf4 | Eth1/4 | 172.16.10.17 | Eth1/7 | 172.16.10.16 |

### Соединения с внешним роутером

| Линк | Client3 интерфейс | IP Client3 | Leaf интерфейс | IP Leaf |
|------|----------------|----------|----------------|---------|
| Client3-Leaf3| Gi1.10 | 192.168.10.0 | Eth1/1.10 | 192.168.10.1 |
| Client3-Leaf3| Gi1.20 | 192.168.20.0 | Eth1/1.20 | 192.168.20.1 |
| Client3-Leaf4| Gi2.10 | 192.168.10.2 | Eth1/1.10 | 192.168.10.3 |
| Client3-Leaf4| Gi2.20 | 192.168.20.2 | Eth1/1.20 | 192.168.20.3 |

### Loopback адреса

| Устройство | Loopback0 IP |
|------------|--------------|
| Spine-1 | 172.16.0.111/32 |
| Spine-2 | 172.16.0.112/32 |
| Leaf-1 | 172.16.0.11/32 |
| Leaf-2 | 172.16.0.12/32 |
| Leaf-3 | 172.16.0.13/32 |
| Leaf-4 | 172.16.0.14/32 |
| Client3(BorderRouter)|30.30.30.30/32|

### Серверные подсети

| Leaf | VLAN | Подсеть | Шлюз |
|------|------|---------|------|
| Leaf-1 | 10 | 10.0.0.0/24 | 10.0.0.254 |
| Leaf-1 | 20 | 20.0.0.0/24 | 20.0.0.254 |
| Leaf-2 | 10 | 10.0.0.0/24 | 10.0.0.254 |
| Leaf-2 | 20 | 20.0.0.0/24 | 20.0.0.254 |

---

## Конфигурация оборудования (VXLAN Routing)

### Spine-1
```
hostname Spine1
!
nv overlay evpn
feature bgp
feature bfd
feature nv overlay
!
no ip domain-lookup
!
bfd interval 100 min_rx 100 multiplier 5
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
!
interface Ethernet1/1
  description Link_To_Leaf1
  no ip redirects
  ip address 172.16.10.1/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/2
  description Link_To_Leaf2
  no ip redirects
  ip address 172.16.10.3/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/3
  description Link_To_Leaf3
  no ip redirects
  ip address 172.16.10.5/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/4
  description Link_To_Leaf4
  no ip redirects
  ip address 172.16.10.15/31
  no ipv6 redirects
  no shutdown
!
interface loopback0
  description ----RID
  ip address 172.16.0.111/32
!
router bgp 65501
  router-id 172.16.0.111
  address-family ipv4 unicast
    redistribute direct route-map RM_RED_FOR_BGP
    maximum-paths 4
    maximum-paths ibgp 4
  template peer LEAFs
    bfd
    remote-as 65501
    password 3 9125d59c18a9b015
    timers 3 9
    address-family ipv4 unicast
      send-community
      send-community extended
      route-reflector-client
      next-hop-self all
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  neighbor 172.16.10.0/24
    inherit peer LEAFs
    description DINAMIC_POOL
```
### Spine-2
```
hostname Spine2
!
nv overlay evpn
feature bgp
feature bfd
feature nv overlay
!
no ip domain-lookup
!
bfd interval 100 min_rx 100 multiplier 5
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
!
interface Ethernet1/1
  description Link_To_Leaf1
  no ip redirects
  ip address 172.16.10.7/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/2
  description Link_To_Leaf2
  no ip redirects
  ip address 172.16.10.9/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/3
  description Link_To_Leaf3
  no ip redirects
  ip address 172.16.10.11/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/4
  description Link_To_Leaf4
  no ip redirects
  ip address 172.16.10.17/31
  no ipv6 redirects
  no shutdown
!
interface loopback0
  ip address 172.16.0.112/32
!
router bgp 65501
  router-id 172.16.0.112
  address-family ipv4 unicast
    redistribute direct route-map RM_RED_FOR_BGP
    maximum-paths 4
    maximum-paths ibgp 4
  template peer LEAFs
    bfd
    remote-as 65501
    password 3 9125d59c18a9b015
    timers 3 9
    address-family ipv4 unicast
      send-community
      send-community extended
      route-reflector-client
      next-hop-self all
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  neighbor 172.16.10.0/24
    inherit peer LEAFs
    description DINAMIC_POOL
```

### Leaf-1
```

hostname Leaf1
vdc Leaf1 id 1

cfs eth distribute
nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature vpc
feature bfd
feature nv overlay

no password strength-check
username admin password 5 $5$DFCNEA$y8LuOtfEyvO/3Lxzqpkto4mzdsgaxwqGqYptvD17r3B 
 role network-admin
no ip domain-lookup
copp profile strict
bfd interval 100 min_rx 100 multiplier 5

fabric forwarding anycast-gateway-mac 0001.0001.0001
vlan 1,10,20,77-78,3900
vlan 10
  name SEVERS_VLAN10
  vn-segment 10000
vlan 20
  name SEVERS_VLAN20
  vn-segment 20000
vlan 77
  name L3VNI_VRF_A
  vn-segment 770000
vlan 78
  name L3VNI_VRF_B
  vn-segment 780000
vlan 3900
  name BACKUP_VLAN_ROUTING

route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
route-map SET_LP_FROM_LEAF2 permit 10
  set local-preference 90
route-map SET_NH_FOR_LEAF1 permit 10
  set ip next-hop 172.16.39.0    
route-map SET_NH_FOR_SPINE permit 10
  set ip next-hop peer-address

vrf context VRF_A
  vni 770000
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
vrf context VRF_B
  vni 780000
  rd auto
  address-family ipv4 unicast
    route-target import 7878:780000
    route-target import 7878:780000 evpn
    route-target export 7878:780000
    route-target export 7878:780000 evpn
vrf context management

hardware access-list tcam region racl 1024
hardware access-list tcam region span 0
hardware access-list tcam region rp-ipv6-qos 0
hardware access-list tcam region arp-ether 256 double-wide

vpc domain 10
  peer-switch
  peer-keepalive destination 192.168.1.1 source 192.168.1.0
  peer-gateway
  layer3 peer-router
  auto-recovery
  fast-convergence
  ip arp synchronize

interface Vlan1
  no ip redirects
  no ipv6 redirects

interface Vlan10
  description ----SERVERS_VLAN10----
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip address 10.0.0.254/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway

interface Vlan20
  description ----SERVERS_VLAN20----
  no shutdown
  vrf member VRF_B
  no ip redirects
  ip address 20.0.0.254/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway

interface Vlan77
  description ####L3VNI_770000####
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip forward
  no ipv6 redirects

interface Vlan78
  description ####L3VNI_780000####
  no shutdown
  vrf member VRF_B
  no ip redirects
  ip forward
  no ipv6 redirects

interface Vlan3900
  description VPC_L3_Peering_VXLAN
  no shutdown
  mtu 9216
  no ip redirects
  ip address 172.16.39.0/31
  no ipv6 redirects

interface port-channel1
  description ----VPC_PEER_TO_LEAF12
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77-78,3900
  spanning-tree port type network
  vpc peer-link

interface port-channel10
  description ----VPC_CLIENT1----
  switchport
  switchport access vlan 10
  vpc 11

interface port-channel20
  description ----VPC_CLIENT2----
  switchport
  switchport access vlan 20
  vpc 12

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  global ingress-replication protocol bgp
  member vni 10000
    suppress-arp
  member vni 20000
    suppress-arp
  member vni 770000 associate-vrf
  member vni 780000 associate-vrf

interface Ethernet1/1
  description ----TO_CLIENT_VLAN10----
  switchport
  switchport access vlan 10
  spanning-tree bpduguard enable
  spanning-tree bpdufilter enable
  channel-group 10 mode active
  no shutdown

interface Ethernet1/2
  description ----TO_CLIENT_VLAN20----
  switchport
  switchport access vlan 20
  spanning-tree bpduguard enable
  spanning-tree bpdufilter enable
  channel-group 20 mode active
  no shutdown

interface Ethernet1/4
  description ----VPC_PEERLINK----
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77-78,3900
  channel-group 1 mode active
  no shutdown

interface Ethernet1/5
  description ----VPC_PEERLINK----
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77-78,3900
  channel-group 1 mode active
  no shutdown

interface Ethernet1/6
  description Link_To_Spine1
  no ip redirects
  ip address 172.16.10.0/31
  no ipv6 redirects
  no shutdown

interface Ethernet1/7
  description Link_To_Spine2
  no ip redirects
  ip address 172.16.10.6/31
  no ipv6 redirects
  no shutdown

interface mgmt0
  vrf member management
  ip address 192.168.1.0/31

interface loopback0
  ip address 172.16.0.11/32
  ip address 172.16.0.100/32 secondary
 
router bgp 65501
  router-id 172.16.0.11
  address-family ipv4 unicast
    redistribute direct route-map RM_RED_FOR_BGP
    maximum-paths 2
    maximum-paths ibgp 2
  template peer SPINEs
    bfd
    remote-as 65501
    password 3 9125d59c18a9b015
    timers 3 9
    address-family ipv4 unicast
      send-community
      send-community extended
      route-map SET_NH_FOR_SPINE out
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 172.16.10.1
    inherit peer SPINEs
    description To_Spine1
  neighbor 172.16.10.7
    inherit peer SPINEs
    description To_Spine2
  neighbor 172.16.39.1
    remote-as 65501
    update-source Vlan3900
    address-family ipv4 unicast
      send-community
      send-community extended
      route-reflector-client
      route-map SET_LP_FROM_LEAF2 in
      route-map SET_NH_FOR_LEAF1 out
      next-hop-self
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  vrf VRF_A
    address-family ipv4 unicast
      network 10.0.0.0/24
      maximum-paths mixed 2
  vrf VRF_B
    address-family ipv4 unicast
      network 20.0.0.0/24
      maximum-paths mixed 2
evpn
  vni 10000 l2
    rd auto
    route-target import 65501:10000
    route-target export 65501:10000
  vni 20000 l2
    rd auto
    route-target import 65501:20000
    route-target export 65501:20000
```
### Leaf-2
```
hostname Leaf2
vdc Leaf2 id 1

cfs eth distribute
nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature vpc
feature bfd
feature nv overlay

no password strength-check
username admin password 5 $5$DJDBOJ$siVll3Ec96Q3dCFJa.DwglQLF8JD.OOfzNDYUkHn8f2 
 role network-admin
no ip domain-lookup
copp profile strict
bfd interval 100 min_rx 100 multiplier 5

fabric forwarding anycast-gateway-mac 0001.0001.0001
vlan 1,10,20,77-78,3900
vlan 10
  name FOR_CLIENT
  vn-segment 10000
vlan 20
  name SERVERS_VLAN20
  vn-segment 20000
vlan 77
  name L3VNI_VRF_A
  vn-segment 770000
vlan 78
  name L3VNI_VRF_B
  vn-segment 780000
vlan 3900
  name BACKUP_VLAN_ROUTING

route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
route-map SET_LP_FROM_LEAF1 permit 10
  set local-preference 90
route-map SET_NH_FOR_LEAF1 permit 10
  set ip next-hop 172.16.39.1    
route-map SET_NH_FOR_SPINE permit 10
  set ip next-hop peer-address

vrf context VRF_A
  vni 770000
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
vrf context VRF_B
  vni 780000
  rd auto
  address-family ipv4 unicast
    route-target import 7878:780000
    route-target import 7878:780000 evpn
    route-target export 7878:780000
    route-target export 7878:780000 evpn
vrf context management

hardware access-list tcam region racl 1024
hardware access-list tcam region span 0
hardware access-list tcam region rp-ipv6-qos 0
hardware access-list tcam region arp-ether 256 double-wide
vpc domain 10
  peer-switch
  peer-keepalive destination 192.168.1.0 source 192.168.1.1
  peer-gateway
  layer3 peer-router
  auto-recovery
  fast-convergence
  ip arp synchronize


interface Vlan1
  no ip redirects
  no ipv6 redirects

interface Vlan10
  description ----SERVERS_VLAN10----
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip address 10.0.0.254/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway

interface Vlan20
  description ----SERVERS_VLAN20----
  no shutdown
  vrf member VRF_B
  no ip redirects
  ip address 20.0.0.254/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway

interface Vlan77
  description ####L3VNI_770000####
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip forward
  no ipv6 redirects

interface Vlan78
  description ####L3VNI_780000####
  no shutdown
  vrf member VRF_B
  no ip redirects
  ip forward
  no ipv6 redirects

interface Vlan3900
  description VPC_L3_Peering_VXLAN
  no shutdown
  mtu 9216
  no ip redirects
  ip address 172.16.39.1/31
  no ipv6 redirects

interface port-channel1
  description ----VPC_PEER_TO_LEAF11----
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77-78,3900
  spanning-tree port type network
  vpc peer-link

interface port-channel10
  description ----VPC_EP1----
  switchport
  switchport access vlan 10
  vpc 11

interface port-channel20
  description ----VPC_EP2----
  switchport
  switchport access vlan 20
  vpc 12

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  global ingress-replication protocol bgp
  member vni 10000
    suppress-arp
  member vni 20000
    suppress-arp
  member vni 770000 associate-vrf
  member vni 780000 associate-vrf

interface Ethernet1/1
  description ----TO_CLIENT_VLAN10----~
  switchport
  switchport access vlan 10
  spanning-tree bpduguard enable
  spanning-tree bpdufilter enable
  channel-group 10 mode active
  no shutdown

interface Ethernet1/2
  description ----TO_CLIENT_VLAN20----
  switchport
  switchport access vlan 20
  spanning-tree bpduguard enable
  spanning-tree bpdufilter enable
  channel-group 20 mode active
  no shutdown

interface Ethernet1/3
  no shutdown

interface Ethernet1/4
  description ----VPC_PEERLINK----
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77-78,3900
  channel-group 1 mode active
  no shutdown

interface Ethernet1/5
  description ----VPC_PEERLINK----
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77-78,3900
  channel-group 1 mode active
  no shutdown

interface Ethernet1/6
  description Link_To_Spine1
  no ip redirects
  ip address 172.16.10.2/31
  no ipv6 redirects
  no shutdown

interface Ethernet1/7
  description Link_To_Spine2
  no ip redirects
  ip address 172.16.10.8/31
  no ipv6 redirects
  no shutdown

interface mgmt0
  vrf member management
  ip address 192.168.1.1/31

interface loopback0
  ip address 172.16.0.12/32
  ip address 172.16.0.100/32 secondary
icam monitor scale

line console
line vty
router bgp 65501
  router-id 172.16.0.12
  address-family ipv4 unicast
    redistribute direct route-map RM_RED_FOR_BGP
    maximum-paths 2
    maximum-paths ibgp 2
  template peer SPINEs
    bfd
    remote-as 65501
    password 3 9125d59c18a9b015
    timers 3 9
    address-family ipv4 unicast
      send-community
      send-community extended
      route-map SET_NH_FOR_SPINE out
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 172.16.10.3
    inherit peer SPINEs
    description ---To_Spine1---
  neighbor 172.16.10.9
    inherit peer SPINEs
    description ---To_Spine2---
  neighbor 172.16.39.0
    remote-as 65501
    update-source Vlan3900
    address-family ipv4 unicast
      send-community
      send-community extended
      route-reflector-client
      route-map SET_LP_FROM_LEAF1 in
      route-map SET_NH_FOR_LEAF1 out
      next-hop-self
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  vrf VRF_A
    address-family ipv4 unicast
      maximum-paths mixed 2
  vrf VRF_B
    address-family ipv4 unicast
      maximum-paths mixed 2
evpn
  vni 10000 l2
    rd auto
    route-target import 65501:10000
    route-target export 65501:10000
  vni 20000 l2
    rd auto
    route-target import 65501:20000
    route-target export 65501:20000
```
### Leaf-3
```
hostname Leaf3

nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature bfd
feature nv overlay

bfd interval 100 min_rx 100 multiplier 5

fabric forwarding anycast-gateway-mac 0001.0001.0001

vlan 1,10,20,77-78
vlan 10
  name SERVERS_VLAN10
  vn-segment 10000
vlan 20
  name SERVERS_VLAN20
  vn-segment 20000
vlan 77
  name L3VNI_VRF_A
  vn-segment 770000
vlan 78
  name L3VNI_VRF_B
  vn-segment 780000

ip prefix-list PL_ALLOW_ONLY_24 seq 10 permit 10.0.0.0/24 
ip prefix-list PL_ALLOW_ONLY_24 seq 15 permit 20.0.0.0/24 
ip prefix-list PL_ALLOW_ONLY_24 seq 20 deny 0.0.0.0/0 le 32 
route-map RM_ALLOW_ONLY_24 permit 10
  match ip address prefix-list PL_ALLOW_ONLY_24 
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 

vrf context VRF_A
  vni 770000
  ip route 30.30.30.30/32 192.168.10.0
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
vrf context VRF_B
  vni 780000
  ip route 30.30.30.30/32 192.168.20.0
  rd auto
  address-family ipv4 unicast
    route-target import 7878:780000
    route-target import 7878:780000 evpn
    route-target export 7878:780000
    route-target export 7878:780000 evpn

interface Vlan77
  description ----L3VNI_770000----
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip forward
  no ipv6 redirects

interface Vlan78
  description ----L3VNI_780000----
  no shutdown
  vrf member VRF_B
  no ip redirects
  ip forward
  no ipv6 redirects

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  global ingress-replication protocol bgp
  member vni 770000 associate-vrf
  member vni 780000 associate-vrf

interface Ethernet1/1
  description -----TO_CLIENT3-----
  no switchport
  no shutdown

interface Ethernet1/1.10
  description ---VRF_A_PEERING---
  encapsulation dot1q 10
  vrf member VRF_A
  ip address 192.168.10.1/31
  no shutdown

interface Ethernet1/1.20
  description ---VRF_B_PEERING---
  encapsulation dot1q 20
  vrf member VRF_B
  ip address 192.168.20.1/31
  no shutdown

interface Ethernet1/6
  description ----To_Spine1----
  no switchport
  evpn multihoming core-tracking
  no ip redirects
  ip address 172.16.10.4/31
  no ipv6 redirects
  no shutdown

interface Ethernet1/7
  description ----To_Spine2----
  no switchport
  evpn multihoming core-tracking
  no ip redirects
  ip address 172.16.10.10/31
  no ipv6 redirects
  no shutdown

interface loopback0
  ip address 172.16.0.13/32
icam monitor scale

router bgp 65501
  router-id 172.16.0.13
  address-family ipv4 unicast
    redistribute direct route-map RM_RED_FOR_BGP
    maximum-paths 2
    maximum-paths ibgp 2
  address-family l2vpn evpn
  template peer SPINEs
    bfd
    remote-as 65501
    password 3 9125d59c18a9b015
    timers 3 9
    address-family ipv4 unicast
      send-community
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 172.16.10.5
    inherit peer SPINEs
    description ----To_Spine1----
  neighbor 172.16.10.11
    inherit peer SPINEs
    description ----To_Spine2----
  vrf VRF_A
    router-id 172.16.0.13
    address-family ipv4 unicast
      network 10.0.0.0/24
      maximum-paths 2
    neighbor 30.30.30.30
      remote-as 65505
      description ---EBGP_TO_CLIENT3_VRF_A---
      password 3 9125d59c18a9b015
      update-source Ethernet1/1.10
      ebgp-multihop 2
      address-family ipv4 unicast
        send-community
        send-community extended
        route-map RM_ALLOW_ONLY_24 out
  vrf VRF_B
    router-id 172.16.0.13
    address-family ipv4 unicast
      network 20.0.0.0/24
      maximum-paths 2
    neighbor 30.30.30.30
      remote-as 65505
      description ---EBGP_TO_CLIENT3_VRF_B---
      password 3 9125d59c18a9b015
      update-source Ethernet1/1.20
      ebgp-multihop 2
      address-family ipv4 unicast
        send-community
        send-community extended
        route-map RM_ALLOW_ONLY_24 out
```
### Leaf-4
```
hostname Leaf4

nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature bfd
feature nv overlay

bfd interval 100 min_rx 100 multiplier 5

fabric forwarding anycast-gateway-mac 0001.0001.0001
vlan 1,10,20,77-78
vlan 10
  name SERVERS_VLAN10
  vn-segment 10000
vlan 20
  name SERVERS_VLAN20
  vn-segment 20000
vlan 77
  name L3VNI_VRF_A
  vn-segment 770000
vlan 78
  name L3VNI_VRF_B
  vn-segment 780000

ip prefix-list PL_ALLOW_ONLY_24 seq 10 permit 10.0.0.0/24 
ip prefix-list PL_ALLOW_ONLY_24 seq 15 permit 20.0.0.0/24 
ip prefix-list PL_ALLOW_ONLY_24 seq 20 deny 0.0.0.0/0 le 32 
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
route-map RM_SEND_ONLY_24 permit 10
  match ip address prefix-list PL_ALLOW_ONLY_24

vrf context VRF_A
  vni 770000
  ip route 30.30.30.30/32 192.168.10.2/31
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
vrf context VRF_B
  vni 780000
  ip route 30.30.30.30/32 192.168.20.2
  rd auto
  address-family ipv4 unicast
    route-target import 7878:780000
    route-target import 7878:780000 evpn
    route-target export 7878:780000
    route-target export 7878:780000 evpn

interface Vlan77
  description ----L3VNI_770000----
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip forward
  no ipv6 redirects

interface Vlan78
  description ----L3VNI_780000----
  no shutdown
  vrf member VRF_B
  no ip redirects
  ip forward
  no ipv6 redirects

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  global ingress-replication protocol bgp
  member vni 770000 associate-vrf
  member vni 780000 associate-vrf

interface Ethernet1/1
  no switchport
  no ip redirects
  no ipv6 redirects
  no shutdown

interface Ethernet1/1.10
  encapsulation dot1q 10
  vrf member VRF_A
  ip address 192.168.10.3/31
  no shutdown

interface Ethernet1/1.20
  encapsulation dot1q 20
  vrf member VRF_B
  ip address 192.168.20.3/31
  no shutdown

interface Ethernet1/6
  description ----To_Spine1----
  no switchport
  evpn multihoming core-tracking
  no ip redirects
  ip address 172.16.10.14/31
  no ipv6 redirects
  no shutdown

interface Ethernet1/7
  description ----To_Spine2----
  no switchport
  evpn multihoming core-tracking
  no ip redirects
  ip address 172.16.10.16/31
  no ipv6 redirects
  no shutdown

interface loopback0
  ip address 172.16.0.14/32

router bgp 65501
  router-id 172.16.0.14
  address-family ipv4 unicast
    redistribute direct route-map RM_RED_FOR_BGP
    maximum-paths 2
    maximum-paths ibgp 2
  template peer SPINEs
    bfd
    remote-as 65501
    password 3 9125d59c18a9b015
    timers 3 9
    address-family ipv4 unicast
      send-community
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 172.16.10.15
    inherit peer SPINEs
    description ----To_Spine1----
  neighbor 172.16.10.17
    inherit peer SPINEs
    description ----To_Spine2----
  vrf VRF_A
    router-id 172.16.0.14
    address-family ipv4 unicast
      network 10.0.0.0/24
      maximum-paths 2
    neighbor 30.30.30.30
      remote-as 65505
      description ----eBGP_to_Client3_Lo----
      password 3 9125d59c18a9b015
      update-source Ethernet1/1.10
      ebgp-multihop 2
      address-family ipv4 unicast
        send-community
        send-community extended
        route-map RM_SEND_ONLY_24 out
  vrf VRF_B
    router-id 172.16.0.14
    address-family ipv4 unicast
      network 20.0.0.0/24
      maximum-paths 2
    neighbor 30.30.30.30
      remote-as 65505
      description ----eBGP_to_Client3_Lo0----
      password 3 9125d59c18a9b015
      update-source Ethernet1/1.20
      ebgp-multihop 2
      address-family ipv4 unicast
        send-community
        send-community extended
        route-map RM_SEND_ONLY_24 out
```
### Client3(BorderRouter/Firewall)
```
hostname Client3

interface Loopback0
 ip address 30.30.30.30 255.255.255.255
!
interface GigabitEthernet1
 no ip address
 negotiation auto
!
interface GigabitEthernet1.10
 encapsulation dot1Q 10
 ip address 192.168.10.0 255.255.255.254
!         
interface GigabitEthernet1.20
 encapsulation dot1Q 20
 ip address 192.168.20.0 255.255.255.254
!
interface GigabitEthernet2
 no ip address
 negotiation auto
!
interface GigabitEthernet2.10
 encapsulation dot1Q 10
 ip address 192.168.10.2 255.255.255.254
!
interface GigabitEthernet2.20
 encapsulation dot1Q 20
 ip address 192.168.20.2 255.255.255.254
!
router bgp 65505
 bgp router-id 30.30.30.30
 bgp log-neighbor-changes
 timers bgp 1 3
 neighbor 192.168.10.1 remote-as 65501
 neighbor 192.168.10.1 password 7 110A1016141D
 neighbor 192.168.10.3 remote-as 65501
 neighbor 192.168.10.3 password 7 094F471A1A0A
 neighbor 192.168.20.1 remote-as 65501
 neighbor 192.168.20.1 password 7 110A1016141D
 neighbor 192.168.20.3 remote-as 65501
 neighbor 192.168.20.3 password 7 110A1016141D
 !
 address-family ipv4
  neighbor 192.168.10.1 activate
  neighbor 192.168.10.1 default-originate
  neighbor 192.168.10.3 activate
  neighbor 192.168.10.3 default-originate
  neighbor 192.168.20.1 activate
  neighbor 192.168.20.1 default-originate
  neighbor 192.168.20.3 activate
  neighbor 192.168.20.3 default-originate
  maximum-paths 2
 exit-address-family
!
route-map RM_FOR_BGP_L0 permit 10
 match interface Loopback0
```
---

## Проверка 
```
Leaf1# sh ip route vrf vrF_A
IP Route Table for VRF "VRF_A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 2/0, all-best (0xac10000b)
    *via 172.16.0.13%default, [200/0], 01:46:19, bgp-65501, internal, tag 65505,
 segid: 770000 tunnelid: 0xac10000d encap: VXLAN
 
    *via 172.16.0.14%default, [200/0], 01:46:17, bgp-65501, internal, tag 65505,
 segid: 770000 tunnelid: 0xac10000e encap: VXLAN
 
10.0.0.0/24, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 1w4d, direct
10.0.0.11/32, ubest/mbest: 1/0, attached
    *via 10.0.0.11, Vlan10, [190/0], 4d03h, hmm
10.0.0.254/32, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 1w4d, local

Leaf1# sh ip route vrf vrF_B
IP Route Table for VRF "VRF_B"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 2/0, all-best (0xac10000b)
    *via 172.16.0.13%default, [200/0], 02:01:15, bgp-65501, internal, tag 65505,
 segid: 780000 tunnelid: 0xac10000d encap: VXLAN
 
    *via 172.16.0.14%default, [200/0], 01:46:22, bgp-65501, internal, tag 65505,
 segid: 780000 tunnelid: 0xac10000e encap: VXLAN
 
20.0.0.0/24, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 6d23h, direct
20.0.0.12/32, ubest/mbest: 1/0, attached
    *via 20.0.0.12, Vlan20, [190/0], 6d23h, hmm
20.0.0.254/32, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 6d23h, local

Leaf1# sh bgp l2vpn evpn 
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 5885, Local Router ID is 172.16.0.11
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100      32768 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100      32768 i
*>l[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100      32768 i

Route Distinguisher: 172.16.0.11:32787    (L2VNI 20000)
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100      32768 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100      32768 i
*>l[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100      32768 i

Route Distinguisher: 172.16.0.13:3
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 65505 i
* i                   172.16.0.13                       100          0 65505 i
*>i                   172.16.0.13                       100          0 65505 i

Route Distinguisher: 172.16.0.13:4
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 65505 i
* i                   172.16.0.13                       100          0 65505 i
*>i                   172.16.0.13                       100          0 65505 i

Route Distinguisher: 172.16.0.14:3
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 65505 i
* i                   172.16.0.14                       100          0 65505 i
*>i                   172.16.0.14                       100          0 65505 i

Route Distinguisher: 172.16.0.14:4
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 65505 i
* i                   172.16.0.14                       100          0 65505 i
*>i                   172.16.0.14                       100          0 65505 i

Route Distinguisher: 172.16.0.11:3    (L3VNI 770000)
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 65505 i
* i                   172.16.0.14                       100          0 65505 i
*>l[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.16.0.100                      100      32768 i

Route Distinguisher: 172.16.0.11:4    (L3VNI 780000)
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 65505 i
*>i                   172.16.0.13                       100          0 65505 i
*>l[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.16.0.100                      100      32768 i

Leaf1# sh bgp l2vpn evpn rou
route-map    route-type   
Leaf1# sh bgp l2vpn evpn route-type 5
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 5879
Paths: (3 available, best #3)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Path type: internal, path is valid, not best reason: RR Cluster Length, no lab
eled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.39.1 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Received path-id 1
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.12 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_A L3-770000
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.39.1    

Route Distinguisher: 172.16.0.13:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 5878
Paths: (3 available, best #3)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Path type: internal, path is valid, not best reason: RR Cluster Length, no lab
eled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.39.1 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Received path-id 1
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.12 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_B L3-780000
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.39.1    

Route Distinguisher: 172.16.0.14:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 5869
Paths: (3 available, best #3)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Path type: internal, path is valid, not best reason: RR Cluster Length, no lab
eled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.39.1 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Received path-id 1
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.12 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_A L3-770000
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.39.1    

Route Distinguisher: 172.16.0.14:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 5883
Paths: (3 available, best #3)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Path type: internal, path is valid, not best reason: RR Cluster Length, no lab
eled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.39.1 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Received path-id 1
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.12 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_B L3-780000
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.39.1    

Route Distinguisher: 172.16.0.11:3    (L3VNI 770000)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 5874
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Router Id, no labeled nex
thop
             Imported from 172.16.0.14:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [5]:[0]:[0]:[24]:[10.0.0.0]/224, version 1696
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.1        172.16.10.7        172.16.39.1    

Route Distinguisher: 172.16.0.11:4    (L3VNI 780000)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 5881
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nex
thop
             Imported from 172.16.0.14:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [5]:[0]:[0]:[24]:[20.0.0.0]/224, version 1697
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.1        172.16.10.7        172.16.39.1    
```
===============================================================================================
```
Leaf3# sh bgp l2vpn evpn 
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 8818, Local Router ID is 172.16.0.13
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:3
*>i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.11:4
* i[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.11:32777
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.11:32787
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.12:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.12:32787
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.14:3
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 65505 i
*>i                   172.16.0.14                       100          0 65505 i

Route Distinguisher: 172.16.0.14:4
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 65505 i
*>i                   172.16.0.14                       100          0 65505 i

Route Distinguisher: 172.16.0.13:3    (L3VNI 770000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 65505 i
*>l                   172.16.0.13                                    0 65505 i
*>i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.13:4    (L3VNI 780000)
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>l[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                                    0 65505 i
* i                   172.16.0.14                       100          0 65505 i
*>i[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.16.0.100                      100          0 i

Leaf3# sh bgp l2vpn evpn route-type 5
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:3
BGP routing table entry for [5]:[0]:[0]:[24]:[10.0.0.0]/224, version 2739
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_A L3-770000
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.11:4
BGP routing table entry for [5]:[0]:[0]:[24]:[20.0.0.0]/224, version 2740
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_B L3-780000
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 8799
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: L3-770000 VRF_A
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 8805
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: L3-780000 VRF_B
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:3    (L3VNI 770000)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 8800
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Path type: internal, path is valid, not best reason: Locally originated, no la
beled nexthop
             Imported from 172.16.0.14:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 0.0.0.0 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08

  Path-id 1 advertised to peers:
    172.16.10.5        172.16.10.11   
BGP routing table entry for [5]:[0]:[0]:[24]:[10.0.0.0]/224, version 2504
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:3:[5]:[0]:[0]:[24]:[10.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:4    (L3VNI 780000)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 8804
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 0.0.0.0 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08

  Path type: internal, path is valid, not best reason: Locally originated, no la
beled nexthop
             Imported from 172.16.0.14:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.10.5        172.16.10.11   
BGP routing table entry for [5]:[0]:[0]:[24]:[20.0.0.0]/224, version 2507
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:4:[5]:[0]:[0]:[24]:[20.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf3# sh ip route vrf VRF_A
IP Route Table for VRF "VRF_A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 1/0
    *via 30.30.30.30, [20/0], 00:46:43, bgp-65501, external, tag 65505
10.0.0.0/24, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 00:48:18, bgp-65501, internal, tag 65501
, segid: 770000 tunnelid: 0xac100064 encap: VXLAN
 
10.0.0.11/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 00:48:18, bgp-65501, internal, tag 65501
, segid: 770000 tunnelid: 0xac100064 encap: VXLAN
 
30.30.30.30/32, ubest/mbest: 1/0
    *via 192.168.10.0, [1/0], 1d03h, static
192.168.10.0/31, ubest/mbest: 1/0, attached
    *via 192.168.10.1, Eth1/1.10, [0/0], 1d03h, direct
192.168.10.1/32, ubest/mbest: 1/0, attached
    *via 192.168.10.1, Eth1/1.10, [0/0], 1d03h, local

Leaf3# sh ip route vrf VRF_B
IP Route Table for VRF "VRF_B"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 1/0
    *via 30.30.30.30, [20/0], 00:46:50, bgp-65501, external, tag 65505
20.0.0.0/24, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 00:48:01, bgp-65501, internal, tag 65501
, segid: 780000 tunnelid: 0xac100064 encap: VXLAN
 
20.0.0.12/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 00:48:01, bgp-65501, internal, tag 65501
, segid: 780000 tunnelid: 0xac100064 encap: VXLAN
 
30.30.30.30/32, ubest/mbest: 1/0
    *via 192.168.20.0, [1/0], 1d03h, static
192.168.20.0/31, ubest/mbest: 1/0, attached
    *via 192.168.20.1, Eth1/1.20, [0/0], 1d03h, direct
192.168.20.1/32, ubest/mbest: 1/0, attached
    *via 192.168.20.1, Eth1/1.20, [0/0], 1d03h, local

```
=================================================================================================
```
Client3#sh bgp ipv4 unicast 
BGP table version is 886, local router ID is 30.30.30.30
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
     0.0.0.0          0.0.0.0                                0 i
 *m  10.0.0.0/24      192.168.10.1                           0 65501 i
 *>                   192.168.10.3                           0 65501 i
 *m  20.0.0.0/24      192.168.20.3                           0 65501 i
 *>                   192.168.20.1                           0 65501 i

Client3#sh bgp ipv4 unicast 10.0.0.0
BGP routing table entry for 10.0.0.0/24, version 885
Paths: (2 available, best #2, table default)
Multipath: eBGP
  Advertised to update-groups:
     60        
  Refresh Epoch 1
  65501
    192.168.10.1 from 192.168.10.1 (172.16.0.13)
      Origin IGP, localpref 100, valid, external, multipath(oldest)
      Extended Community: RT:7777:770000
      rx pathid: 0, tx pathid: 0
  Refresh Epoch 1
  65501
    192.168.10.3 from 192.168.10.3 (172.16.0.14)
      Origin IGP, localpref 100, valid, external, multipath, best
      Extended Community: RT:7777:770000
      rx pathid: 0, tx pathid: 0x0

Client3#sh bgp ipv4 unicast 20.0.0.0
BGP routing table entry for 20.0.0.0/24, version 886
Paths: (2 available, best #2, table default)
Multipath: eBGP
  Advertised to update-groups:
     60        
  Refresh Epoch 1
  65501
    192.168.20.3 from 192.168.20.3 (172.16.0.14)
      Origin IGP, localpref 100, valid, external, multipath(oldest)
      Extended Community: RT:7878:780000
      rx pathid: 0, tx pathid: 0
  Refresh Epoch 1
  65501
    192.168.20.1 from 192.168.20.1 (172.16.0.13)
      Origin IGP, localpref 100, valid, external, multipath, best
      Extended Community: RT:7878:780000
      rx pathid: 0, tx pathid: 0x0

Client3#sh ip route 
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override

Gateway of last resort is not set

      10.0.0.0/24 is subnetted, 1 subnets
B        10.0.0.0 [20/0] via 192.168.10.3, 00:53:17
                  [20/0] via 192.168.10.1, 00:53:17
      20.0.0.0/24 is subnetted, 1 subnets
B        20.0.0.0 [20/0] via 192.168.20.3, 00:53:16
                  [20/0] via 192.168.20.1, 00:53:16
      30.0.0.0/32 is subnetted, 1 subnets
C        30.30.30.30 is directly connected, Loopback0
      192.168.10.0/24 is variably subnetted, 4 subnets, 2 masks
C        192.168.10.0/31 is directly connected, GigabitEthernet1.10
L        192.168.10.0/32 is directly connected, GigabitEthernet1.10
C        192.168.10.2/31 is directly connected, GigabitEthernet2.10
L        192.168.10.2/32 is directly connected, GigabitEthernet2.10
      192.168.20.0/24 is variably subnetted, 4 subnets, 2 masks
C        192.168.20.0/31 is directly connected, GigabitEthernet1.20
L        192.168.20.0/32 is directly connected, GigabitEthernet1.20
C        192.168.20.2/31 is directly connected, GigabitEthernet2.20
L        192.168.20.2/32 is directly connected, GigabitEthernet2.20
```
=============================================================================
```
Client1#traceroute 20.0.0.12
Type escape sequence to abort.
Tracing the route to 20.0.0.12
VRF info: (vrf in name/id, vrf out name/id)
  1 10.0.0.254 4 msec 3 msec 1 msec
  2 192.168.10.3 8 msec
    192.168.10.1 8 msec 7 msec
  3 192.168.10.0 8 msec
    192.168.10.2 12 msec 9 msec
  4 192.168.20.3 11 msec 11 msec 11 msec
  5 20.0.0.254 17 msec 16 msec 18 msec
  6 20.0.0.12 21 msec *  33 msec
```
=======================================================================================
```
Client2#traceroute 10.0.0.11
Type escape sequence to abort.
Tracing the route to 10.0.0.11
VRF info: (vrf in name/id, vrf out name/id)
  1 20.0.0.254 4 msec 2 msec 2 msec
  2 192.168.20.3 7 msec 7 msec
    192.168.20.1 7 msec
  3 192.168.20.0 9 msec
    192.168.20.2 12 msec
    192.168.20.0 25 msec
  4 192.168.10.1 15 msec 11 msec 9 msec
  5 10.0.0.254 15 msec 22 msec 19 msec
  6 10.0.0.11 20 msec *  22 msec

```




