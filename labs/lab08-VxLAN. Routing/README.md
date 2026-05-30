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
| Client3-Leaf3| Gi1 | 192.168.1.0 | Eth1/1 | 192.168.1.1 |
| Client3-Leaf4| Gi2 | 192.168.2.0 | Eth1/1 | 192.168.2.1 |


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

## Конфигурация оборудования (VXLAN L3VNI)

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
vdc Leaf3 id 1

nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature bfd
feature nv overlay

no password strength-check
username admin password 5 $5$APHICI$69VwcanG/iitboPeY2V1rXQPezajjuwomUVQLYBm6wC 
 role network-admin
no ip domain-lookup
copp profile strict
bfd interval 100 min_rx 100 multiplier 5
evpn esi multihoming 
  ethernet-segment delay-restore time 30

fabric forwarding anycast-gateway-mac 0001.0001.0001
vlan 1,10,20,77-78,999
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
vlan 999
  vn-segment 999999

route-map RM_PERMIT_NET permit 10
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0

vrf context VRF_A
  vni 770000
  ip route 0.0.0.0/0 30.30.30.30
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target import 9999:999999
    route-target import 9999:999999 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
vrf context VRF_B
  vni 780000
  ip route 0.0.0.0/0 30.30.30.30
  rd auto
  address-family ipv4 unicast
    route-target import 7878:780000
    route-target import 7878:780000 evpn
    route-target import 9999:999999
    route-target import 9999:999999 evpn
    route-target export 7878:780000
    route-target export 7878:780000 evpn
vrf context VRF_EXT
  vni 999999
  ip route 30.30.30.30/32 192.168.1.0
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target import 7878:780000
    route-target import 7878:780000 evpn
    route-target export 9999:999999
    route-target export 9999:999999 evpn
vrf context management

hardware access-list tcam region racl 1024
hardware access-list tcam region span 0
hardware access-list tcam region rp-ipv6-qos 0
hardware access-list tcam region arp-ether 256 double-wide

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

interface Vlan999
  no shutdown
  vrf member VRF_EXT
  no ip redirects
  ip forward
  no ipv6 redirects

interface port-channel10
  description ----ESI_TO_CLIENT3_VLAN999----
  switchport access vlan 999
  ethernet-segment 10
    system-mac 1234.1234.1234

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
  member vni 999999 associate-vrf

interface Ethernet1/1
  no switchport
  vrf member VRF_EXT
  no bfd
  no ip redirects
  ip address 192.168.1.1/31
  no ipv6 redirects
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
 
router bgp 65501
  router-id 172.16.0.13
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
  neighbor 172.16.10.5
    inherit peer SPINEs
    description ----To_Spine1----
  neighbor 172.16.10.11
    inherit peer SPINEs
    description ----To_Spine2----
  vrf VRF_A
    address-family ipv4 unicast
      network 0.0.0.0/0
  vrf VRF_B
    address-family ipv4 unicast
      network 0.0.0.0/0
  vrf VRF_EXT
    router-id 172.16.0.13
    address-family ipv4 unicast
      maximum-paths 2
    neighbor 30.30.30.30
      bfd
      remote-as 65505
      description ----TO_CLIENT3_Lo----
      update-source Ethernet1/1
      ebgp-multihop 2
      timers 10 30
      address-family ipv4 unicast
        send-community
        send-community extended
```
### Leaf-4
```
hostname Leaf4
vdc Leaf4 id 1

nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature bfd
feature nv overlay

no password strength-check
username admin password 5 $5$DEFJMG$OpKaNWAlMeZhjEnUXMjLcTNMz9OIT45Abxv4dFV1yr1 
 role network-admin
ip domain-lookup
copp profile strict
bfd interval 100 min_rx 100 multiplier 5
evpn esi multihoming 
  ethernet-segment delay-restore time 30

fabric forwarding anycast-gateway-mac 0001.0001.0001
vlan 1,10,20,77-78,999
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
vlan 999
  vn-segment 999999

route-map RM_PERMIT_NET permit 10
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0

vrf context VRF_A
  vni 770000
  ip route 0.0.0.0/0 30.30.30.30
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target import 9999:999999
    route-target import 9999:999999 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
vrf context VRF_B
  vni 780000
  ip route 0.0.0.0/0 30.30.30.30
  rd auto
  address-family ipv4 unicast
    route-target import 7878:780000
    route-target import 7878:780000 evpn
    route-target import 9999:999999
    route-target import 9999:999999 evpn
    route-target export 7878:780000
    route-target export 7878:780000 evpn
vrf context VRF_EXT
  vni 999999
  ip route 30.30.30.30/32 192.168.2.0
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target import 7878:780000
    route-target import 7878:780000 evpn
    route-target export 9999:999999
    route-target export 9999:999999 evpn
 
hardware access-list tcam region racl 1024
hardware access-list tcam region span 0
hardware access-list tcam region rp-ipv6-qos 0
hardware access-list tcam region arp-ether 256 double-wide


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

interface Vlan999
  no shutdown
  vrf member VRF_EXT
  no ip redirects
  ip forward
  no ipv6 redirects

interface port-channel10
  description ----ESI_TO_CLIENT3_VLAN999----
  switchport access vlan 999
  ethernet-segment 10
    system-mac 1234.1234.1234

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
  member vni 999999 associate-vrf

interface Ethernet1/1
  no switchport
  vrf member VRF_EXT
  no bfd
  no ip redirects
  ip address 192.168.2.1/31
  no ipv6 redirects
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
    address-family ipv4 unicast
      network 0.0.0.0/0
  vrf VRF_B
    address-family ipv4 unicast
      network 0.0.0.0/0
  vrf VRF_EXT
    router-id 172.16.0.13
    address-family ipv4 unicast
      maximum-paths 2
    neighbor 30.30.30.30
      bfd
      remote-as 65505
      description ----TO_CLIENT3_Lo----
      update-source Ethernet1/1
      ebgp-multihop 2
      timers 10 30
      address-family ipv4 unicast
        send-community
        send-community extended
```
### Client3(BorderRouter/Firewall)
```
hostname Client3
!
no ip domain lookup
!
interface Loopback0
 ip address 30.30.30.30 255.255.255.255
!
interface GigabitEthernet1
 ip address 192.168.1.0 255.255.255.254
 negotiation auto
!
interface GigabitEthernet2
 ip address 192.168.2.0 255.255.255.254
 negotiation auto
!
router bgp 65505
 bgp router-id 30.30.30.30
 bgp log-neighbor-changes
 neighbor 192.168.1.1 remote-as 65501
 neighbor 192.168.2.1 remote-as 65501
 !
 address-family ipv4
  redistribute connected route-map RM_FOR_BGP_L0
  neighbor 192.168.1.1 activate
  neighbor 192.168.1.1 default-originate
  neighbor 192.168.2.1 activate
  neighbor 192.168.2.1 default-originate
  maximum-paths 2
 exit-address-family
!
ip forward-protocol nd
!
route-map RM_FOR_BGP_L0 permit 10
 match interface Loopback0
```
---

## Проверка 
```
Spine1# show ip route
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

172.16.0.11/32, ubest/mbest: 1/0
    *via 172.16.10.0, [200/0], 1w0d, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 1/0
    *via 172.16.10.2, [200/0], 1w0d, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 1/0
    *via 172.16.10.4, [200/0], 17:24:54, bgp-65501, internal, tag 65501
172.16.0.14/32, ubest/mbest: 1/0
    *via 172.16.10.14, [200/0], 1w0d, bgp-65501, internal, tag 65501
172.16.0.100/32, ubest/mbest: 2/0
    *via 172.16.10.0, [200/0], 1w0d, bgp-65501, internal, tag 65501
    *via 172.16.10.2, [200/0], 1w0d, bgp-65501, internal, tag 65501
172.16.0.111/32, ubest/mbest: 2/0, attached
    *via 172.16.0.111, Lo0, [0/0], 3w5d, local
    *via 172.16.0.111, Lo0, [0/0], 3w5d, direct
172.16.10.0/31, ubest/mbest: 1/0, attached
    *via 172.16.10.1, Eth1/1, [0/0], 3w5d, direct
172.16.10.1/32, ubest/mbest: 1/0, attached
    *via 172.16.10.1, Eth1/1, [0/0], 3w5d, local
172.16.10.2/31, ubest/mbest: 1/0, attached
    *via 172.16.10.3, Eth1/2, [0/0], 3w5d, direct
172.16.10.3/32, ubest/mbest: 1/0, attached
    *via 172.16.10.3, Eth1/2, [0/0], 3w5d, local
172.16.10.4/31, ubest/mbest: 1/0, attached
    *via 172.16.10.5, Eth1/3, [0/0], 3w5d, direct
172.16.10.5/32, ubest/mbest: 1/0, attached
    *via 172.16.10.5, Eth1/3, [0/0], 3w5d, local
172.16.10.14/31, ubest/mbest: 1/0, attached
    *via 172.16.10.15, Eth1/4, [0/0], 1w1d, direct
172.16.10.15/32, ubest/mbest: 1/0, attached
    *via 172.16.10.15, Eth1/4, [0/0], 1w1d, local

Spine1# show bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.111, local AS number 65501
BGP table version is 114, IPv4 Unicast config peers 5, capable peers 4
6 network entries and 7 paths using 1584 bytes of memory
BGP attribute entries [3/516], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.0     4 65501  222161  221765      114    0    0     1w0d 2         
172.16.10.2     4 65501  223771  223488      114    0    0     1w0d 2         
172.16.10.4     4 65501  223114  223205      114    0    0 17:25:04 1         
172.16.10.14    4 65501  223118  223152      114    0    0     1w0d 1         
Spine1# show bgp ipv4 unicast
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 114, Local Router ID is 172.16.0.111
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>i172.16.0.11/32     172.16.10.0              0        100          0 ?
*>i172.16.0.12/32     172.16.10.2              0        100          0 ?
*>i172.16.0.13/32     172.16.10.4              0        100          0 i
*>i172.16.0.14/32     172.16.10.14             0        100          0 ?
*>i172.16.0.100/32    172.16.10.0              0        100          0 ?
*|i                   172.16.10.2              0        100          0 ?
*>r172.16.0.111/32    0.0.0.0                  0        100      32768 ?

Spine1# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 2002, Local Router ID is 172.16.0.111
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:3
*>i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.11:4
*>i[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.11:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.11:32787
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.12:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.12:32787
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.13:3
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:4
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:5
*>i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.13              0        100          0 65505 ?

Route Distinguisher: 172.16.0.13:32777
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:32787
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.14:3
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:4
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:5
*>i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.14              0        100          0 65505 ?

Route Distinguisher: 172.16.0.14:32777
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:32787
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i

Spine1#  show bgp l2vpn evpn route-type 5
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:3
BGP routing table entry for [5]:[0]:[0]:[24]:[10.0.0.0]/224, version 1633
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.2        172.16.10.4        172.16.10.14   

Route Distinguisher: 172.16.0.11:4
BGP routing table entry for [5]:[0]:[0]:[24]:[20.0.0.0]/224, version 1634
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.2        172.16.10.4        172.16.10.14   

Route Distinguisher: 172.16.0.13:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1924
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.4 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.14   

Route Distinguisher: 172.16.0.13:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1925
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.4 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.14   

Route Distinguisher: 172.16.0.13:5
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 1983
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.4 (172.16.0.13)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.14   

Route Distinguisher: 172.16.0.14:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1928
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.14 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.4    

Route Distinguisher: 172.16.0.14:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1929
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.14 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.4    

Route Distinguisher: 172.16.0.14:5
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 1922
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.14 (172.16.0.14)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.4    
 ```
=======================================================================
```
Spine2# show ip route
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

172.16.0.11/32, ubest/mbest: 1/0
    *via 172.16.10.6, [200/0], 1w0d, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 1/0
    *via 172.16.10.8, [200/0], 1w0d, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 1/0
    *via 172.16.10.10, [200/0], 17:27:43, bgp-65501, internal, tag 65501
172.16.0.14/32, ubest/mbest: 1/0
    *via 172.16.10.16, [200/0], 1w0d, bgp-65501, internal, tag 65501
172.16.0.100/32, ubest/mbest: 2/0
    *via 172.16.10.6, [200/0], 1w0d, bgp-65501, internal, tag 65501
    *via 172.16.10.8, [200/0], 1w0d, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 2/0, attached
    *via 172.16.0.112, Lo0, [0/0], 3w6d, local
    *via 172.16.0.112, Lo0, [0/0], 3w6d, direct
172.16.10.6/31, ubest/mbest: 1/0, attached
    *via 172.16.10.7, Eth1/1, [0/0], 3w5d, direct
172.16.10.7/32, ubest/mbest: 1/0, attached
    *via 172.16.10.7, Eth1/1, [0/0], 3w5d, local
172.16.10.8/31, ubest/mbest: 1/0, attached
    *via 172.16.10.9, Eth1/2, [0/0], 3w5d, direct
172.16.10.9/32, ubest/mbest: 1/0, attached
    *via 172.16.10.9, Eth1/2, [0/0], 3w5d, local
172.16.10.10/31, ubest/mbest: 1/0, attached
    *via 172.16.10.11, Eth1/3, [0/0], 3w5d, direct
172.16.10.11/32, ubest/mbest: 1/0, attached
    *via 172.16.10.11, Eth1/3, [0/0], 3w5d, local
172.16.10.16/31, ubest/mbest: 1/0, attached
    *via 172.16.10.17, Eth1/4, [0/0], 1w1d, direct
172.16.10.17/32, ubest/mbest: 1/0, attached
    *via 172.16.10.17, Eth1/4, [0/0], 1w1d, local

Spine2# show bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.112, local AS number 65501
BGP table version is 173, IPv4 Unicast config peers 5, capable peers 4
6 network entries and 7 paths using 1584 bytes of memory
BGP attribute entries [3/516], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.6     4 65501  222233  221904      173    0    0     1w0d 2         
172.16.10.8     4 65501  223993  223791      173    0    0     1w0d 2         
172.16.10.10    4 65501  223156  223361      173    0    0 17:27:56 1         
172.16.10.16    4 65501  223181  223292      173    0    0     1w0d 1         
Spine2# show bgp ipv4 unicast
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 173, Local Router ID is 172.16.0.112
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>i172.16.0.11/32     172.16.10.6              0        100          0 ?
*>i172.16.0.12/32     172.16.10.8              0        100          0 ?
*>i172.16.0.13/32     172.16.10.10             0        100          0 i
*>i172.16.0.14/32     172.16.10.16             0        100          0 ?
*>i172.16.0.100/32    172.16.10.6              0        100          0 ?
*|i                   172.16.10.8              0        100          0 ?
*>r172.16.0.112/32    0.0.0.0                  0        100      32768 ?

Spine2# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 2222, Local Router ID is 172.16.0.112
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:3
*>i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.11:4
*>i[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.11:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.11:32787
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.12:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.12:32787
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.13:3
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:4
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:5
*>i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.13              0        100          0 65505 ?

Route Distinguisher: 172.16.0.13:32777
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:32787
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.14:3
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:4
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:5
*>i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.14              0        100          0 65505 ?

Route Distinguisher: 172.16.0.14:32777
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:32787
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i

Spine2# show bgp l2vpn evpn route-type 5
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:3
BGP routing table entry for [5]:[0]:[0]:[24]:[10.0.0.0]/224, version 1796
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.6 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.8        172.16.10.10       172.16.10.16   

Route Distinguisher: 172.16.0.11:4
BGP routing table entry for [5]:[0]:[0]:[24]:[20.0.0.0]/224, version 1797
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.6 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.8        172.16.10.10       172.16.10.16   

Route Distinguisher: 172.16.0.13:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 2143
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.10 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.16   

Route Distinguisher: 172.16.0.13:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 2144
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.10 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.16   

Route Distinguisher: 172.16.0.13:5
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 2203
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.10 (172.16.0.13)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.16   

Route Distinguisher: 172.16.0.14:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 2146
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.16 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.10   

Route Distinguisher: 172.16.0.14:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 2150
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.16 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.10   

Route Distinguisher: 172.16.0.14:5
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 2141
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.16 (172.16.0.14)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.10     
```
=======================================================================
```
Leaf1# show ip route
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

172.16.0.11/32, ubest/mbest: 2/0, attached
    *via 172.16.0.11, Lo0, [0/0], 1w0d, local
    *via 172.16.0.11, Lo0, [0/0], 1w0d, direct
172.16.0.12/32, ubest/mbest: 2/0
    *via 172.16.10.1, [200/0], 00:53:25, bgp-65501, internal, tag 65501
    *via 172.16.10.7, [200/0], 00:53:25, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 2/0
    *via 172.16.10.1, [200/0], 00:53:25, bgp-65501, internal, tag 65501
    *via 172.16.10.7, [200/0], 00:53:25, bgp-65501, internal, tag 65501
172.16.0.14/32, ubest/mbest: 2/0
    *via 172.16.10.1, [200/0], 00:53:25, bgp-65501, internal, tag 65501
    *via 172.16.10.7, [200/0], 00:53:25, bgp-65501, internal, tag 65501
172.16.0.100/32, ubest/mbest: 2/0, attached
    *via 172.16.0.100, Lo0, [0/0], 1w0d, local
    *via 172.16.0.100, Lo0, [0/0], 1w0d, direct
172.16.0.111/32, ubest/mbest: 1/0
    *via 172.16.10.1, [200/0], 00:53:25, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 1/0
    *via 172.16.10.7, [200/0], 00:53:25, bgp-65501, internal, tag 65501
172.16.10.0/31, ubest/mbest: 1/0, attached
    *via 172.16.10.0, Eth1/6, [0/0], 1w0d, direct
172.16.10.0/32, ubest/mbest: 1/0, attached
    *via 172.16.10.0, Eth1/6, [0/0], 1w0d, local
172.16.10.6/31, ubest/mbest: 1/0, attached
    *via 172.16.10.6, Eth1/7, [0/0], 1w0d, direct
172.16.10.6/32, ubest/mbest: 1/0, attached
    *via 172.16.10.6, Eth1/7, [0/0], 1w0d, local
172.16.39.0/31, ubest/mbest: 1/0, attached
    *via 172.16.39.0, Vlan3900, [0/0], 1w0d, direct
172.16.39.0/32, ubest/mbest: 1/0, attached
    *via 172.16.39.0, Vlan3900, [0/0], 1w0d, local

Leaf1# show bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.11, local AS number 65501
BGP table version is 74, IPv4 Unicast config peers 3, capable peers 3
7 network entries and 16 paths using 2788 bytes of memory
BGP attribute entries [13/2236], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [10/48]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.1     4 65501  222708  222211       74    0    0     1w0d 4         
172.16.10.7     4 65501  222860  222225       74    0    0     1w0d 4         
172.16.39.1     4 65501   11970   11474       74    0    0     1w0d 6         
Leaf1# show bgp ipv4 unicast
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 74, Local Router ID is 172.16.0.11
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>r172.16.0.11/32     0.0.0.0                  0        100      32768 ?
*>i172.16.0.12/32     172.16.10.1              0        100          0 ?
*|i                   172.16.10.7              0        100          0 ?
* i                   172.16.39.1              0         90          0 ?
*>i172.16.0.13/32     172.16.10.1              0        100          0 i
*|i                   172.16.10.7              0        100          0 i
* i                   172.16.39.1              0         90          0 i
*>i172.16.0.14/32     172.16.10.1              0        100          0 ?
*|i                   172.16.10.7              0        100          0 ?
* i                   172.16.39.1              0         90          0 ?
*>r172.16.0.100/32    0.0.0.0                  0        100      32768 ?
* i                   172.16.39.1              0         90          0 ?
*>i172.16.0.111/32    172.16.10.1              0        100          0 ?
* i                   172.16.39.1              0         90          0 ?
*>i172.16.0.112/32    172.16.10.7              0        100          0 ?
* i                   172.16.39.1              0         90          0 ?

Leaf1# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 2136, Local Router ID is 172.16.0.11
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
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
*>l[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100      32768 i

Route Distinguisher: 172.16.0.11:32787    (L2VNI 20000)
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100      32768 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100      32768 i
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
*>l[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100      32768 i

Route Distinguisher: 172.16.0.13:3
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 i
* i                   172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:4
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 i
* i                   172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:5
* i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.13              0        100          0 65505 ?
* i                   172.16.0.13              0        100          0 65505 ?
*>i                   172.16.0.13              0        100          0 65505 ?

Route Distinguisher: 172.16.0.13:32777
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
* i                   172.16.0.13                       100          0 i
* i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:32787
* i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i
* i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.14:3
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:4
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:5
* i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.14              0        100          0 65505 ?
* i                   172.16.0.14              0        100          0 65505 ?
*>i                   172.16.0.14              0        100          0 65505 ?

Route Distinguisher: 172.16.0.14:32777
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:32787
* i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.11:3    (L3VNI 770000)
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
*>i                   172.16.0.13                       100          0 i
*>l[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.16.0.100                      100      32768 i

Route Distinguisher: 172.16.0.11:4    (L3VNI 780000)
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
*>i                   172.16.0.13                       100          0 i
*>l[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.16.0.100                      100      32768 i

Leaf1# show bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216,
 version 2136
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.1        172.16.10.7        172.16.39.1    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/2
72, version 2127
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.1        172.16.10.7        172.16.39.1    

Leaf1# show bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32787    (L2VNI 20000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216,
 version 2134
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.1        172.16.10.7        172.16.39.1    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/2
72, version 1530
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.1        172.16.10.7        172.16.39.1    

Leaf1# show bgp l2vpn evpn route-type 5
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 2080
Paths: (3 available, best #3)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: RR Cluster Length, no lab
eled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.39.1 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Received path-id 1
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.12 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_A L3-770000
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.39.1    

Route Distinguisher: 172.16.0.13:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 2085
Paths: (3 available, best #3)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: RR Cluster Length, no lab
eled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.39.1 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Received path-id 1
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.12 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_B L3-780000
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.39.1    

Route Distinguisher: 172.16.0.13:5
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 2126
Paths: (3 available, best #3)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: RR Cluster Length, no lab
eled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.39.1 (172.16.0.12)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Received path-id 1
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.12 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.39.1    

Route Distinguisher: 172.16.0.14:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 2094
Paths: (3 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: RR Cluster Length, no lab
eled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
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
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Path-id 1 advertised to peers:
    172.16.39.1    

Route Distinguisher: 172.16.0.14:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 2096
Paths: (3 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: RR Cluster Length, no lab
eled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
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
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Path-id 1 advertised to peers:
    172.16.39.1    

Route Distinguisher: 172.16.0.14:5
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 2073
Paths: (3 available, best #3)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: RR Cluster Length, no lab
eled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.39.1 (172.16.0.12)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Received path-id 1
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.12 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.39.1    

Route Distinguisher: 172.16.0.11:3    (L3VNI 770000)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 2092
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nex
thop
             Imported from 172.16.0.14:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

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
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 2093
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nex
thop
             Imported from 172.16.0.14:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
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

Leaf1# show ip route vrf vRF_A
IP Route Table for VRF "VRF_A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 2/0, all-best (0xac10000b)
    *via 172.16.0.13%default, [200/0], 02:23:29, bgp-65501, internal, tag 65501,
 segid: 770000 tunnelid: 0xac10000d encap: VXLAN
 
    *via 172.16.0.14%default, [200/0], 02:23:29, bgp-65501, internal, tag 65501,
 segid: 770000 tunnelid: 0xac10000e encap: VXLAN
 
10.0.0.0/24, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 1w0d, direct
10.0.0.11/32, ubest/mbest: 1/0, attached
    *via 10.0.0.11, Vlan10, [190/0], 02:02:30, hmm
10.0.0.254/32, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 1w0d, local

Leaf1# sh ip route vrf VRF_B
IP Route Table for VRF "VRF_B"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 2/0, all-best (0xac10000b)
    *via 172.16.0.13%default, [200/0], 02:32:00, bgp-65501, internal, tag 65501,
 segid: 780000 tunnelid: 0xac10000d encap: VXLAN
 
    *via 172.16.0.14%default, [200/0], 02:32:00, bgp-65501, internal, tag 65501,
 segid: 780000 tunnelid: 0xac10000e encap: VXLAN
 
20.0.0.0/24, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 2d22h, direct
20.0.0.12/32, ubest/mbest: 1/0, attached
    *via 20.0.0.12, Vlan20, [190/0], 2d22h, hmm
20.0.0.254/32, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 2d22h, local

Leaf1# sh nve peer
Interface Peer-IP                                 State LearnType Uptime   Route
r-Mac       
--------- --------------------------------------  ----- --------- -------- -----
------------
nve1      172.16.0.13                             Up    CP        17:51:44 5002.
8900.1b08   
nve1      172.16.0.14                             Up    CP        1w0d     5002.
8500.1b08   

Leaf1# sh mac address-table 
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
+   10     0000.0000.0011   dynamic  0         F      F    Po10
+   20     0000.0000.0012   dynamic  0         F      F    Po20
*   77     5002.8500.1b08   static   -         F      F    nve1(172.16.0.14)
*   77     5002.8600.1b08   static   -         F      F    Vlan77
*   77     5002.8900.1b08   static   -         F      F    nve1(172.16.0.13)
*   78     5002.8500.1b08   static   -         F      F    nve1(172.16.0.14)
*   78     5002.8600.1b08   static   -         F      F    Vlan78
*   78     5002.8900.1b08   static   -         F      F    nve1(172.16.0.13)
G    -     0001.0001.0001   static   -         F      F    sup-eth1(R)
G   10     5002.8400.1b08   static   -         F      F    vPC Peer-Link(R)
G   20     5002.8400.1b08   static   -         F      F    vPC Peer-Link(R)
G 3900     5002.8400.1b08   static   -         F      F    vPC Peer-Link(R)
G   77     5002.8400.1b08   static   -         F      F    vPC Peer-Link(R)
G   78     5002.8400.1b08   static   -         F      F    vPC Peer-Link(R)
G    -     5002.8600.1b08   static   -         F      F    sup-eth1(R)
G   10     5002.8600.1b08   static   -         F      F    sup-eth1(R)
G   20     5002.8600.1b08   static   -         F      F    sup-eth1(R)
G 3900     5002.8600.1b08   static   -         F      F    sup-eth1(R)
G   77     5002.8600.1b08   static   -         F      F    sup-eth1(R)
G   78     5002.8600.1b08   static   -         F      F    sup-eth1(R)
Leaf1# sh vpc
Legend:
                (*) - local vPC is down, forwarding via vPC peer-link

vPC domain id                     : 10  
Peer status                       : peer adjacency formed ok      
vPC keep-alive status             : peer is alive                 
Configuration consistency status  : success 
Per-vlan consistency status       : success                       
Type-2 consistency status         : success 
vPC role                          : secondary                     
Number of vPCs configured         : 2   
Peer Gateway                      : Enabled
Dual-active excluded VLANs        : -
Graceful Consistency Check        : Enabled
Auto-recovery status              : Enabled, timer is off.(timeout = 240s)
Delay-restore status              : Timer is off.(timeout = 30s)
Delay-restore SVI status          : Timer is off.(timeout = 10s)
Operational Layer3 Peer-router    : Enabled
Virtual-peerlink mode             : Disabled

vPC Peer-link status
---------------------------------------------------------------------
id    Port   Status Active vlans    
--    ----   ------ -------------------------------------------------
1     Po1    up     10,20,77-78,3900                                            
         

vPC status
----------------------------------------------------------------------------
Id    Port          Status Consistency Reason                Active vlans
--    ------------  ------ ----------- ------                ---------------
11    Po10          up     success     success               10                 
         
                                                                                
         
12    Po20          up     success     success               20 
     
```
=============================================================================
```
Leaf2# show ip route
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

172.16.0.11/32, ubest/mbest: 2/0
    *via 172.16.10.3, [200/0], 1w0d, bgp-65501, internal, tag 65501
    *via 172.16.10.9, [200/0], 1w0d, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 2/0, attached
    *via 172.16.0.12, Lo0, [0/0], 1w0d, local
    *via 172.16.0.12, Lo0, [0/0], 1w0d, direct
172.16.0.13/32, ubest/mbest: 2/0
    *via 172.16.10.3, [200/0], 17:57:57, bgp-65501, internal, tag 65501
    *via 172.16.10.9, [200/0], 17:57:57, bgp-65501, internal, tag 65501
172.16.0.14/32, ubest/mbest: 2/0
    *via 172.16.10.3, [200/0], 1w0d, bgp-65501, internal, tag 65501
    *via 172.16.10.9, [200/0], 1w0d, bgp-65501, internal, tag 65501
172.16.0.100/32, ubest/mbest: 2/0, attached
    *via 172.16.0.100, Lo0, [0/0], 1w0d, local
    *via 172.16.0.100, Lo0, [0/0], 1w0d, direct
172.16.0.111/32, ubest/mbest: 1/0
    *via 172.16.10.3, [200/0], 1w0d, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 1/0
    *via 172.16.10.9, [200/0], 1w0d, bgp-65501, internal, tag 65501
172.16.10.2/31, ubest/mbest: 1/0, attached
    *via 172.16.10.2, Eth1/6, [0/0], 1w0d, direct
172.16.10.2/32, ubest/mbest: 1/0, attached
    *via 172.16.10.2, Eth1/6, [0/0], 1w0d, local
172.16.10.8/31, ubest/mbest: 1/0, attached
    *via 172.16.10.8, Eth1/7, [0/0], 1w0d, direct
172.16.10.8/32, ubest/mbest: 1/0, attached
    *via 172.16.10.8, Eth1/7, [0/0], 1w0d, local
172.16.39.0/31, ubest/mbest: 1/0, attached
    *via 172.16.39.1, Vlan3900, [0/0], 1w0d, direct
172.16.39.1/32, ubest/mbest: 1/0, attached
    *via 172.16.39.1, Vlan3900, [0/0], 1w0d, local

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.12, local AS number 65501
BGP table version is 47, IPv4 Unicast config peers 3, capable peers 3
7 network entries and 18 paths using 3028 bytes of memory
BGP attribute entries [13/2236], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [10/48]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.3     4 65501  224641  224043       47    0    0     1w0d 5         
172.16.10.9     4 65501  224965  224208       47    0    0     1w0d 5         
172.16.39.0     4 65501   12132   11451       47    0    0     1w0d 6         
Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp ipv4 unicast
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 47, Local Router ID is 172.16.0.12
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>i172.16.0.11/32     172.16.10.3              0        100          0 ?
*|i                   172.16.10.9              0        100          0 ?
* i                   172.16.39.0              0         90          0 ?
*>r172.16.0.12/32     0.0.0.0                  0        100      32768 ?
* i172.16.0.13/32     172.16.39.0              0         90          0 i
*|i                   172.16.10.9              0        100          0 i
*>i                   172.16.10.3              0        100          0 i
* i172.16.0.14/32     172.16.39.0              0         90          0 ?
*>i                   172.16.10.3              0        100          0 ?
*|i                   172.16.10.9              0        100          0 ?
* i172.16.0.100/32    172.16.10.3              0        100          0 ?
* i                   172.16.10.9              0        100          0 ?
* i                   172.16.39.0              0         90          0 ?
*>r                   0.0.0.0                  0        100      32768 ?
* i172.16.0.111/32    172.16.39.0              0         90          0 ?
*>i                   172.16.10.3              0        100          0 ?
* i172.16.0.112/32    172.16.39.0              0         90          0 ?
*>i                   172.16.10.9              0        100          0 ?

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 2017, Local Router ID is 172.16.0.12
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100      32768 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100      32768 i
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
*>l[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100      32768 i

Route Distinguisher: 172.16.0.12:32787    (L2VNI 20000)
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100      32768 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100      32768 i
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
*>l[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100      32768 i

Route Distinguisher: 172.16.0.13:3
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 i
* i                   172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:4
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 i
* i                   172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:5
* i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.13              0        100          0 65505 ?
*>i                   172.16.0.13              0        100          0 65505 ?
* i                   172.16.0.13              0        100          0 65505 ?

Route Distinguisher: 172.16.0.13:32777
* i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
* i                   172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:32787
* i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i
* i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.14:3
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:4
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:5
* i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.14              0        100          0 65505 ?
* i                   172.16.0.14              0        100          0 65505 ?
*>i                   172.16.0.14              0        100          0 65505 ?

Route Distinguisher: 172.16.0.14:32777
* i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:32787
* i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.12:3    (L3VNI 770000)
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.12:4    (L3VNI 780000)
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
*>i                   172.16.0.13                       100          0 i

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 201
7
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.3        172.16.10.9        172.16.39.0    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
2004
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08

  Path-id 1 advertised to peers:
    172.16.10.3        172.16.10.9        172.16.39.0    

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.12:32787    (L2VNI 20000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 201
5
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.3        172.16.10.9        172.16.39.0    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 
2009
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08

  Path-id 1 advertised to peers:
    172.16.10.3        172.16.10.9        172.16.39.0    

Leaf2# show bgp l2vpn evpn route-type 5
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1957
Paths: (3 available, best #3)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Path type: internal, path is valid, not best reason: RR Cluster Length, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.39.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Received path-id 1
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.11 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_A L3-770000
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.39.0    

Route Distinguisher: 172.16.0.13:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1965
Paths: (3 available, best #3)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Path type: internal, path is valid, not best reason: RR Cluster Length, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.39.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Received path-id 1
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.11 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_B L3-780000
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.39.0    

Route Distinguisher: 172.16.0.13:5
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 2003
Paths: (3 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: RR Cluster Length, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.39.0 (172.16.0.11)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Received path-id 1
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.11 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Path-id 1 advertised to peers:
    172.16.39.0    

Route Distinguisher: 172.16.0.14:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1972
Paths: (3 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: RR Cluster Length, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.39.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Received path-id 1
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.11 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_A L3-770000
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Path-id 1 advertised to peers:
    172.16.39.0    

Route Distinguisher: 172.16.0.14:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1974
Paths: (3 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: RR Cluster Length, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.39.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Received path-id 1
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.11 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_B L3-780000
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Path-id 1 advertised to peers:
    172.16.39.0    

Route Distinguisher: 172.16.0.14:5
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 1950
Paths: (3 available, best #3)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: RR Cluster Length, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.39.0 (172.16.0.11)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Received path-id 1
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.11 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.39.0    

Route Distinguisher: 172.16.0.12:3    (L3VNI 770000)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1970
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.14:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:4    (L3VNI 780000)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 2008
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.14:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf2#  show ip route vrf vRF_A
IP Route Table for VRF "VRF_A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 2/0, all-best (0xac10000c)
    *via 172.16.0.13%default, [200/0], 02:28:29, bgp-65501, internal, tag 65501, segid: 7700
00 tunnelid: 0xac10000d encap: VXLAN
 
    *via 172.16.0.14%default, [200/0], 02:28:29, bgp-65501, internal, tag 65501, segid: 7700
00 tunnelid: 0xac10000e encap: VXLAN
 
10.0.0.0/24, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 1w0d, direct
10.0.0.11/32, ubest/mbest: 1/0, attached
    *via 10.0.0.11, Vlan10, [190/0], 02:09:59, hmm
10.0.0.254/32, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 1w0d, local

Leaf2# show ip route vrf vRF_B
IP Route Table for VRF "VRF_B"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 2/0, all-best (0xac10000c)
    *via 172.16.0.13%default, [200/0], 01:48:43, bgp-65501, internal, tag 65501, segid: 7800
00 tunnelid: 0xac10000d encap: VXLAN
 
    *via 172.16.0.14%default, [200/0], 01:48:43, bgp-65501, internal, tag 65501, segid: 7800
00 tunnelid: 0xac10000e encap: VXLAN
 
20.0.0.0/24, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 2d21h, direct
20.0.0.12/32, ubest/mbest: 1/0, attached
    *via 20.0.0.12, Vlan20, [190/0], 2d21h, hmm
20.0.0.254/32, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 2d21h, local

Leaf2# sh nve peers 
Interface Peer-IP                                 State LearnType Uptime   Router-Mac       
--------- --------------------------------------  ----- --------- -------- -----------------
nve1      172.16.0.13                             Up    CP        17:58:59 5002.8900.1b08   
nve1      172.16.0.14                             Up    CP        1w0d     5002.8500.1b08   

Leaf2# sh mac address-table 
Legend: 
* - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
age - seconds since last seen,+ - primary entry using vPC Peer-Link,
(T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
*   10     0000.0000.0011   dynamic  0         F      F    Po10
*   20     0000.0000.0012   dynamic  0         F      F    Po20
*   77     5002.8400.1b08   static   -         F      F    Vlan77
*   77     5002.8500.1b08   static   -         F      F    nve1(172.16.0.14)
*   77     5002.8900.1b08   static   -         F      F    nve1(172.16.0.13)
*   78     5002.8400.1b08   static   -         F      F    Vlan78
*   78     5002.8500.1b08   static   -         F      F    nve1(172.16.0.14)
*   78     5002.8900.1b08   static   -         F      F    nve1(172.16.0.13)
G    -     0001.0001.0001   static   -         F      F    sup-eth1(R)
G    -     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G   10     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G   20     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G 3900     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G   77     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G   78     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G   10     5002.8600.1b08   static   -         F      F    vPC Peer-Link(R)
G   20     5002.8600.1b08   static   -         F      F    vPC Peer-Link(R)
G 3900     5002.8600.1b08   static   -         F      F    vPC Peer-Link(R)
G   77     5002.8600.1b08   static   -         F      F    vPC Peer-Link(R)
G   78     5002.8600.1b08   static   -         F      F    vPC Peer-Link(R)

Leaf1# sh ip route vrf VRF_B
IP Route Table for VRF "VRF_B"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 2/0, all-best (0xac10000b)
    *via 172.16.0.13%default, [200/0], 02:32:00, bgp-65501, internal, tag 65501,
 segid: 780000 tunnelid: 0xac10000d encap: VXLAN
 
    *via 172.16.0.14%default, [200/0], 02:32:00, bgp-65501, internal, tag 65501,
 segid: 780000 tunnelid: 0xac10000e encap: VXLAN
 
20.0.0.0/24, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 2d22h, direct
20.0.0.12/32, ubest/mbest: 1/0, attached
    *via 20.0.0.12, Vlan20, [190/0], 2d22h, hmm
20.0.0.254/32, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 2d22h, local
```
===============================================================================================
```
Leaf3# show ip route
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

172.16.0.11/32, ubest/mbest: 2/0
    *via 172.16.10.5, [200/0], 02:27:40, bgp-65501, internal, tag 65501
    *via 172.16.10.11, [200/0], 02:27:40, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 2/0
    *via 172.16.10.5, [200/0], 02:27:40, bgp-65501, internal, tag 65501
    *via 172.16.10.11, [200/0], 02:27:40, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 2/0, attached
    *via 172.16.0.13, Lo0, [0/0], 1w0d, local
    *via 172.16.0.13, Lo0, [0/0], 1w0d, direct
172.16.0.14/32, ubest/mbest: 2/0
    *via 172.16.10.5, [200/0], 02:27:40, bgp-65501, internal, tag 65501
    *via 172.16.10.11, [200/0], 02:27:40, bgp-65501, internal, tag 65501
172.16.0.100/32, ubest/mbest: 2/0
    *via 172.16.10.5, [200/0], 02:27:40, bgp-65501, internal, tag 65501
    *via 172.16.10.11, [200/0], 02:27:40, bgp-65501, internal, tag 65501
172.16.0.111/32, ubest/mbest: 1/0
    *via 172.16.10.5, [200/0], 02:27:40, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 1/0
    *via 172.16.10.11, [200/0], 02:27:40, bgp-65501, internal, tag 65501
172.16.10.4/31, ubest/mbest: 1/0, attached
    *via 172.16.10.4, Eth1/6, [0/0], 1w0d, direct
172.16.10.4/32, ubest/mbest: 1/0, attached
    *via 172.16.10.4, Eth1/6, [0/0], 1w0d, local
172.16.10.10/31, ubest/mbest: 1/0, attached
    *via 172.16.10.10, Eth1/7, [0/0], 1w0d, direct
172.16.10.10/32, ubest/mbest: 1/0, attached
    *via 172.16.10.10, Eth1/7, [0/0], 1w0d, local

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.13, local AS number 65501
BGP table version is 61, IPv4 Unicast config peers 2, capable peers 2
7 network entries and 11 paths using 2188 bytes of memory
BGP attribute entries [8/1376], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [6/24]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.5     4 65501  224967  223893       61    0    0 18:08:00 5         
172.16.10.11    4 65501  225213  223878       61    0    0 18:07:59 5         
Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp ipv4 unicast detail
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 172.16.0.11/32, version 55
Paths: (2 available, best #2)
Flags: (0x08001a) (high32 00000000) on xmit-list, is in urib, is best urib route, is in HW
Multipath: eBGP iBGP

  Path type: internal, path is valid, not best reason: Neighbor Address, multipath, no label
ed nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.11 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.5 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.12/32, version 56
Paths: (2 available, best #2)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib route, is in HW
Multipath: eBGP iBGP

  Path type: internal, path is valid, not best reason: Neighbor Address, multipath, no label
ed nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.11 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.5 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.13/32, version 57
Paths: (1 available, best #1)
Flags: (0x080002) (high32 00000000) on xmit-list, is not in urib
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: redist, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    0.0.0.0 (metric 0) from 0.0.0.0 (172.16.0.13)
      Origin IGP, MED 0, localpref 100, weight 32768

  Path-id 1 advertised to peers:
    172.16.10.5        172.16.10.11   
BGP routing table entry for 172.16.0.14/32, version 58
Paths: (2 available, best #1)
Flags: (0x08001a) (high32 00000000) on xmit-list, is in urib, is best urib route, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.5 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, multipath, no label
ed nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.11 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.100/32, version 59
Paths: (2 available, best #2)
Flags: (0x08001a) (high32 00000000) on xmit-list, is in urib, is best urib route, is in HW
Multipath: eBGP iBGP

  Path type: internal, path is valid, not best reason: Neighbor Address, multipath, no label
ed nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.11 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.5 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.111/32, version 60
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib route, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.5 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.112/32, version 61
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib route, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.11 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 not advertised to any peer

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 3255, Local Router ID is 172.16.0.13
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

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
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.12:32777
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.12:32787
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.13:32777    (L2VNI 10000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
*>l[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100      32768 i
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.13:32787    (L2VNI 20000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>l[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100      32768 i
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.14:3
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:4
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:5
* i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.14              0        100          0 65505 ?
*>i                   172.16.0.14              0        100          0 65505 ?

Route Distinguisher: 172.16.0.14:32777
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:32787
* i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.13:3    (L3VNI 770000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
*>l                   172.16.0.13                       100      32768 i
*>i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.16.0.100                      100          0 i
*>i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.14              0        100          0 65505 ?

Route Distinguisher: 172.16.0.13:4    (L3VNI 780000)
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
*>l                   172.16.0.13                       100      32768 i
*>i[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.16.0.100                      100          0 i
*>i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.14              0        100          0 65505 ?

Route Distinguisher: 172.16.0.13:5    (L3VNI 999999)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i
*>i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.16.0.100                      100          0 i
*>i[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.16.0.100                      100          0 i
*>l[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.13              0                     0 65505 ?

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 325
5
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
3187
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 5 destination(s)
             Imported paths list: VRF_EXT L2-10000 L3-770000 L3-999999 VRF_A
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 325
4
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
3186
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 5 destination(s)
             Imported paths list: VRF_EXT L2-10000 L3-770000 L3-999999 VRF_A
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 325
3
Paths: (2 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]
/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]
/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
3190
Paths: (2 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.
11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.
11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
3189
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.
11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.
11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:5    (L3VNI 999999)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
3188
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.
11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.
11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 324
1
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-20000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 
2745
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 5 destination(s)
             Imported paths list: VRF_B L3-999999 VRF_EXT L2-20000 L3-780000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 323
9
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-20000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 
3208
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 5 destination(s)
             Imported paths list: L2-20000 L3-780000 L3-999999 VRF_B VRF_EXT
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:32787    (L2VNI 20000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 324
2
Paths: (2 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]
/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]
/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 
3207
Paths: (2 available, best #2)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.
12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.
12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:4    (L3VNI 780000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 
3206
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.
12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.
12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:5    (L3VNI 999999)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 
3205
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.
12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.
12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp l2vpn evpn route-type 5
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:3
BGP routing table entry for [5]:[0]:[0]:[24]:[10.0.0.0]/224, version 2739
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 4 destination(s)
             Imported paths list: VRF_A L3-999999 VRF_EXT L3-770000
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
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
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 4 destination(s)
             Imported paths list: VRF_B L3-999999 VRF_EXT L3-780000
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 3015
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 4 destination(s)
             Imported paths list: VRF_EXT L3-770000 L3-999999 VRF_A
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 3026
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 4 destination(s)
             Imported paths list: VRF_EXT L3-780000 L3-999999 VRF_B
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:5
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 3002
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 4 destination(s)
             Imported paths list: VRF_B L3-770000 L3-780000 VRF_A
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:3    (L3VNI 770000)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 3024
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Path type: internal, path is valid, not best reason: Weight, no labeled nexthop
             Imported from 172.16.0.14:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path locally originated
    172.16.0.13 (metric 0) from 0.0.0.0 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08

  Path-id 1 advertised to peers:
    172.16.10.5        172.16.10.11   
BGP routing table entry for [5]:[0]:[0]:[24]:[10.0.0.0]/224, version 2504
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

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
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 2999
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.14:5:[5]:[0]:[0]:[32]:[30.30.30.30]/224 
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:4    (L3VNI 780000)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 3025
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Path type: internal, path is valid, not best reason: Weight, no labeled nexthop
             Imported from 172.16.0.14:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path locally originated
    172.16.0.13 (metric 0) from 0.0.0.0 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08

  Path-id 1 advertised to peers:
    172.16.10.5        172.16.10.11   
BGP routing table entry for [5]:[0]:[0]:[24]:[20.0.0.0]/224, version 2507
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

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
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 2998
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.14:5:[5]:[0]:[0]:[32]:[30.30.30.30]/224 
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:5    (L3VNI 999999)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 3017
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.14:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
             Imported from 172.16.0.14:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [5]:[0]:[0]:[24]:[10.0.0.0]/224, version 2512
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

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
BGP routing table entry for [5]:[0]:[0]:[24]:[20.0.0.0]/224, version 2513
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

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
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 3181
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 0.0.0.0 (172.16.0.13)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08

  Path-id 1 advertised to peers:
    172.16.10.5        172.16.10.11   

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show ip route vrf vRF_A
IP Route Table for VRF "VRF_A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 1/0
    *via 30.30.30.30, [1/0], 02:43:28, static
10.0.0.0/24, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 18:06:13, bgp-65501, internal, tag 65501, segid: 770
000 tunnelid: 0xac100064 encap: VXLAN
 
10.0.0.11/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 02:17:32, bgp-65501, internal, tag 65501, segid: 770
000 tunnelid: 0xac100064 encap: VXLAN
 
30.30.30.30/32, ubest/mbest: 1/0
    *via 30.30.30.30%VRF_EXT, [20/0], 02:43:28, bgp-65501, external, tag 65505

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show ip route vrf vRF_B
IP Route Table for VRF "VRF_B"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 1/0
    *via 30.30.30.30, [1/0], 02:43:28, static
20.0.0.0/24, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 18:06:13, bgp-65501, internal, tag 65501, segid: 780
000 tunnelid: 0xac100064 encap: VXLAN
 
20.0.0.12/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 18:06:13, bgp-65501, internal, tag 65501, segid: 780
000 tunnelid: 0xac100064 encap: VXLAN
 
30.30.30.30/32, ubest/mbest: 1/0
    *via 30.30.30.30%VRF_EXT, [20/0], 02:43:28, bgp-65501, external, tag 65505

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show ip route vrf vRF_EXT
IP Route Table for VRF "VRF_EXT"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 1/0
    *via 30.30.30.30%VRF_B, [20/0], 15:01:29, bgp-65501, external, tag 65501
10.0.0.0/24, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 18:06:13, bgp-65501, internal, tag 65501, segid: 770
000 (Asymmetric) tunnelid: 0xac100064 encap: VXLAN
 
10.0.0.11/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 02:17:32, bgp-65501, internal, tag 65501, segid: 770
000 (Asymmetric) tunnelid: 0xac100064 encap: VXLAN
 
20.0.0.0/24, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 18:06:13, bgp-65501, internal, tag 65501, segid: 780
000 (Asymmetric) tunnelid: 0xac100064 encap: VXLAN
 
20.0.0.12/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 18:06:13, bgp-65501, internal, tag 65501, segid: 780
000 (Asymmetric) tunnelid: 0xac100064 encap: VXLAN
 
30.30.30.30/32, ubest/mbest: 1/0
    *via 192.168.1.0, [1/0], 02:44:14, static
192.168.1.0/31, ubest/mbest: 1/0, attached
    *via 192.168.1.1, Eth1/1, [0/0], 1d17h, direct
192.168.1.1/32, ubest/mbest: 1/0, attached
    *via 192.168.1.1, Eth1/1, [0/0], 1d17h, local

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show nve peer
Interface Peer-IP                                 State LearnType Uptime   Router-Mac       
--------- --------------------------------------  ----- --------- -------- -----------------
nve1      172.16.0.14                             Up    CP        1w0d     5002.8500.1b08   
nve1      172.16.0.100                            Up    CP        1w0d     5002.8600.1b08   

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# sh mac address-table 
Legend: 
* - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
age - seconds since last seen,+ - primary entry using vPC Peer-Link,
(T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
C   10     0000.0000.0011   dynamic  0         F      F    nve1(172.16.0.100)
C   20     0000.0000.0012   dynamic  0         F      F    nve1(172.16.0.100)
*   77     5002.8500.1b08   static   -         F      F    nve1(172.16.0.14)
*   77     5002.8600.1b08   static   -         F      F    nve1(172.16.0.100)
*   77     5002.8900.1b08   static   -         F      F    Vlan77
*   78     5002.8500.1b08   static   -         F      F    nve1(172.16.0.14)
*   78     5002.8600.1b08   static   -         F      F    nve1(172.16.0.100)
*   78     5002.8900.1b08   static   -         F      F    Vlan78
*  999     5002.8500.1b08   static   -         F      F    nve1(172.16.0.14)
*  999     5002.8600.1b08   static   -         F      F    nve1(172.16.0.100)
*  999     5002.8900.1b08   static   -         F      F    Vlan999
G    -     0001.0001.0001   static   -         F      F    sup-eth1(R)
G    -     5002.8900.1b08   static   -         F      F    sup-eth1(R)
G   77     5002.8900.1b08   static   -         F      F    sup-eth1(R)
G   78     5002.8900.1b08   static   -         F      F    sup-eth1(R)
G  999     5002.8900.1b08   static   -         F      F    sup-eth1(R)
```
=================================================================================================
```
Leaf4# show ip route
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

172.16.0.11/32, ubest/mbest: 2/0
    *via 172.16.10.15, [200/0], 02:30:48, bgp-65501, internal, tag 65501
    *via 172.16.10.17, [200/0], 02:30:48, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 2/0
    *via 172.16.10.15, [200/0], 02:30:48, bgp-65501, internal, tag 65501
    *via 172.16.10.17, [200/0], 02:30:48, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 2/0
    *via 172.16.10.15, [200/0], 02:30:48, bgp-65501, internal, tag 65501
    *via 172.16.10.17, [200/0], 02:30:48, bgp-65501, internal, tag 65501
172.16.0.14/32, ubest/mbest: 2/0, attached
    *via 172.16.0.14, Lo0, [0/0], 1w0d, local
    *via 172.16.0.14, Lo0, [0/0], 1w0d, direct
172.16.0.100/32, ubest/mbest: 2/0
    *via 172.16.10.15, [200/0], 02:30:48, bgp-65501, internal, tag 65501
    *via 172.16.10.17, [200/0], 02:30:48, bgp-65501, internal, tag 65501
172.16.0.111/32, ubest/mbest: 1/0
    *via 172.16.10.15, [200/0], 02:30:48, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 1/0
    *via 172.16.10.17, [200/0], 02:30:48, bgp-65501, internal, tag 65501
172.16.10.14/31, ubest/mbest: 1/0, attached
    *via 172.16.10.14, Eth1/6, [0/0], 1w0d, direct
172.16.10.14/32, ubest/mbest: 1/0, attached
    *via 172.16.10.14, Eth1/6, [0/0], 1w0d, local
172.16.10.16/31, ubest/mbest: 1/0, attached
    *via 172.16.10.16, Eth1/7, [0/0], 1w0d, direct
172.16.10.16/32, ubest/mbest: 1/0, attached
    *via 172.16.10.16, Eth1/7, [0/0], 1w0d, local

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.14, local AS number 65501
BGP table version is 47, IPv4 Unicast config peers 2, capable peers 2
7 network entries and 11 paths using 2188 bytes of memory
BGP attribute entries [8/1376], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [6/24]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.15    4 65501  224670  223898       47    0    0     1w0d 5         
172.16.10.17    4 65501  224909  223904       47    0    0     1w0d 5         
Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp ipv4 unicast detail
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 172.16.0.11/32, version 41
Paths: (2 available, best #2)
Flags: (0x08001a) (high32 00000000) on xmit-list, is in urib, is best urib route, is in HW
Multipath: eBGP iBGP

  Path type: internal, path is valid, not best reason: Neighbor Address, multipath, no label
ed nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.17 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.15 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.12/32, version 42
Paths: (2 available, best #1)
Flags: (0x08001a) (high32 00000000) on xmit-list, is in urib, is best urib route, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.15 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, multipath, no label
ed nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.17 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.13/32, version 43
Paths: (2 available, best #1)
Flags: (0x08001a) (high32 00000000) on xmit-list, is in urib, is best urib route, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.15 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED 0, localpref 100, weight 0
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, multipath, no label
ed nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.17 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED 0, localpref 100, weight 0
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.14/32, version 44
Paths: (1 available, best #1)
Flags: (0x080002) (high32 00000000) on xmit-list, is not in urib
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: redist, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    0.0.0.0 (metric 0) from 0.0.0.0 (172.16.0.14)
      Origin incomplete, MED 0, localpref 100, weight 32768

  Path-id 1 advertised to peers:
    172.16.10.15       172.16.10.17   
BGP routing table entry for 172.16.0.100/32, version 45
Paths: (2 available, best #1)
Flags: (0x08001a) (high32 00000000) on xmit-list, is in urib, is best urib route, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.15 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, multipath, no label
ed nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.17 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.111/32, version 46
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib route, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.15 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.112/32, version 47
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib route, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.17 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 not advertised to any peer

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 1465, Local Router ID is 172.16.0.14
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:3
*>i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.11:4
*>i[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

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
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
* i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.12:32777
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.12:32787
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.13:3
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:4
* i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:5
*>i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.13              0        100          0 65505 ?
* i                   172.16.0.13              0        100          0 65505 ?

Route Distinguisher: 172.16.0.13:32777
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
* i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:32787
* i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.14:32777    (L2VNI 10000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>l[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100      32768 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.14:32787    (L2VNI 20000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>l[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100      32768 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.14:3    (L3VNI 770000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
*>l[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100      32768 i
* i                   172.16.0.13                       100          0 i
*>i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.16.0.100                      100          0 i
*>i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.13              0        100          0 65505 ?

Route Distinguisher: 172.16.0.14:4    (L3VNI 780000)
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>l[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.14                       100      32768 i
* i                   172.16.0.13                       100          0 i
*>i[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.16.0.100                      100          0 i
*>i[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.13              0        100          0 65505 ?

Route Distinguisher: 172.16.0.14:5    (L3VNI 999999)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>i[5]:[0]:[0]:[0]:[0.0.0.0]/224
                      172.16.0.13                       100          0 i
* i                   172.16.0.13                       100          0 i
*>i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.16.0.100                      100          0 i
*>i[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.16.0.100                      100          0 i
*>l[5]:[0]:[0]:[32]:[30.30.30.30]/224
                      172.16.0.14              0                     0 65505 ?

Leaf4# show bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 146
5
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
1399
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 5 destination(s)
             Imported paths list: VRF_EXT L2-10000 L3-770000 L3-999999 VRF_A
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 146
4
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
1394
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 5 destination(s)
             Imported paths list: VRF_EXT L2-10000 L3-770000 L3-999999 VRF_A
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 146
3
Paths: (2 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]
/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]
/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
1402
Paths: (2 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.
11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.
11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
1401
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.
11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.
11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:5    (L3VNI 999999)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
1400
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.
11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.
11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 145
2
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-20000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 
702
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 5 destination(s)
             Imported paths list: VRF_EXT L2-20000 L3-780000 L3-999999 VRF_B
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 145
0
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-20000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 
1415
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 5 destination(s)
             Imported paths list: L2-20000 L3-780000 L3-999999 VRF_B VRF_EXT
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32787    (L2VNI 20000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 145
3
Paths: (2 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]
/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]
/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 
1418
Paths: (2 available, best #2)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.
12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.
12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:4    (L3VNI 780000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 
1417
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.
12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.
12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:5    (L3VNI 999999)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 
1416
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.
12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.
12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 780000
      Extcommunity: RT:7878:780000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp l2vpn evpn route-type 5
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:3
BGP routing table entry for [5]:[0]:[0]:[24]:[10.0.0.0]/224, version 657
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 4 destination(s)
             Imported paths list: VRF_A L3-999999 VRF_EXT L3-770000
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.11:4
BGP routing table entry for [5]:[0]:[0]:[24]:[20.0.0.0]/224, version 703
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 4 destination(s)
             Imported paths list: VRF_EXT L3-780000 L3-999999 VRF_B
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:3
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1228
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 4 destination(s)
             Imported paths list: VRF_EXT L3-770000 L3-999999 VRF_A
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:4
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1233
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 4 destination(s)
             Imported paths list: VRF_EXT L3-780000 L3-999999 VRF_B
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:5
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 1387
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 4 destination(s)
             Imported paths list: VRF_B L3-770000 L3-780000 VRF_A
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:3    (L3VNI 770000)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1232
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path locally originated
    172.16.0.14 (metric 0) from 0.0.0.0 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8500.1b08

  Path type: internal, path is valid, not best reason: Weight, no labeled nexthop
             Imported from 172.16.0.13:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.10.15       172.16.10.17   
BGP routing table entry for [5]:[0]:[0]:[24]:[10.0.0.0]/224, version 658
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:3:[5]:[0]:[0]:[24]:[10.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 1389
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:5:[5]:[0]:[0]:[32]:[30.30.30.30]/224 
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:4    (L3VNI 780000)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1236
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path locally originated
    172.16.0.14 (metric 0) from 0.0.0.0 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8500.1b08

  Path type: internal, path is valid, not best reason: Weight, no labeled nexthop
             Imported from 172.16.0.13:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.10.15       172.16.10.17   
BGP routing table entry for [5]:[0]:[0]:[24]:[20.0.0.0]/224, version 724
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:4:[5]:[0]:[0]:[24]:[20.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 1388
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:5:[5]:[0]:[0]:[32]:[30.30.30.30]/224 
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:5    (L3VNI 999999)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 1235
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
             Imported from 172.16.0.13:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [5]:[0]:[0]:[24]:[10.0.0.0]/224, version 729
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:3:[5]:[0]:[0]:[24]:[10.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 770000
      Extcommunity: RT:7777:770000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [5]:[0]:[0]:[24]:[20.0.0.0]/224, version 730
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:4:[5]:[0]:[0]:[24]:[20.0.0.0]/224 
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 780000
      Extcommunity: RT:7878:780000 ENCAP:8 Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [5]:[0]:[0]:[32]:[30.30.30.30]/224, version 1222
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  Gateway IP: 0.0.0.0
  AS-Path: 65505 , path sourced external to AS
    172.16.0.14 (metric 0) from 0.0.0.0 (172.16.0.14)
      Origin incomplete, MED 0, localpref 100, weight 0
      Received label 999999
      Extcommunity: RT:9999:999999 ENCAP:8 Router MAC:5002.8500.1b08

  Path-id 1 advertised to peers:
    172.16.10.15       172.16.10.17   

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show ip route vrf vRF_A
IP Route Table for VRF "VRF_A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 1/0
    *via 30.30.30.30, [1/0], 15:08:49, static
10.0.0.0/24, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 22:29:38, bgp-65501, internal, tag 65501, segid: 770
000 tunnelid: 0xac100064 encap: VXLAN
 
10.0.0.11/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 02:24:59, bgp-65501, internal, tag 65501, segid: 770
000 tunnelid: 0xac100064 encap: VXLAN
 
30.30.30.30/32, ubest/mbest: 1/0
    *via 30.30.30.30%VRF_EXT, [20/0], 15:08:49, bgp-65501, external, tag 65505

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show ip route vrf vRF_B
IP Route Table for VRF "VRF_B"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 1/0
    *via 30.30.30.30, [1/0], 15:08:49, static
20.0.0.0/24, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 22:28:41, bgp-65501, internal, tag 65501, segid: 780
000 tunnelid: 0xac100064 encap: VXLAN
 
20.0.0.12/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 22:28:41, bgp-65501, internal, tag 65501, segid: 780
000 tunnelid: 0xac100064 encap: VXLAN
 
30.30.30.30/32, ubest/mbest: 1/0
    *via 30.30.30.30%VRF_EXT, [20/0], 15:08:49, bgp-65501, external, tag 65505

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show ip route vrf vRF_EXT
IP Route Table for VRF "VRF_EXT"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 1/0
    *via 30.30.30.30%VRF_B, [20/0], 15:08:55, bgp-65501, external, tag 65501
10.0.0.0/24, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 22:22:04, bgp-65501, internal, tag 65501, segid: 770
000 (Asymmetric) tunnelid: 0xac100064 encap: VXLAN
 
10.0.0.11/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 02:24:59, bgp-65501, internal, tag 65501, segid: 770
000 (Asymmetric) tunnelid: 0xac100064 encap: VXLAN
 
20.0.0.0/24, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 22:22:04, bgp-65501, internal, tag 65501, segid: 780
000 (Asymmetric) tunnelid: 0xac100064 encap: VXLAN
 
20.0.0.12/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 22:22:04, bgp-65501, internal, tag 65501, segid: 780
000 (Asymmetric) tunnelid: 0xac100064 encap: VXLAN
 
30.30.30.30/32, ubest/mbest: 1/0
    *via 192.168.2.0, [1/0], 22:16:08, static
192.168.2.0/31, ubest/mbest: 1/0, attached
    *via 192.168.2.1, Eth1/1, [0/0], 22:27:10, direct
192.168.2.1/32, ubest/mbest: 1/0, attached
    *via 192.168.2.1, Eth1/1, [0/0], 22:27:10, local

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show nve peer
Interface Peer-IP                                 State LearnType Uptime   Router-Mac       
--------- --------------------------------------  ----- --------- -------- -----------------
nve1      172.16.0.13                             Up    CP        18:13:38 5002.8900.1b08   
nve1      172.16.0.100                            Up    CP        1w0d     5002.8600.1b08   

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# sh mac address-table 
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
C   10     0000.0000.0011   dynamic  0         F      F    nve1(172.16.0.100)
C   20     0000.0000.0012   dynamic  0         F      F    nve1(172.16.0.100)
*   77     5002.8500.1b08   static   -         F      F    Vlan77
*   77     5002.8600.1b08   static   -         F      F    nve1(172.16.0.100)
*   77     5002.8900.1b08   static   -         F      F    nve1(172.16.0.13)
*   78     5002.8500.1b08   static   -         F      F    Vlan78
*   78     5002.8600.1b08   static   -         F      F    nve1(172.16.0.100)
*   78     5002.8900.1b08   static   -         F      F    nve1(172.16.0.13)
*  999     5002.8500.1b08   static   -         F      F    Vlan999
*  999     5002.8600.1b08   static   -         F      F    nve1(172.16.0.100)
*  999     5002.8900.1b08   static   -         F      F    nve1(172.16.0.13)
G    -     0001.0001.0001   static   -         F      F    sup-eth1(R)
G    -     5002.8500.1b08   static   -         F      F    sup-eth1(R)
G   77     5002.8500.1b08   static   -         F      F    sup-eth1(R)
G   78     5002.8500.1b08   static   -         F      F    sup-eth1(R)
G  999     5002.8500.1b08   static   -         F      F    sup-eth1(R)

```
=================================================================================================
```
Client3#sh ip bgp summary 
BGP router identifier 30.30.30.30, local AS number 65505
BGP table version is 198, main routing table version 198
6 network entries using 1488 bytes of memory
12 path entries using 1440 bytes of memory
5 multipath network entries and 10 multipath paths
6/5 BGP path/bestpath attribute entries using 1488 bytes of memory
1 BGP AS-PATH entries using 24 bytes of memory
4 BGP extended community entries using 128 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 4568 total bytes of memory
BGP activity 17/11 prefixes, 68/56 paths, scan interval 60 secs

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.1.1     4        65501    1141    1202      198    0    0 03:08:53        5
192.168.2.1     4        65501    5566    5858      198    0    0 15:26:56        5
Client3#sh bgp ipv4 unicast 
BGP table version is 198, local router ID is 30.30.30.30
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  0.0.0.0          192.168.1.1                            0 65501 i
 *m                   192.168.2.1                            0 65501 i
                      0.0.0.0                                0 i
 *>  10.0.0.0/24      192.168.1.1                            0 65501 i
 *m                   192.168.2.1                            0 65501 i
 *>  10.0.0.11/32     192.168.1.1                            0 65501 i
 *m                   192.168.2.1                            0 65501 i
 *>  20.0.0.0/24      192.168.1.1                            0 65501 i
 *m                   192.168.2.1                            0 65501 i
 *>  20.0.0.12/32     192.168.1.1                            0 65501 i
 *m                   192.168.2.1                            0 65501 i
 *>  30.30.30.30/32   0.0.0.0                  0         32768 ?
Client3#sh bgp ipv4 unicast 10.0.0.0  
BGP routing table entry for 10.0.0.0/24, version 188
Paths: (2 available, best #1, table default)
Multipath: eBGP
  Advertised to update-groups:
     18        
  Refresh Epoch 1
  65501
    192.168.1.1 from 192.168.1.1 (172.16.0.13)
      Origin IGP, localpref 100, valid, external, multipath, best
      Extended Community: RT:7777:770000
      rx pathid: 0, tx pathid: 0x0
  Refresh Epoch 1
  65501
    192.168.2.1 from 192.168.2.1 (172.16.0.13)
      Origin IGP, localpref 100, valid, external, multipath(oldest)
      Extended Community: RT:7777:770000
      rx pathid: 0, tx pathid: 0
Client3#sh bgp ipv4 unicast 20.0.0.0
BGP routing table entry for 20.0.0.0/24, version 189
Paths: (2 available, best #1, table default)
Multipath: eBGP
  Advertised to update-groups:
     18        
  Refresh Epoch 1
  65501
    192.168.1.1 from 192.168.1.1 (172.16.0.13)
      Origin IGP, localpref 100, valid, external, multipath, best
      Extended Community: RT:7878:780000
      rx pathid: 0, tx pathid: 0x0
  Refresh Epoch 1
  65501
    192.168.2.1 from 192.168.2.1 (172.16.0.13)
      Origin IGP, localpref 100, valid, external, multipath(oldest)
      Extended Community: RT:7878:780000
      rx pathid: 0, tx pathid: 0

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

Gateway of last resort is 192.168.2.1 to network 0.0.0.0

B*    0.0.0.0/0 [20/0] via 192.168.2.1, 03:12:32
                [20/0] via 192.168.1.1, 03:12:32
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
B        10.0.0.0/24 [20/0] via 192.168.2.1, 03:12:32
                     [20/0] via 192.168.1.1, 03:12:32
B        10.0.0.11/32 [20/0] via 192.168.2.1, 02:46:37
                      [20/0] via 192.168.1.1, 02:46:37
      20.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
B        20.0.0.0/24 [20/0] via 192.168.2.1, 03:12:32
                     [20/0] via 192.168.1.1, 03:12:32
B        20.0.0.12/32 [20/0] via 192.168.2.1, 03:12:32
                      [20/0] via 192.168.1.1, 03:12:32
      30.0.0.0/32 is subnetted, 1 subnets
C        30.30.30.30 is directly connected, Loopback0
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/31 is directly connected, GigabitEthernet1
L        192.168.1.0/32 is directly connected, GigabitEthernet1
      192.168.2.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.2.0/31 is directly connected, GigabitEthernet2
L        192.168.2.0/32 is directly connected, GigabitEthernet2
```
=============================================================================
```
Client1#ping 20.0.0.12 
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 20.0.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 16/24/29 ms
Client1#sh ip arp 
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  10.0.0.11               -   0000.0000.0011  ARPA   Port-channel1
Internet  10.0.0.254              2   0001.0001.0001  ARPA   Port-channel1
```
=======================================================================================
```
Client2#ping 10.0.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.11, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 20/26/33 ms
Client2#sh ip arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  20.0.0.12               -   0000.0000.0012  ARPA   Port-channel1
Internet  20.0.0.254             17   0001.0001.0001  ARPA   Port-channel1
```

