# Настройка VxLAN Multihoming

### Цель работы
Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming.

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

### Loopback адреса

| Устройство | Loopback0 IP |
|------------|--------------|
| Spine-1 | 172.16.0.111/32 |
| Spine-2 | 172.16.0.112/32 |
| Leaf-1 | 172.16.0.11/32 |
| Leaf-2 | 172.16.0.12/32 |
| Leaf-3 | 172.16.0.13/32 |
| Leaf-4 | 172.16.0.14/32 |

### Серверные подсети

| Leaf | VLAN | Подсеть | Шлюз |
|------|------|---------|------|
| Leaf-1 | 10 | 10.0.0.0/24 | 10.0.0.254 |
| Leaf-1 | 20 | 20.0.0.0/24 | 20.0.0.254 |
| Leaf-2 | 10 | 10.0.0.0/24 | 10.0.0.254 |
| Leaf-2 | 20 | 20.0.0.0/24 | 20.0.0.254 |
| Leaf-3 | 10 | 10.0.0.0/24 | 10.0.0.254 |
| Leaf-4 | 10 | 10.0.0.0/24 | 10.0.0.254 |

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
!
nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature vpc
feature bfd
feature nv overlay
!
no ip domain-lookup
!
bfd interval 100 min_rx 100 multiplier 5
!
fabric forwarding anycast-gateway-mac 0001.0001.0001
vlan 1,10,20,77,3900
vlan 10
  name SEVERS_VLAN10
  vn-segment 10000
vlan 20
  name SEVERS_VLAN20
  vn-segment 20000
vlan 77
  name L3VNI
  vn-segment 770000
vlan 3900
  name BACKUP_VLAN_ROUTING
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
route-map SET_NH_FOR_LEAF2 permit 10
  set ip next-hop 172.16.39.0    
route-map SET_NH_FOR_SPINE permit 10
  set ip next-hop peer-address
route-map SET_LP_FROM_LEAF2 permit 10
  set local-preference 90
!
vrf context VRF_A
  vni 770000
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
!
hardware access-list tcam region racl 1024
hardware access-list tcam region span 0
hardware access-list tcam region rp-ipv6-qos 0
hardware access-list tcam region arp-ether 256 double-wide
!
vpc domain 10
  peer-switch
  peer-keepalive destination 192.168.1.1 source 192.168.1.0
  peer-gateway
  layer3 peer-router
  auto-recovery
  fast-convergence
  ip arp synchronize
!
interface Vlan10
  description ----SERVERS_VLAN10----
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip address 10.0.0.254/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway
!
interface Vlan20
  description ----SERVERS_VLAN20----
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip address 20.0.0.254/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway
!
interface Vlan77
  description ####L3VNI_770000####
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip forward
  no ipv6 redirects
!
interface Vlan3900
  description VPC_L3_Peering_VXLAN
  no shutdown
  mtu 9216
  no ip redirects
  ip address 172.16.39.0/31
  no ipv6 redirects
!
interface port-channel1
  description ----VPC_PEER_TO_LEAF12
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77,3900
  spanning-tree port type network
  vpc peer-link
!
interface port-channel10
  description ----VPC_CLIENT1----
  switchport
  switchport access vlan 10
  vpc 11
!
interface port-channel20
  description ----VPC_CLIENT2----
  switchport
  switchport access vlan 20
  vpc 12
!
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
!
interface Ethernet1/1
  description ----TO_CLIENT_VLAN10----
  switchport
  switchport access vlan 10
  spanning-tree bpduguard enable
  spanning-tree bpdufilter enable
  channel-group 10 mode active
  no shutdown
!
interface Ethernet1/2
  description ----TO_CLIENT_VLAN20----
  switchport
  switchport access vlan 20
  spanning-tree bpduguard enable
  spanning-tree bpdufilter enable
  channel-group 20 mode active
  no shutdown
!
interface Ethernet1/4
  description ----VPC_PEERLINK----
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77,3900
  channel-group 1 mode active
  no shutdown
!
interface Ethernet1/5
  description ----VPC_PEERLINK----
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77,3900
  channel-group 1 mode active
  no shutdown
!
interface Ethernet1/6
  description Link_To_Spine1
  no ip redirects
  ip address 172.16.10.0/31
  no ipv6 redirects
!
interface Ethernet1/7
  description Link_To_Spine2
  no ip redirects
  ip address 172.16.10.6/31
  no ipv6 redirects
!
interface mgmt0
  vrf member management
  ip address 192.168.1.0/31
!
interface loopback0
  ip address 172.16.0.11/32
  ip address 172.16.0.100/32 secondary
!
router bgp 65501
  router-id 172.16.0.11
  address-family ipv4 unicast
    redistribute direct route-map RM_RED_FOR_BGP
    maximum-paths 4
    maximum-paths ibgp 4
    additional-paths receive
    additional-paths install backup
  address-family l2vpn evpn
    additional-paths send
    additional-paths receive
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
      route-map SET_NH_FOR_LEAF2 out
      route-map SET_LP_FROM_LEAF2 in
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
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
!
nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature vpc
feature bfd
feature nv overlay
!
no ip domain-lookup
!
bfd interval 100 min_rx 100 multiplier 5
!
fabric forwarding anycast-gateway-mac 0001.0001.0001
!
vlan 1,10,20,77,3900
vlan 10
  name FOR_CLIENT
  vn-segment 10000
vlan 20
  name SERVERS_VLAN20
  vn-segment 20000
vlan 77
  name L3VNI
  vn-segment 770000
vlan 3900
  name BACKUP_VLAN_ROUTING
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
route-map SET_NH_FOR_LEAF1 permit 10
  set ip next-hop 172.16.39.1    
route-map SET_NH_FOR_SPINE permit 10
  set ip next-hop peer-address
route-map SET_LP_FROM_LEAF1 permit 10
  set local-preference 90
!
vrf context VRF_A
  vni 770000
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
!
hardware access-list tcam region racl 1024
hardware access-list tcam region span 0
hardware access-list tcam region rp-ipv6-qos 0
hardware access-list tcam region arp-ether 256 double-wide
!
vpc domain 10
  peer-switch
  peer-keepalive destination 192.168.1.0 source 192.168.1.1
  peer-gateway
  layer3 peer-router
  auto-recovery
  fast-convergence
  ip arp synchronize
!
interface Vlan10
  description ----SERVERS_VLAN10----
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip address 10.0.0.254/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway
!
interface Vlan20
  description ----SERVERS_VLAN20----
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip address 20.0.0.254/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway
!
interface Vlan77
  description ####L3VNI_770000####
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip forward
  no ipv6 redirects
!
interface Vlan3900
  description VPC_L3_Peering_VXLAN
  no shutdown
  mtu 9216
  no ip redirects
  ip address 172.16.39.1/31
  no ipv6 redirects
!
interface port-channel1
  description ----VPC_PEER_TO_LEAF11----
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77,3900
  spanning-tree port type network
  vpc peer-link
!
interface port-channel10
  description ----VPC_EP1----
  switchport
  switchport access vlan 10
  vpc 11
!
interface port-channel20
  description ----VPC_EP2----
  switchport
  switchport access vlan 20
  vpc 12
!
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
!
interface Ethernet1/1
  description ----TO_CLIENT_VLAN10----~
  switchport
  switchport access vlan 10
  spanning-tree bpduguard enable
  spanning-tree bpdufilter enable
  channel-group 10 mode active
  no shutdown
!
interface Ethernet1/2
  description ----TO_CLIENT_VLAN20----
  switchport
  switchport access vlan 20
  spanning-tree bpduguard enable
  spanning-tree bpdufilter enable
  channel-group 20 mode active
  no shutdown
!
interface Ethernet1/4
  description ----VPC_PEERLINK----
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77,3900
  channel-group 1 mode active
  no shutdown
!
interface Ethernet1/5
  description ----VPC_PEERLINK----
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77,3900
  channel-group 1 mode active
  no shutdown
!
interface Ethernet1/6
  description Link_To_Spine1
  no ip redirects
  ip address 172.16.10.2/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/7
  description Link_To_Spine2
  no ip redirects
  ip address 172.16.10.8/31
  no ipv6 redirects
  no shutdown
!
interface mgmt0
  vrf member management
  ip address 192.168.1.1/31
!
interface loopback0
  ip address 172.16.0.12/32
  ip address 172.16.0.100/32 secondary
!
router bgp 65501
  router-id 172.16.0.12
  address-family ipv4 unicast
    redistribute direct route-map RM_RED_FOR_BGP
    maximum-paths 2
    maximum-paths ibgp 2
  address-family l2vpn evpn
    additional-paths send
    additional-paths receive
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
      route-map SET_NH_FOR_LEAF1 out
      route-map SET_LP_FROM_LEAF1 in
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
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
!
nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature bfd
feature nv overlay
!
no ip domain-lookup
!
bfd interval 100 min_rx 100 multiplier 5
!
evpn esi multihoming 
  ethernet-segment delay-restore time 30
!
fabric forwarding anycast-gateway-mac 0001.0001.0001
!
vlan 1,10,77
vlan 10
  name SERVERS_VLAN10
  vn-segment 10000
vlan 77
  name L3VNI
  vn-segment 770000
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
!
vrf context VRF_A
  vni 770000
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
!
interface Vlan10
  description ----SERVERS_VLAN10----
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip address 10.0.0.254/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway
!
interface Vlan77
  description ----L3VNI_770000----
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip forward
  no ipv6 redirects
!
interface port-channel10
  description ---PC_CLIENT_VLAN10---
  switchport access vlan 10
  ethernet-segment 10
    system-mac 1234.1234.1234
!
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  global ingress-replication protocol bgp
  member vni 10000
    suppress-arp
  member vni 770000 associate-vrf
!
interface Ethernet1/1
  description ----To_SERVERS_VLAN10----
  switchport access vlan 10
  channel-group 10 mode active
!
interface Ethernet1/6
  description ----To_Spine1----
  no switchport
  evpn multihoming core-tracking
  no ip redirects
  ip address 172.16.10.4/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/7
  description ----To_Spine2----
  no switchport
  evpn multihoming core-tracking
  no ip redirects
  ip address 172.16.10.10/31
  no ipv6 redirects
  no shutdown
!
interface loopback0
  ip address 172.16.0.13/32
!
router bgp 65501
  router-id 172.16.0.13
  address-family ipv4 unicast
    redistribute direct route-map RM_RED_FOR_BGP
    maximum-paths 4
    maximum-paths ibgp 4
    additional-paths receive
    additional-paths install backup
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
evpn
  vni 10000 l2
    rd auto
    route-target import 65501:10000
    route-target export 65501:10000
```
### Leaf-4
```
hostname Leaf4
!
nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature bfd
feature nv overlay
!
ip domain-lookup
!
bfd interval 100 min_rx 100 multiplier 5
!
evpn esi multihoming 
  ethernet-segment delay-restore time 30
!
fabric forwarding anycast-gateway-mac 0001.0001.0001
!
vlan 1,10,77
vlan 10
  name SERVERS_VLAN10
  vn-segment 10000
vlan 77
  name L3VNI
  vn-segment 770000
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0
!
vrf context VRF_A
  vni 770000
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
!
interface Vlan10
  description ----SERVERS_VLAN10----
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip address 10.0.0.254/24
  no ipv6 redirects
  fabric forwarding mode anycast-gateway
!
interface Vlan77
  description ----L3VNI_770000----
  no shutdown
  vrf member VRF_A
  no ip redirects
  ip forward
  no ipv6 redirects
!
interface port-channel10
  description ---PC_CLIENT_VLAN10---
  switchport access vlan 10
  ethernet-segment 10
    system-mac 1234.1234.1234
!
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  global ingress-replication protocol bgp
  member vni 10000
    suppress-arp
  member vni 770000 associate-vrf
!
interface Ethernet1/1
  description ----To_SERVERS_VLAN10----
  switchport access vlan 10
  channel-group 10 mode active
!
interface Ethernet1/6
  description ----To_Spine1----
  no switchport
  evpn multihoming core-tracking
  no ip redirects
  ip address 172.16.10.14/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/7
  description ----To_Spine2----
  no switchport
  evpn multihoming core-tracking
  no ip redirects
  ip address 172.16.10.16/31
  no ipv6 redirects
  no shutdown
!
interface loopback0
  ip address 172.16.0.14/32
!
router bgp 65501
  router-id 172.16.0.14
  address-family ipv4 unicast
    redistribute direct route-map RM_RED_FOR_BGP
    maximum-paths 4
    maximum-paths ibgp 4
    additional-paths receive
    additional-paths install backup
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
evpn
  vni 10000 l2
    rd auto
    route-target import 65501:10000
    route-target export 65501:10000
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
    *via 172.16.10.0, [200/0], 1d06h, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 1/0
    *via 172.16.10.2, [200/0], 1d06h, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 1/0
    *via 172.16.10.4, [200/0], 18:09:25, bgp-65501, internal, tag 65501
172.16.0.14/32, ubest/mbest: 1/0
    *via 172.16.10.14, [200/0], 18:12:09, bgp-65501, internal, tag 65501
172.16.0.100/32, ubest/mbest: 2/0
    *via 172.16.10.0, [200/0], 1d06h, bgp-65501, internal, tag 65501
    *via 172.16.10.2, [200/0], 1d06h, bgp-65501, internal, tag 65501
172.16.0.111/32, ubest/mbest: 2/0, attached
    *via 172.16.0.111, Lo0, [0/0], 2w4d, local
    *via 172.16.0.111, Lo0, [0/0], 2w4d, direct
172.16.10.0/31, ubest/mbest: 1/0, attached
    *via 172.16.10.1, Eth1/1, [0/0], 2w4d, direct
172.16.10.1/32, ubest/mbest: 1/0, attached
    *via 172.16.10.1, Eth1/1, [0/0], 2w4d, local
172.16.10.2/31, ubest/mbest: 1/0, attached
    *via 172.16.10.3, Eth1/2, [0/0], 2w4d, direct
172.16.10.3/32, ubest/mbest: 1/0, attached
    *via 172.16.10.3, Eth1/2, [0/0], 2w4d, local
172.16.10.4/31, ubest/mbest: 1/0, attached
    *via 172.16.10.5, Eth1/3, [0/0], 2w4d, direct
172.16.10.5/32, ubest/mbest: 1/0, attached
    *via 172.16.10.5, Eth1/3, [0/0], 2w4d, local
172.16.10.14/31, ubest/mbest: 1/0, attached
    *via 172.16.10.15, Eth1/4, [0/0], 20:06:42, direct
172.16.10.15/32, ubest/mbest: 1/0, attached
    *via 172.16.10.15, Eth1/4, [0/0], 20:06:43, local

Spine1# show bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.111, local AS number 65501
BGP table version is 74, IPv4 Unicast config peers 5, capable peers 4
6 network entries and 7 paths using 1584 bytes of memory
BGP attribute entries [3/516], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.0     4 65501  431154  431055       74    0    0     2w1d 2         
172.16.10.2     4 65501  431559  431464       74    0    0     2w1d 2         
172.16.10.4     4 65501   21734   21755       74    0    0 18:10:03 1         
172.16.10.14    4 65501   21786   21797       74    0    0 18:12:19 1         
Spine1# show bgp ipv4 unicast
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 74, Local Router ID is 172.16.0.111
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
*|i172.16.0.100/32    172.16.10.2              0        100          0 ?
*>i                   172.16.10.0              0        100          0 ?
*>r172.16.0.111/32    0.0.0.0                  0        100      32768 ?

Spine1# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 431, Local Router ID is 172.16.0.111
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
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

Route Distinguisher: 172.16.0.13:40
*>i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:27009
*>i[4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.13]/136
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:32777
*>i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.13                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.13                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.14:40
*>i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:27009
*>i[4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.14]/136
                      172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:32777
*>i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.14                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.14                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.14                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i

Spine1# show bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216,
 version 427
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.2        172.16.10.4        172.16.10.14   
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 120
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.2        172.16.10.4        172.16.10.14   

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 426
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.2 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.4        172.16.10.14   
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 121
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.2 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.4        172.16.10.14   

Spine1# show bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 431
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.2        172.16.10.4        172.16.10.14   
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 124
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.2        172.16.10.4        172.16.10.14   

Route Distinguisher: 172.16.0.12:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 430
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.2 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.4        172.16.10.14   
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 127
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.2 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.4        172.16.10.14   

Spine1# show bgp l2vpn evpn 0000.0000.0013
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 288
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.4 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.14   

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216, version 299
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.14 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.4    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 287
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.14 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.4    

Spine1# show bgp l2vpn evpn route-type 1
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:40
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 275
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.4 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.14   

Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 276
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.4 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.14   

Route Distinguisher: 172.16.0.14:40
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 283
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.14 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.4    

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 284
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.14 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.4    

Spine1# show bgp l2vpn evpn route-type 3
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 117
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path-id 1 advertised to peers:
    172.16.10.2        172.16.10.4        172.16.10.14   

Route Distinguisher: 172.16.0.11:32787
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 118
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

------------------------


  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:20000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 20000, Tunnel Id: 172.16.0.100

  Path-id 1 advertised to peers:
    172.16.10.2        172.16.10.4        172.16.10.14   

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 119
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.2 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.4        172.16.10.14   

Route Distinguisher: 172.16.0.12:32787
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 129
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.2 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:20000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 20000, Tunnel Id: 172.16.0.100

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.4        172.16.10.14   

Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.13]/88, version 272
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.4 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.13

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.14   

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.14]/88, version 269
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.14 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.14

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.4    

Spine1# show bgp l2vpn evpn route-type 4
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:27009
BGP routing table entry for [4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.13]/13
6, version 273
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.4 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: ENCAP:8 RT:1234.1234.1234

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.14   

Route Distinguisher: 172.16.0.14:27009
BGP routing table entry for [4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.14]/13
6, version 279
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.14 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: ENCAP:8 RT:1234.1234.1234

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
    *via 172.16.10.6, [200/0], 1d04h, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 1/0
    *via 172.16.10.8, [200/0], 1d04h, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 1/0
    *via 172.16.10.10, [200/0], 18:14:06, bgp-65501, internal, tag 65501
172.16.0.14/32, ubest/mbest: 1/0
    *via 172.16.10.16, [200/0], 18:17:01, bgp-65501, internal, tag 65501
172.16.0.100/32, ubest/mbest: 2/0
    *via 172.16.10.6, [200/0], 1d04h, bgp-65501, internal, tag 65501
    *via 172.16.10.8, [200/0], 1d04h, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 2/0, attached
    *via 172.16.0.112, Lo0, [0/0], 2w4d, local
    *via 172.16.0.112, Lo0, [0/0], 2w4d, direct
172.16.10.6/31, ubest/mbest: 1/0, attached
    *via 172.16.10.7, Eth1/1, [0/0], 2w4d, direct
172.16.10.7/32, ubest/mbest: 1/0, attached
    *via 172.16.10.7, Eth1/1, [0/0], 2w4d, local
172.16.10.8/31, ubest/mbest: 1/0, attached
    *via 172.16.10.9, Eth1/2, [0/0], 2w4d, direct
172.16.10.9/32, ubest/mbest: 1/0, attached
    *via 172.16.10.9, Eth1/2, [0/0], 2w4d, local
172.16.10.10/31, ubest/mbest: 1/0, attached
    *via 172.16.10.11, Eth1/3, [0/0], 2w4d, direct
172.16.10.11/32, ubest/mbest: 1/0, attached
    *via 172.16.10.11, Eth1/3, [0/0], 2w4d, local
172.16.10.16/31, ubest/mbest: 1/0, attached
    *via 172.16.10.17, Eth1/4, [0/0], 20:09:57, direct
172.16.10.17/32, ubest/mbest: 1/0, attached
    *via 172.16.10.17, Eth1/4, [0/0], 20:09:57, local

Spine2# show bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.112, local AS number 65501
BGP table version is 128, IPv4 Unicast config peers 5, capable peers 4
6 network entries and 7 paths using 1584 bytes of memory
BGP attribute entries [3/516], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.6     4 65501   37211   37149      128    0    0    1d04h 2         
172.16.10.8     4 65501   37194   37133      128    0    0    1d04h 2         
172.16.10.10    4 65501   21819   21837      128    0    0 18:14:16 1         
172.16.10.16    4 65501   21883   21894      128    0    0 18:17:12 1         
Spine2# show bgp ipv4 unicast
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 128, Local Router ID is 172.16.0.112
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
*|i172.16.0.100/32    172.16.10.8              0        100          0 ?
*>i                   172.16.10.6              0        100          0 ?
*>r172.16.0.112/32    0.0.0.0                  0        100      32768 ?

Spine2# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 338, Local Router ID is 172.16.0.112
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
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

Route Distinguisher: 172.16.0.13:40
*>i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:27009
*>i[4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.13]/136
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:32777
*>i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.13                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.13                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.14:40
*>i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:27009
*>i[4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.14]/136
                      172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:32777
*>i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.14                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.14                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.14                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i

Spine2# show bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 334
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.6 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.8        172.16.10.10       172.16.10.16   
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 35
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.6 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.8        172.16.10.10       172.16.10.16   

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 333
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.8 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.10       172.16.10.16   
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 40
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.8 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.10       172.16.10.16   

Spine2# show bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 338
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.6 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.8        172.16.10.10       172.16.10.16   
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 36
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.6 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.8        172.16.10.10       172.16.10.16   

Route Distinguisher: 172.16.0.12:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 337
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.8 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.10       172.16.10.16   
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 41
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.8 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.10       172.16.10.16   

Spine2# show bgp l2vpn evpn 0000.0000.0013
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 195
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.10 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.16   

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216, version 206
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.16 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.10   
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 194
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.16 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.10   

Spine2# show bgp l2vpn evpn route-type 1
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:40
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 182
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.10 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.16   

Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 183
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.10 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.16   

Route Distinguisher: 172.16.0.14:40
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 190
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.16 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.10   

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 191
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.16 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.10   

Spine2# show bgp l2vpn evpn route-type 3
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 33
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.6 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path-id 1 advertised to peers:
    172.16.10.8        172.16.10.10       172.16.10.16   

Route Distinguisher: 172.16.0.11:32787
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 34
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.6 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:20000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 20000, Tunnel Id: 172.16.0.100

  Path-id 1 advertised to peers:
    172.16.10.8        172.16.10.10       172.16.10.16   

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 39
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.8 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.10       172.16.10.16   

Route Distinguisher: 172.16.0.12:32787
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 43
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.8 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:20000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 20000, Tunnel Id: 172.16.0.100

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.10       172.16.10.16   

Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.13]/88, version 179
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.10 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.13

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.16   

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.14]/88, version 176
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.16 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.14

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.10   

Spine2# show bgp l2vpn evpn route-type 4
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:27009
BGP routing table entry for [4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.13]/13
6, version 180
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.10 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: ENCAP:8 RT:1234.1234.1234

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.16   

Route Distinguisher: 172.16.0.14:27009
BGP routing table entry for [4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.14]/136, version 186
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.16 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: ENCAP:8 RT:1234.1234.1234

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
    *via 172.16.0.11, Lo0, [0/0], 01:52:07, local
    *via 172.16.0.11, Lo0, [0/0], 01:52:07, direct
172.16.0.12/32, ubest/mbest: 2/0
    *via 172.16.10.1, [200/0], 00:52:11, bgp-65501, internal, tag 65501
    *via 172.16.10.7, [200/0], 00:52:11, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 2/0
    *via 172.16.10.1, [200/0], 00:52:11, bgp-65501, internal, tag 65501
    *via 172.16.10.7, [200/0], 00:52:11, bgp-65501, internal, tag 65501
172.16.0.14/32, ubest/mbest: 2/0
    *via 172.16.10.1, [200/0], 00:52:11, bgp-65501, internal, tag 65501
    *via 172.16.10.7, [200/0], 00:52:11, bgp-65501, internal, tag 65501
172.16.0.100/32, ubest/mbest: 2/0, attached
    *via 172.16.0.100, Lo0, [0/0], 01:52:07, local
    *via 172.16.0.100, Lo0, [0/0], 01:52:07, direct
172.16.0.111/32, ubest/mbest: 1/0
    *via 172.16.10.1, [200/0], 00:52:11, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 1/0
    *via 172.16.10.7, [200/0], 00:52:11, bgp-65501, internal, tag 65501
172.16.10.0/31, ubest/mbest: 1/0, attached
    *via 172.16.10.0, Eth1/6, [0/0], 01:02:47, direct
172.16.10.0/32, ubest/mbest: 1/0, attached
    *via 172.16.10.0, Eth1/6, [0/0], 01:02:47, local
172.16.10.6/31, ubest/mbest: 1/0, attached
    *via 172.16.10.6, Eth1/7, [0/0], 01:02:48, direct
172.16.10.6/32, ubest/mbest: 1/0, attached
    *via 172.16.10.6, Eth1/7, [0/0], 01:02:48, local
172.16.39.0/31, ubest/mbest: 1/0, attached
    *via 172.16.39.0, Vlan3900, [0/0], 01:55:01, direct
172.16.39.0/32, ubest/mbest: 1/0, attached
    *via 172.16.39.0, Vlan3900, [0/0], 01:55:01, local

Leaf1# show bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.11, local AS number 65501
BGP table version is 46, IPv4 Unicast config peers 3, capable peers 3
7 network entries and 16 paths using 2788 bytes of memory
BGP attribute entries [13/2236], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [10/48]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.1     4 65501    1300    1279       46    0    0 01:03:40 4         
172.16.10.7     4 65501    1301    1284       46    0    0 01:03:52 4         
172.16.39.1     4 65501     152     127       46    0    0 01:55:32 6         

Leaf1#  show bgp ipv4 unicast
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 46, Local Router ID is 172.16.0.11
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>r172.16.0.11/32     0.0.0.0                  0        100      32768 ?
*>i172.16.0.12/32     172.16.10.1              0        100          0 ?
*|i                   172.16.10.7              0        100          0 ?
*&i                   172.16.39.1              0         90          0 ?
*>i172.16.0.13/32     172.16.10.1              0        100          0 i
*|i                   172.16.10.7              0        100          0 i
*&i                   172.16.39.1              0         90          0 i
*>i172.16.0.14/32     172.16.10.1              0        100          0 ?
*|i                   172.16.10.7              0        100          0 ?
*&i                   172.16.39.1              0         90          0 ?
*>r172.16.0.100/32    0.0.0.0                  0        100      32768 ?
*&i                   172.16.39.1              0         90          0 ?
*>i172.16.0.111/32    172.16.10.1              0        100          0 ?
*&i                   172.16.39.1              0         90          0 ?
*>i172.16.0.112/32    172.16.10.7              0        100          0 ?
*&i                   172.16.39.1              0         90          0 ?

Leaf1# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 497, Local Router ID is 172.16.0.11
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
* i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.14                       100          0 i
*>i                   172.16.0.13                       100          0 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.14                       100          0 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100      32768 i
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.14                       100          0 i
*>i                   172.16.0.13                       100          0 i
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
*>l[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100      32768 i

Route Distinguisher: 172.16.0.11:65534    (L2VNI 0)
* i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.14                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:40
* i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:32777
* i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i
* i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.14:40
*>i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:32777
*>i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.11:3    (L3VNI 770000)
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.14                       100          0 i
*>i                   172.16.0.13                       100          0 i

Leaf1# show bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 495
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
    172.16.10.1        172.16.10.7    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 172
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
    172.16.10.1        172.16.10.7    

Leaf1# show bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32787    (L2VNI 20000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 497
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
    172.16.10.1        172.16.10.7    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 174
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08

  Path-id 1 advertised to peers:
    172.16.10.1        172.16.10.7    

Leaf1# show bgp l2vpn evpn 0000.0000.0013
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216, version 423
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.14:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/2
72, version 407
Paths: (2 available, best #2)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW
  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.14:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.13:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 405
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 
      ESI: 0312.3412.3412.3400.000a

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 3 destination(s)
             Imported paths list: VRF_A L2-10000 L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216, version 422
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 402
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 
      ESI: 0312.3412.3412.3400.000a

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 3 destination(s)
             Imported paths list: VRF_A L2-10000 L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.11:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 406
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.14:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer

Leaf1# show bgp l2vpn evpn route-type 1
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 397
Paths: (2 available, best #2)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.14:32777:[1]:[0312.3412.3412.3400.000a]:[0x0]/152 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.13:32777:[1]:[0312.3412.3412.3400.000a]:[0x0]/152 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.11:65534    (L2VNI 0)
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 394
Paths: (2 available, best #2)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.14:40:[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.13:40:[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:40
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 374
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: ead_es_global_ctx
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 375
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:40
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 393
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: ead_es_global_ctx
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 395
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Leaf1# show bgp l2vpn evpn route-type 3
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
BGP routing table entry for [3]:[0]:[32]:[172.16.0.13]/88, version 368
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:32777:[3]:[0]:[32]:[172.16.0.13]/88 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.13

  Path-id 1 not advertised to any peer
BGP routing table entry for [3]:[0]:[32]:[172.16.0.14]/88, version 366
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.14:32777:[3]:[0]:[32]:[172.16.0.14]/88 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.14

  Path-id 1 not advertised to any peer
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 170
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path-id 1 advertised to peers:
    172.16.10.1        172.16.10.7    

Route Distinguisher: 172.16.0.11:32787    (L2VNI 20000)
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 171
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Extcommunity: RT:65501:20000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 20000, Tunnel Id: 172.16.0.100

  Path-id 1 advertised to peers:
    172.16.10.1        172.16.10.7    

Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.13]/88, version 369
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.13

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.13

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.14]/88, version 365
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.14

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.14

  Path-id 1 not advertised to any peer

Leaf1# sh ip route vrf VRF_A 
IP Route Table for VRF "VRF_A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.0.0.0/24, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 1d07h, direct
10.0.0.11/32, ubest/mbest: 1/0, attached
    *via 10.0.0.11, Vlan10, [190/0], 1d07h, hmm
10.0.0.13/32, ubest/mbest: 1/0
    *via 172.16.0.13%default, [200/0], 19:20:25, bgp-65501, internal, tag 65501,
 segid: 770000 tunnelid: 0xac10000d encap: VXLAN
 
10.0.0.254/32, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 1d07h, local
20.0.0.0/24, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 1w0d, direct
20.0.0.12/32, ubest/mbest: 1/0, attached
    *via 20.0.0.12, Vlan20, [190/0], 1d06h, hmm
20.0.0.254/32, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 1w0d, local
    
Leaf1# sh nve interface nve 1 detail 
Interface: nve1, State: Up, encapsulation: VXLAN
 VPC Capability: VPC-VIP-Only [notified]
 Local Router MAC: 5002.8600.1b08
 Host Learning Mode: Control-Plane
 Source-Interface: loopback0 (primary: 172.16.0.11, secondary: 172.16.0.100)
 Source Interface State: Up
 Virtual RMAC Advertisement: No
 NVE Flags: 
 Interface Handle: 0x49000001
 Source Interface hold-down-time: 180
 Source Interface hold-up-time: 30
 Remaining hold-down time: 0 seconds
 Virtual Router MAC: 0200.ac10.0064
 Interface state: nve-intf-add-complete

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
1     Po1    up     10,20,77,3900                                               
         

vPC status
----------------------------------------------------------------------------
Id    Port          Status Consistency Reason                Active vlans
--    ------------  ------ ----------- ------                ---------------
11    Po10          up     success     success               10                 
         
                                                                                
         
12    Po20          up     success     success               20                 

Leaf1# sh vpc consistency-parameters interface port-channel 1 
Note: **** Global type-1 parameters will be displayed for peer-link *****
    Legend:
        Type 1 : vPC will be suspended in case of mismatch

Name                        Type  Local Value            Peer Value             
-------------               ----  ---------------------- -----------------------
STP MST Simulate PVST       1     Enabled                Enabled               
STP Port Type, Edge         1     Normal, Disabled,      Normal, Disabled,     
BPDUFilter, Edge BPDUGuard        Disabled               Disabled              
STP MST Region Name         1     ""                     ""                    
STP Disabled                1     None                   None                  
STP Mode                    1     Rapid-PVST             Rapid-PVST            
STP Bridge Assurance        1     Enabled                Enabled               
STP Loopguard               1     Disabled               Disabled              
STP MST Region Instance to  1                                                  
 VLAN Mapping                                                                  
STP MST Region Revision     1     0                      0                     
Interface-vlan admin up     2     10,20,77,3900          10,20,77,3900         
Interface-vlan routing      2     10,20,77,3900          10,20,77,3900         
capability                                                                     
Nve1 Adm St, Src Adm St,    1     Up, Up, 172.16.0.100,  Up, Up, 172.16.0.100, 
Sec IP, Host Reach, VMAC          CP, FALSE, Disabled,   CP, FALSE, Disabled,  
Adv, SA,mcast l2, mcast           0.0.0.0, 0.0.0.0,      0.0.0.0, 0.0.0.0,     
l3, IR BGP,MS Adm St, Reo         Enabled, Down, 0.0.0.0 Enabled, Down, 0.0.0.0
Nve1 Vni, Mcast, Mode,      1     770000, 0.0.0.0, n/a,  770000, 0.0.0.0, n/a, 
Type, Flags                       L3, L3VNI              L3, L3VNI             
Nve1 Vni, Mcast, Mode,      1     10000, 0.0.0.0, Mcast, 10000, 0.0.0.0, Mcast,
Type, Flags                        L2, None               L2, None             
Nve1 Vni, Mcast, Mode,      1     20000, 0.0.0.0, Mcast, 20000, 0.0.0.0, Mcast,
Type, Flags                        L2, None               L2, None             
Xconnect Vlans              1                                                  
QoS (Cos)                   2     ([0-7], [], [], [],    ([0-7], [], [], [],   
                                  [], [], [], [])        [], [], [], [])       
Network QoS (MTU)           2     (1500, 1500, 1500,     (1500, 1500, 1500,    
                                  1500, 0, 0, 0, 0)      1500, 0, 0, 0, 0)     
Network Qos (Pause:         2     (F, F, F, F, F, F, F,  (F, F, F, F, F, F, F, 
T->Enabled, F->Disabled)          F)                     F)                    
Input Queuing (Bandwidth)   2     (0, 0, 0, 0, 0, 0, 0,  (0, 0, 0, 0, 0, 0, 0, 
                                  0)                     0)                    
Input Queuing (Absolute     2     (F, F, F, F, F, F, F,  (F, F, F, F, F, F, F, 
Priority: T->Enabled,             F)                     F)                    
F->Disabled)                                                                   
Output Queuing (Bandwidth   2     (100, 0, 0, 0, 0, 0,   (100, 0, 0, 0, 0, 0,  
Remaining)                        0, 0)                  0, 0)                 
Output Queuing (Absolute    2     (F, F, F, T, F, F, F,  (F, F, F, T, F, F, F, 
Priority: T->Enabled,             F)                     F)                    
F->Disabled)                                                                   
Allowed VLANs               -     10,20,77,3900          10,20,77,3900         
Local suspended VLANs       -     -                      -                     

Leaf1# sh vpc consistency-parameters interface port-channel 10

    Legend:
        Type 1 : vPC will be suspended in case of mismatch

Name                        Type  Local Value            Peer Value             
-------------               ----  ---------------------- -----------------------
delayed-lacp                1     disabled               disabled              
mode                        1     active                 active                
Switchport Isolated         1     0                      0                     
Interface type              1     port-channel           port-channel          
LACP Mode                   1     on                     on                    
Virtual-ethernet-bridge     1     Disabled               Disabled              
Speed                       1     1000 Mb/s              1000 Mb/s             
Duplex                      1     full                   full                  
MTU                         1     1500                   1500                  
Port Mode                   1     access                 access                
Native Vlan                 1     1                      1                     
Admin port mode             1     access                 access                
STP Port Guard              1     Default                Default               
STP Port Type               1     Default                Default               
STP MST Simulate PVST       1     Default                Default               
lag-id                      1     [(7f9b,                [(7f9b,               
                                  0-23-4-ee-be-a, 800b,  0-23-4-ee-be-a, 800b, 
                                  0, 0), (8000,          0, 0), (8000,         
                                  0-1e-14-b3-d3-0, 1, 0, 0-1e-14-b3-d3-0, 1, 0,
                                   0)]                    0)]                  
Allow-Multi-Tag             1     Disabled               Disabled              
Vlan xlt mapping            1     Disabled               Disabled              
vPC card type               1     N9K EOR LC             N9K EOR LC            
Allowed VLANs               -     10                     10                    
Local suspended VLANs       -     -                      -                     
Leaf1# sh vpc consistency-parameters interface port-channel 20

    Legend:
        Type 1 : vPC will be suspended in case of mismatch

Name                        Type  Local Value            Peer Value             
-------------               ----  ---------------------- -----------------------
delayed-lacp                1     disabled               disabled              
mode                        1     active                 active                
Switchport Isolated         1     0                      0                     
Interface type              1     port-channel           port-channel          
LACP Mode                   1     on                     on                    
Virtual-ethernet-bridge     1     Disabled               Disabled              
Speed                       1     1000 Mb/s              1000 Mb/s             
Duplex                      1     full                   full                  
MTU                         1     1500                   1500                  
Port Mode                   1     access                 access                
Native Vlan                 1     1                      1                     
Admin port mode             1     access                 access                
STP Port Guard              1     Default                Default               
STP Port Type               1     Default                Default               
STP MST Simulate PVST       1     Default                Default               
lag-id                      1     [(7f9b,                [(7f9b,               
                                  0-23-4-ee-be-a, 800c,  0-23-4-ee-be-a, 800c, 
                                  0, 0), (8000,          0, 0), (8000,         
                                  0-1e-bd-57-12-0, 1, 0, 0-1e-bd-57-12-0, 1, 0,
                                   0)]                    0)]                  
Allow-Multi-Tag             1     Disabled               Disabled              
Vlan xlt mapping            1     Disabled               Disabled              
vPC card type               1     N9K EOR LC             N9K EOR LC            
Allowed VLANs               -     20                     20                    
Local suspended VLANs       -     -                      -

Leaf1# sh mac address-table 
Legend: 
* - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
age - seconds since last seen,+ - primary entry using vPC Peer-Link,
(T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
+   10     0000.0000.0011   dynamic  0         F      F    Po10
C   10     0000.0000.0013   dynamic  0         F      F    nve1(172.16.0.13)
+   20     0000.0000.0012   dynamic  0         F      F    Po20
*   77     5002.8500.1b08   static   -         F      F    nve1(172.16.0.14)
*   77     5002.8600.1b08   static   -         F      F    Vlan77
*   77     5002.8900.1b08   static   -         F      F    nve1(172.16.0.13)
G    -     0001.0001.0001   static   -         F      F    sup-eth1(R)
G   10     5002.8400.1b08   static   -         F      F    vPC Peer-Link(R)
G   20     5002.8400.1b08   static   -         F      F    vPC Peer-Link(R)
G 3900     5002.8400.1b08   static   -         F      F    vPC Peer-Link(R)
G   77     5002.8400.1b08   static   -         F      F    vPC Peer-Link(R)
G    -     5002.8600.1b08   static   -         F      F    sup-eth1(R)
G   10     5002.8600.1b08   static   -         F      F    sup-eth1(R)
G   20     5002.8600.1b08   static   -         F      F    sup-eth1(R)
G 3900     5002.8600.1b08   static   -         F      F    sup-eth1(R)
G   77     5002.8600.1b08   static   -         F      F    sup-eth1(R)         
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
    *via 172.16.10.3, [200/0], 1d06h, bgp-65501, internal, tag 65501
    *via 172.16.10.9, [200/0], 1d06h, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 2/0, attached
    *via 172.16.0.12, Lo0, [0/0], 1d08h, local
    *via 172.16.0.12, Lo0, [0/0], 1d08h, direct
172.16.0.13/32, ubest/mbest: 2/0
    *via 172.16.10.3, [200/0], 20:07:59, bgp-65501, internal, tag 65501
    *via 172.16.10.9, [200/0], 20:07:59, bgp-65501, internal, tag 65501
172.16.0.14/32, ubest/mbest: 2/0
    *via 172.16.10.3, [200/0], 20:10:43, bgp-65501, internal, tag 65501
    *via 172.16.10.9, [200/0], 20:10:54, bgp-65501, internal, tag 65501
172.16.0.100/32, ubest/mbest: 2/0, attached
    *via 172.16.0.100, Lo0, [0/0], 1d08h, local
    *via 172.16.0.100, Lo0, [0/0], 1d08h, direct
172.16.0.111/32, ubest/mbest: 1/0
    *via 172.16.10.3, [200/0], 2w1d, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 1/0
    *via 172.16.10.9, [200/0], 1d06h, bgp-65501, internal, tag 65501
172.16.10.2/31, ubest/mbest: 1/0, attached
    *via 172.16.10.2, Eth1/6, [0/0], 2w4d, direct
172.16.10.2/32, ubest/mbest: 1/0, attached
    *via 172.16.10.2, Eth1/6, [0/0], 2w4d, local
172.16.10.8/31, ubest/mbest: 1/0, attached
    *via 172.16.10.8, Eth1/7, [0/0], 2w4d, direct
172.16.10.8/32, ubest/mbest: 1/0, attached
    *via 172.16.10.8, Eth1/7, [0/0], 2w4d, local
172.16.39.0/31, ubest/mbest: 1/0, attached
    *via 172.16.39.1, Vlan3900, [0/0], 1d08h, direct
172.16.39.1/32, ubest/mbest: 1/0, attached
    *via 172.16.39.1, Vlan3900, [0/0], 1d08h, local

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.12, local AS number 65501
BGP table version is 129, IPv4 Unicast config peers 2, capable peers 2
7 network entries and 12 paths using 2308 bytes of memory
BGP attribute entries [8/1376], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [6/24]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.3     4 65501  531093  530940      129    0    0     2w1d 5         
172.16.10.9     4 65501  145498  145331      129    0    0    1d06h 5
172.16.39.0     4 65501     185     138       42    0    0 05:51:33 6        
Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp ipv4 unicast
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 129, Local Router ID is 172.16.0.12
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

   Network            Next Hop            Metric     LocPrf     Weight Path
*|i172.16.0.11/32     172.16.10.9              0        100          0 ?
*>i                   172.16.10.3              0        100          0 ?
*>r172.16.0.12/32     0.0.0.0                  0        100      32768 ?
*|i172.16.0.13/32     172.16.10.9              0        100          0 i
*>i                   172.16.10.3              0        100          0 i
*>i172.16.0.14/32     172.16.10.3              0        100          0 ?
*|i                   172.16.10.9              0        100          0 ?
*>r172.16.0.100/32    0.0.0.0                  0        100      32768 ?
* i                   172.16.10.9              0        100          0 ?
* i                   172.16.10.3              0        100          0 ?
*>i172.16.0.111/32    172.16.10.3              0        100          0 ?
*>i172.16.0.112/32    172.16.10.9              0        100          0 ?

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 447, Local Router ID is 172.16.0.12
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
* i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.14                       100          0 i
*>i                   172.16.0.13                       100          0 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.14                       100          0 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100      32768 i
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.14                       100          0 i
*>i                   172.16.0.13                       100          0 i
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
*>l[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100      32768 i

Route Distinguisher: 172.16.0.12:65534    (L2VNI 0)
* i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.14                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:40
* i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:32777
* i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i
* i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.14:40
* i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:32777
* i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.12:3    (L3VNI 770000)
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.14                       100          0 i
*>i                   172.16.0.13                       100          0 i

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 445
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
    172.16.10.3        172.16.10.9    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 125
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
    172.16.10.3        172.16.10.9    

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.12:32787    (L2VNI 20000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216, version 447
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
    172.16.10.3        172.16.10.9    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 127
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08

  Path-id 1 advertised to peers:
    172.16.10.3        172.16.10.9    

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp l2vpn evpn 0000.0000.0013
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216, version 373
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.14:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]
/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 358
Paths: (2 available, best #2)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.14:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.13:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 359
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 
      ESI: 0312.3412.3412.3400.000a

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 3 destination(s)
             Imported paths list: VRF_A L2-10000 L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216, version 372
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 353
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 
      ESI: 0312.3412.3412.3400.000a

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 3 destination(s)
             Imported paths list: VRF_A L2-10000 L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 357
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.14:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp l2vpn evpn route-type 1
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 348
Paths: (2 available, best #2)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.14:32777:[1]:[0312.3412.3412.3400.000a]:[0x0]/152 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.13:32777:[1]:[0312.3412.3412.3400.000a]:[0x0]/152 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:65534    (L2VNI 0)
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 345
Paths: (2 available, best #2)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.14:40:[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.13:40:[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:40
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 328
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: ead_es_global_ctx
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 329
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:40
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 344
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: ead_es_global_ctx
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 346
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show bgp l2vpn evpn route-type 3
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
BGP routing table entry for [3]:[0]:[32]:[172.16.0.13]/88, version 322
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:32777:[3]:[0]:[32]:[172.16.0.13]/88 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.13

  Path-id 1 not advertised to any peer
BGP routing table entry for [3]:[0]:[32]:[172.16.0.14]/88, version 320
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.14:32777:[3]:[0]:[32]:[172.16.0.14]/88 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.14

  Path-id 1 not advertised to any peer
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 124
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 32768
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path-id 1 advertised to peers:
    172.16.10.3        172.16.10.9    

Route Distinguisher: 172.16.0.12:32787    (L2VNI 20000)
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 129
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.100 (metric 0) from 0.0.0.0 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 32768
      Extcommunity: RT:65501:20000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 20000, Tunnel Id: 172.16.0.100

  Path-id 1 advertised to peers:
    172.16.10.3        172.16.10.9    

Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.13]/88, version 323
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.13

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.13

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.14]/88, version 319
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.14

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.14

  Path-id 1 not advertised to any peer

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show ip route vrf vRF_A
IP Route Table for VRF "VRF_A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.0.0.0/24, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 1w0d, direct
10.0.0.11/32, ubest/mbest: 1/0, attached
    *via 10.0.0.11, Vlan10, [190/0], 1d07h, hmm
10.0.0.13/32, ubest/mbest: 1/0
    *via 172.16.0.13%default, [200/0], 19:33:41, bgp-65501, internal, tag 65501, segid: 7700
00 tunnelid: 0xac10000d encap: VXLAN
 
10.0.0.254/32, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 1w0d, local
20.0.0.0/24, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 1d07h, direct
20.0.0.12/32, ubest/mbest: 1/0, attached
    *via 20.0.0.12, Vlan20, [190/0], 1d07h, hmm
20.0.0.254/32, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 1d07h, local

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# show nve interface nve 1 detail
Interface: nve1, State: Up, encapsulation: VXLAN
 VPC Capability: VPC-VIP-Only [notified]
 Local Router MAC: 5002.8400.1b08
 Host Learning Mode: Control-Plane
 Source-Interface: loopback0 (primary: 172.16.0.12, secondary: 172.16.0.100)
 Source Interface State: Up
 Virtual RMAC Advertisement: No
 NVE Flags: 
 Interface Handle: 0x49000001
 Source Interface hold-down-time: 180
 Source Interface hold-up-time: 30
 Remaining hold-down time: 0 seconds
 Virtual Router MAC: 0200.ac10.0064
 Interface state: nve-intf-add-complete

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# sh vpc
Legend:
                (*) - local vPC is down, forwarding via vPC peer-link

vPC domain id                     : 10  
Peer status                       : peer adjacency formed ok      
vPC keep-alive status             : peer is alive                 
Configuration consistency status  : success 
Per-vlan consistency status       : success                       
Type-2 consistency status         : success 
vPC role                          : primary                       
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
1     Po1    up     10,20,77,3900                                                        

vPC status
----------------------------------------------------------------------------
Id    Port          Status Consistency Reason                Active vlans
--    ------------  ------ ----------- ------                ---------------
11    Po10          up     success     success               10                          
                                                                                         
12    Po20          up     success     success               20                          
                                                                                         

Please check "show vpc consistency-parameters vpc <vpc-num>" for the 
consistency reason of down vpc and for type-2 consistency reasons for 
any vpc.

Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# sh vpc consistency-parameters interface port-channel 1 
Note: **** Global type-1 parameters will be displayed for peer-link *****
    Legend:
        Type 1 : vPC will be suspended in case of mismatch

Name                        Type  Local Value            Peer Value             
-------------               ----  ---------------------- -----------------------
STP MST Simulate PVST       1     Enabled                Enabled               
STP Port Type, Edge         1     Normal, Disabled,      Normal, Disabled,     
BPDUFilter, Edge BPDUGuard        Disabled               Disabled              
STP MST Region Name         1     ""                     ""                    
STP Disabled                1     None                   None                  
STP Mode                    1     Rapid-PVST             Rapid-PVST            
STP Bridge Assurance        1     Enabled                Enabled               
STP Loopguard               1     Disabled               Disabled              
STP MST Region Instance to  1                                                  
 VLAN Mapping                                                                  
STP MST Region Revision     1     0                      0                     
Interface-vlan admin up     2     10,20,77,3900          10,20,77,3900         
Interface-vlan routing      2     10,20,77,3900          10,20,77,3900         
capability                                                                     
Nve1 Adm St, Src Adm St,    1     Up, Up, 172.16.0.100,  Up, Up, 172.16.0.100, 
Sec IP, Host Reach, VMAC          CP, FALSE, Disabled,   CP, FALSE, Disabled,  
Adv, SA,mcast l2, mcast           0.0.0.0, 0.0.0.0,      0.0.0.0, 0.0.0.0,     
l3, IR BGP,MS Adm St, Reo         Enabled, Down, 0.0.0.0 Enabled, Down, 0.0.0.0
Nve1 Vni, Mcast, Mode,      1     770000, 0.0.0.0, n/a,  770000, 0.0.0.0, n/a, 
Type, Flags                       L3, L3VNI              L3, L3VNI             
Nve1 Vni, Mcast, Mode,      1     10000, 0.0.0.0, Mcast, 10000, 0.0.0.0, Mcast,
Type, Flags                        L2, None               L2, None             
Nve1 Vni, Mcast, Mode,      1     20000, 0.0.0.0, Mcast, 20000, 0.0.0.0, Mcast,
Type, Flags                        L2, None               L2, None             
Xconnect Vlans              1                                                  
QoS (Cos)                   2     ([0-7], [], [], [],    ([0-7], [], [], [],   
                                  [], [], [], [])        [], [], [], [])       
Network QoS (MTU)           2     (1500, 1500, 1500,     (1500, 1500, 1500,    
                                  1500, 0, 0, 0, 0)      1500, 0, 0, 0, 0)     
Network Qos (Pause:         2     (F, F, F, F, F, F, F,  (F, F, F, F, F, F, F, 
T->Enabled, F->Disabled)          F)                     F)                    
Input Queuing (Bandwidth)   2     (0, 0, 0, 0, 0, 0, 0,  (0, 0, 0, 0, 0, 0, 0, 
                                  0)                     0)                    
Input Queuing (Absolute     2     (F, F, F, F, F, F, F,  (F, F, F, F, F, F, F, 
Priority: T->Enabled,             F)                     F)                    
F->Disabled)                                                                   
Output Queuing (Bandwidth   2     (100, 0, 0, 0, 0, 0,   (100, 0, 0, 0, 0, 0,  
Remaining)                        0, 0)                  0, 0)                 
Output Queuing (Absolute    2     (F, F, F, T, F, F, F,  (F, F, F, T, F, F, F, 
Priority: T->Enabled,             F)                     F)                    
F->Disabled)                                                                   
Allowed VLANs               -     10,20,77,3900          10,20,77,3900         
Local suspended VLANs       -     -                      -                     
Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# sh vpc consistency-parameters interface port-channel 10

    Legend:
        Type 1 : vPC will be suspended in case of mismatch

Name                        Type  Local Value            Peer Value             
-------------               ----  ---------------------- -----------------------
delayed-lacp                1     disabled               disabled              
mode                        1     active                 active                
Switchport Isolated         1     0                      0                     
Interface type              1     port-channel           port-channel          
LACP Mode                   1     on                     on                    
Virtual-ethernet-bridge     1     Disabled               Disabled              
Speed                       1     1000 Mb/s              1000 Mb/s             
Duplex                      1     full                   full                  
MTU                         1     1500                   1500                  
Port Mode                   1     access                 access                
Native Vlan                 1     1                      1                     
Admin port mode             1     access                 access                
STP Port Guard              1     Default                Default               
STP Port Type               1     Default                Default               
STP MST Simulate PVST       1     Default                Default               
lag-id                      1     [(7f9b,                [(7f9b,               
                                  0-23-4-ee-be-a, 800b,  0-23-4-ee-be-a, 800b, 
                                  0, 0), (8000,          0, 0), (8000,         
                                  0-1e-14-b3-d3-0, 1, 0, 0-1e-14-b3-d3-0, 1, 0,
                                   0)]                    0)]                  
Allow-Multi-Tag             1     Disabled               Disabled              
Vlan xlt mapping            1     Disabled               Disabled              
vPC card type               1     N9K EOR LC             N9K EOR LC            
Allowed VLANs               -     10                     10                    
Local suspended VLANs       -     -                      -                     
Leaf2# 
Leaf2# !
Leaf2# 
Leaf2# sh vpc consistency-parameters interface port-channel 20

    Legend:
        Type 1 : vPC will be suspended in case of mismatch

Name                        Type  Local Value            Peer Value             
-------------               ----  ---------------------- -----------------------
delayed-lacp                1     disabled               disabled              
mode                        1     active                 active                
Switchport Isolated         1     0                      0                     
Interface type              1     port-channel           port-channel          
LACP Mode                   1     on                     on                    
Virtual-ethernet-bridge     1     Disabled               Disabled              
Speed                       1     1000 Mb/s              1000 Mb/s             
Duplex                      1     full                   full                  
MTU                         1     1500                   1500                  
Port Mode                   1     access                 access                
Native Vlan                 1     1                      1                     
Admin port mode             1     access                 access                
STP Port Guard              1     Default                Default               
STP Port Type               1     Default                Default               
STP MST Simulate PVST       1     Default                Default               
lag-id                      1     [(7f9b,                [(7f9b,               
                                  0-23-4-ee-be-a, 800c,  0-23-4-ee-be-a, 800c, 
                                  0, 0), (8000,          0, 0), (8000,         
                                  0-1e-bd-57-12-0, 1, 0, 0-1e-bd-57-12-0, 1, 0,
                                   0)]                    0)]                  
Allow-Multi-Tag             1     Disabled               Disabled              
Vlan xlt mapping            1     Disabled               Disabled              
vPC card type               1     N9K EOR LC             N9K EOR LC            
Allowed VLANs               -     20                     20                    
Local suspended VLANs       -     -                      -

Leaf2# sh mac address-table 
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
*   10     0000.0000.0011   dynamic  0         F      F    Po10
C   10     0000.0000.0013   dynamic  0         F      F    nve1(172.16.0.13)
*   20     0000.0000.0012   dynamic  0         F      F    Po20
*   77     5002.8400.1b08   static   -         F      F    Vlan77
*   77     5002.8500.1b08   static   -         F      F    nve1(172.16.0.14)
*   77     5002.8900.1b08   static   -         F      F    nve1(172.16.0.13)
G    -     0001.0001.0001   static   -         F      F    sup-eth1(R)
G    -     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G   10     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G   20     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G 3900     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G   77     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G   10     5002.8600.1b08   static   -         F      F    vPC Peer-Link(R)
G   20     5002.8600.1b08   static   -         F      F    vPC Peer-Link(R)
G 3900     5002.8600.1b08   static   -         F      F    vPC Peer-Link(R)
G   77     5002.8600.1b08   static   -         F      F    vPC Peer-Link(R)
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
    *via 172.16.10.5, [200/0], 20:10:08, bgp-65501, internal, tag 65501
    *via 172.16.10.11, [200/0], 20:10:08, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 2/0
    *via 172.16.10.5, [200/0], 20:10:08, bgp-65501, internal, tag 65501
    *via 172.16.10.11, [200/0], 20:10:08, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 2/0, attached
    *via 172.16.0.13, Lo0, [0/0], 20:13:10, local
    *via 172.16.0.13, Lo0, [0/0], 20:13:10, direct
172.16.0.14/32, ubest/mbest: 2/0
    *via 172.16.10.5, [200/0], 20:10:08, bgp-65501, internal, tag 65501
    *via 172.16.10.11, [200/0], 20:10:08, bgp-65501, internal, tag 65501
172.16.0.100/32, ubest/mbest: 2/0
    *via 172.16.10.5, [200/0], 20:10:08, bgp-65501, internal, tag 65501
    *via 172.16.10.11, [200/0], 20:10:08, bgp-65501, internal, tag 65501
172.16.0.111/32, ubest/mbest: 1/0
    *via 172.16.10.5, [200/0], 20:10:08, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 1/0
    *via 172.16.10.11, [200/0], 20:10:08, bgp-65501, internal, tag 65501
172.16.10.4/31, ubest/mbest: 1/0, attached
    *via 172.16.10.4, Eth1/6, [0/0], 20:11:15, direct
172.16.10.4/32, ubest/mbest: 1/0, attached
    *via 172.16.10.4, Eth1/6, [0/0], 20:11:15, local
172.16.10.10/31, ubest/mbest: 1/0, attached
    *via 172.16.10.10, Eth1/7, [0/0], 20:11:15, direct
172.16.10.10/32, ubest/mbest: 1/0, attached
    *via 172.16.10.10, Eth1/7, [0/0], 20:11:15, local

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.13, local AS number 65501
BGP table version is 9, IPv4 Unicast config peers 2, capable peers 2
7 network entries and 11 paths using 2188 bytes of memory
BGP attribute entries [8/1376], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [6/24]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.5     4 65501   24268   24130        9    0    0 20:10:37 5         
172.16.10.11    4 65501   24258   24121        9    0    0 20:10:08 5         
Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp ipv4 unicast
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 9, Local Router ID is 172.16.0.13
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

   Network            Next Hop            Metric     LocPrf     Weight Path
*|i172.16.0.11/32     172.16.10.11             0        100          0 ?
*>i                   172.16.10.5              0        100          0 ?
*|i172.16.0.12/32     172.16.10.11             0        100          0 ?
*>i                   172.16.10.5              0        100          0 ?
*>r172.16.0.13/32     0.0.0.0                  0        100      32768 i
*|i172.16.0.14/32     172.16.10.11             0        100          0 ?
*>i                   172.16.10.5              0        100          0 ?
*|i172.16.0.100/32    172.16.10.11             0        100          0 ?
*>i                   172.16.10.5              0        100          0 ?
*>i172.16.0.111/32    172.16.10.5              0        100          0 ?
*>i172.16.0.112/32    172.16.10.11             0        100          0 ?

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 307, Local Router ID is 172.16.0.13
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:32777
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.11:32787
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i

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
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.13:32777    (L2VNI 10000)
* i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.14                       100          0 i
*>l                   172.16.0.13                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.14                       100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.14                       100          0 i
*>l                   172.16.0.13                       100      32768 i
*>l[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100      32768 i
*>i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.13:65534    (L2VNI 0)
*>i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:40
* i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:27009
* i[4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.14]/136
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.14:32777
* i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.14                       100          0 i
* i                   172.16.0.14                       100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i
* i[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100          0 i
*>i                   172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.13:27009   (ES [0312.3412.3412.3400.000a 0])
*>l[4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.13]/136
                      172.16.0.13                       100      32768 i
*>i[4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.14]/136
                      172.16.0.14                       100          0 i

Route Distinguisher: 172.16.0.13:40   (EAD-ES [0312.3412.3412.3400.000a 40])
*>l[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.13                       100      32768 i

Route Distinguisher: 172.16.0.13:3    (L3VNI 770000)
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.14                       100          0 i

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 306
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
15
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 3 destination(s)
             Imported paths list: VRF_A L2-10000 L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 303
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
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 16
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
             Imported to 3 destination(s)
             Imported paths list: VRF_A L2-10000 L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 307
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
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 24
Paths: (2 available, best #2)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
23
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 17
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_A L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 18
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_A L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 25
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp l2vpn evpn 0000.0000.0013
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216, version 74
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, mh_peer_synced, no labeled nexthop, in rib
             Imported from 172.16.0.14:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 55
Paths: (2 available, best #2)
Flags: (0x000312) (high32 00000000) on xmit-list, is in l2rib/evpn

  Path type: internal, path is valid, not best reason: Local ESI, mh_peer_synced, no labeled
 nexthop, in rib
             Imported from 172.16.0.14:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.13 (metric 0) from 0.0.0.0 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 advertised to peers:
    172.16.10.5        172.16.10.11   

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216, version 73
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 52
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 
      ESI: 0312.3412.3412.3400.000a

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 3 destination(s)
             Imported paths list: VRF_A L2-10000 L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 53
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.14:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp l2vpn evpn route-type 1
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:32777    (L2VNI 10000)
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 47
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Path type: internal, path is valid, not best reason: Weight, no labeled nexthop
             Imported from 172.16.0.14:32777:[1]:[0312.3412.3412.3400.000a]:[0x0]/152 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.13 (metric 0) from 0.0.0.0 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.5        172.16.10.11   

Route Distinguisher: 172.16.0.13:65534    (L2VNI 0)
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 44
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.14:40:[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:40
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 43
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: ead_es_global_ctx
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 45
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:40   (EAD-ES [0312.3412.3412.3400.000a 40])
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 29
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.13 (metric 0) from 0.0.0.0 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000

  Path-id 1 advertised to peers:
    172.16.10.5        172.16.10.11   

Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show bgp l2vpn evpn route-type 3
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 13
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 14
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:32777    (L2VNI 10000)
BGP routing table entry for [3]:[0]:[32]:[172.16.0.13]/88, version 2
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.13 (metric 0) from 0.0.0.0 (172.16.0.13)
      Origin IGP, MED not set, localpref 100, weight 32768
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.13

  Path-id 1 advertised to peers:
    172.16.10.5        172.16.10.11   
BGP routing table entry for [3]:[0]:[32]:[172.16.0.14]/88, version 27
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.14:32777:[3]:[0]:[32]:[172.16.0.14]/88 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.14

  Path-id 1 not advertised to any peer
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 22
Paths: (2 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.12:32777:[3]:[0]:[32]:[172.16.0.100]/88 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
             Imported from 172.16.0.11:32777:[3]:[0]:[32]:[172.16.0.100]/88 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.14]/88, version 21
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.112 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.14

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.14 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.14 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.14

  Path-id 1 not advertised to any peer
Leaf3# 
Leaf3# show ip route vrf vRF_A
IP Route Table for VRF "VRF_A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.0.0.0/24, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 20:13:11, direct
10.0.0.11/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 20:10:06, bgp-65501, internal, tag 65501, segid: 770
000 tunnelid: 0xac100064 encap: VXLAN
 
10.0.0.13/32, ubest/mbest: 1/0, attached
    *via 10.0.0.13, Vlan10, [190/0], 19:35:50, hmm
10.0.0.254/32, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 20:13:11, local
20.0.0.12/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 20:10:06, bgp-65501, internal, tag 65501, segid: 770
000 tunnelid: 0xac100064 encap: VXLAN
Leaf3# 
Leaf3# !
Leaf3# 
Leaf3# show nve interface nve 1 detail
Interface: nve1, State: Up, encapsulation: VXLAN
 VPC Capability: VPC-VIP-Only [not-notified]
 Local Router MAC: 5002.8900.1b08
 Host Learning Mode: Control-Plane
 Source-Interface: loopback0 (primary: 172.16.0.13, secondary: 0.0.0.0)
 Source Interface State: Up
 Virtual RMAC Advertisement: No
 NVE Flags: 
 Interface Handle: 0x49000001
 Source Interface hold-down-time: 180
 Source Interface hold-up-time: 30
 Remaining hold-down time: 0 seconds
 Virtual Router MAC: N/A
 Interface state: nve-intf-add-complete
 ESI multihoming delay-restore time: 30 seconds
 ESI multihoming delay-restore time left: 0 seconds

Leaf3# show nve ethernet-segment summary 
ESI                              Parent interface   ES State  
------------------------------   ------------------ ----------
0312.3412.3412.3400.000a         port-channel10     Up

Leaf3# show nve ethernet-segment

ESI: 0312.3412.3412.3400.000a
   Parent interface: port-channel10
  ES State: Up 
  Port-channel state: Up
  NVE Interface: nve1 
   NVE State: Up 
   Host Learning Mode: control-plane
  Active Vlans: 10 
   DF Vlans: 10 
   Active VNIs: 10000 
  CC failed for VLANs:  
  VLAN CC timer: no-timer 
  Number of ES members: 2 
  My ordinal: 0 
  DF timer start time: 00:00:00 
  Config State: config-applied 
  DF List: 172.16.0.13 172.16.0.14  
  ES route added to L2RIB: True
  EAD/ES routes added to L2RIB: True
  EAD/EVI route timer age: not running 

Leaf3# sh mac address-table 
Legend: 
* - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
age - seconds since last seen,+ - primary entry using vPC Peer-Link,
(T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
C   10     0000.0000.0011   dynamic  0         F      F    nve1(172.16.0.100)
*   10     0000.0000.0013   dynamic  0         F      F    Po10
*   77     5002.8500.1b08   static   -         F      F    nve1(172.16.0.14)
*   77     5002.8600.1b08   static   -         F      F    nve1(172.16.0.100)
*   77     5002.8900.1b08   static   -         F      F    Vlan77
G    -     0001.0001.0001   static   -         F      F    sup-eth1(R)
G    -     5002.8900.1b08   static   -         F      F    sup-eth1(R)
G   10     5002.8900.1b08   static   -         F      F    sup-eth1(R)
G   77     5002.8900.1b08   static   -         F      F    sup-eth1(R)
```
=================================================================================================
```
Leaf4# show ip route
show bgp lIP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

172.16.0.11/32, ubest/mbest: 2/0
    *via 172.16.10.15, [200/0], 20:20:14, bgp-65501, internal, tag 65501
    *via 172.16.10.17, [200/0], 20:20:24, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 2/0
    *via 172.16.10.15, [200/0], 20:20:14, bgp-65501, internal, tag 65501
    *via 172.16.10.17, [200/0], 20:20:24, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 2/0
    *via 172.16.10.15, [200/0], 20:17:29, bgp-65501, internal, tag 65501
    *via 172.16.10.17, [200/0], 20:17:29, bgp-65501, internal, tag 65501
172.16.0.14/32, ubest/mbest: 2/0, attached
    *via 172.16.0.14, Lo0, [0/0], 20:24:58, local
    *via 172.16.0.14, Lo0, [0/0], 20:24:58, direct
172.16.0.100/32, ubest/mbest: 2/0
    *via 172.16.10.15, [200/0], 20:20:14, bgp-65501, internal, tag 65501
    *via 172.16.10.17, [200/0], 20:20:24, bgp-65501, internal, tag 65501
172.16.0.111/32, ubest/mbest: 1/0
    *via 172.16.10.15, [200/0], 20:20:14, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 1/0
    *via 172.16.10.17, [200/0], 20:20:24, bgp-65501, internal, tag 65501
172.16.10.14/31, ubest/mbest: 1/0, attached
    *via 172.16.10.14, Eth1/6, [0/0], 20:25:40, direct
172.16.10.14/32, ubest/mbest: 1/0, attached
    *via 172.16.10.14, Eth1/6, [0/0], 20:25:40, local
172.16.10.16/31, ubest/mbest: 1/0, attached
    *via 172.16.10.16, Eth1/7, [0/0], 20:25:21, direct
172.16.10.16/32, ubest/mbest: 1/0, attached
    *via 172.16.10.16, Eth1/7, [0/0], 20:25:21, local

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.14, local AS number 65501
BGP table version is 16, IPv4 Unicast config peers 2, capable peers 2
7 network entries and 11 paths using 2188 bytes of memory
BGP attribute entries [8/1376], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [6/24]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.15    4 65501   24461   24325       16    0    0 20:20:14 5         
172.16.10.17    4 65501   24466   24329       16    0    0 20:20:24 5         
Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp ipv4 unicast
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 16, Local Router ID is 172.16.0.14
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>i172.16.0.11/32     172.16.10.15             0        100          0 ?
*|i                   172.16.10.17             0        100          0 ?
*>i172.16.0.12/32     172.16.10.15             0        100          0 ?
*|i                   172.16.10.17             0        100          0 ?
*|i172.16.0.13/32     172.16.10.17             0        100          0 i
*>i                   172.16.10.15             0        100          0 i
*>r172.16.0.14/32     0.0.0.0                  0        100      32768 ?
*>i172.16.0.100/32    172.16.10.15             0        100          0 ?
*|i                   172.16.10.17             0        100          0 ?
*>i172.16.0.111/32    172.16.10.15             0        100          0 ?
*>i172.16.0.112/32    172.16.10.17             0        100          0 ?

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 327, Local Router ID is 172.16.0.14
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.11:32787
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.12:32777
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.12:32787
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.13:40
* i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:27009
* i[4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.13]/136
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.13:32777
* i[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i
* i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>i                   172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.14:32777    (L2VNI 10000)
*>l[1]:[0312.3412.3412.3400.000a]:[0x0]/152
                      172.16.0.14                       100      32768 i
* i                   172.16.0.13                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.14                       100      32768 i
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.14                       100      32768 i
* i                   172.16.0.13                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.13]/88
                      172.16.0.13                       100          0 i
*>l[3]:[0]:[32]:[172.16.0.14]/88
                      172.16.0.14                       100      32768 i
*>i[3]:[0]:[32]:[172.16.0.100]/88
                      172.16.0.100                      100          0 i
* i                   172.16.0.100                      100          0 i

Route Distinguisher: 172.16.0.14:65534    (L2VNI 0)
*>i[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.13                       100          0 i

Route Distinguisher: 172.16.0.14:27009   (ES [0312.3412.3412.3400.000a 0])
*>i[4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.13]/136
                      172.16.0.13                       100          0 i
*>l[4]:[0312.3412.3412.3400.000a]:[32]:[172.16.0.14]/136
                      172.16.0.14                       100      32768 i

Route Distinguisher: 172.16.0.14:40   (EAD-ES [0312.3412.3412.3400.000a 40])
*>l[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152
                      172.16.0.14                       100      32768 i

Route Distinguisher: 172.16.0.14:3    (L3VNI 770000)
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.100                      100          0 i
*>i                   172.16.0.100                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.13                       100          0 i

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 326
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

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

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 
42
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 3 destination(s)
             Imported paths list: VRF_A L2-10000 L3-770000
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
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 323
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
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 44
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 3 destination(s)
             Imported paths list: VRF_A L2-10000 L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216, version 327
Paths: (2 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 50
Paths: (2 available, best #2)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272, version 49
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 43
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_A L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 45
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_A L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272, version 51
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nexthop
             Imported from 172.16.0.12:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8400.1b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 SOO:172.16.0.100:0 ENCAP:8
          Router MAC:5002.8600.1b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp l2vpn evpn 0000.0000.0013
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 80
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 
      ESI: 0312.3412.3412.3400.000a

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 3 destination(s)
             Imported paths list: VRF_A L2-10000 L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216, version 100
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.14 (metric 0) from 0.0.0.0 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 advertised to peers:
    172.16.10.15       172.16.10.17   
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 82
Paths: (2 available, best #1)
Flags: (0x000302) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.14 (metric 0) from 0.0.0.0 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8500.1b08
      ESI: 0312.3412.3412.3400.000a

  Path type: internal, path is valid, not best reason: Local ESI, mh_peer_reoriginated, no l
abeled nexthop
             Imported from 172.16.0.13:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 advertised to peers:
    172.16.10.15       172.16.10.17   

Route Distinguisher: 172.16.0.14:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272, version 81
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 SOO:172.16.0.13:256 ENCAP:8
          Router MAC:5002.8900.1b08
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      ESI: 0312.3412.3412.3400.000a

  Path-id 1 not advertised to any peer

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp l2vpn evpn route-type 1
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.13:40
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 60
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: ead_es_global_ctx
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 61
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777    (L2VNI 10000)
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0x0]/152, version 76
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.14 (metric 0) from 0.0.0.0 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8

  Path type: internal, path is valid, not best reason: Weight, no labeled nexthop, in rib
             Imported from 172.16.0.13:32777:[1]:[0312.3412.3412.3400.000a]:[0x0]/152 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 advertised to peers:
    172.16.10.15       172.16.10.17   

Route Distinguisher: 172.16.0.14:65534    (L2VNI 0)
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 57
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.13:40:[1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:40   (EAD-ES [0312.3412.3412.3400.000a 40])
BGP routing table entry for [1]:[0312.3412.3412.3400.000a]:[0xffffffff]/152, version 75
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.14 (metric 0) from 0.0.0.0 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 0
      Extcommunity: RT:65501:10000 ENCAP:8 ESI:0:000000

  Path-id 1 advertised to peers:
    172.16.10.15       172.16.10.17   

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show bgp l2vpn evpn route-type 3
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 40
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 41
Paths: (2 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.13:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.13]/88, version 55
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.17 (172.16.0.112)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.13

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.13

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.14:32777    (L2VNI 10000)
BGP routing table entry for [3]:[0]:[32]:[172.16.0.13]/88, version 54
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.13:32777:[3]:[0]:[32]:[172.16.0.13]/88 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.13 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.13

  Path-id 1 not advertised to any peer
BGP routing table entry for [3]:[0]:[32]:[172.16.0.14]/88, version 3
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.14 (metric 0) from 0.0.0.0 (172.16.0.14)
      Origin IGP, MED not set, localpref 100, weight 32768
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.14

  Path-id 1 advertised to peers:
    172.16.10.15       172.16.10.17   
BGP routing table entry for [3]:[0]:[32]:[172.16.0.100]/88, version 48
Paths: (2 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.12:32777:[3]:[0]:[32]:[172.16.0.100]/88 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path type: internal, path is valid, not best reason: Neighbor Address, no labeled nexthop
             Imported from 172.16.0.11:32777:[3]:[0]:[32]:[172.16.0.100]/88 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.100 (metric 0) from 172.16.10.15 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.100

  Path-id 1 not advertised to any peer

Leaf4# 
Leaf4# 
Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show ip route vrf vRF_A
IP Route Table for VRF "VRF_A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.0.0.0/24, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 20:26:38, direct
10.0.0.11/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 20:20:12, bgp-65501, internal, tag 65501, segid: 770
000 tunnelid: 0xac100064 encap: VXLAN
 
10.0.0.13/32, ubest/mbest: 1/0, attached
    *via 10.0.0.13, Vlan10, [190/0], 19:49:21, hmm
10.0.0.254/32, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 20:26:38, local
20.0.0.12/32, ubest/mbest: 1/0
    *via 172.16.0.100%default, [200/0], 20:20:12, bgp-65501, internal, tag 65501, segid: 770
000 tunnelid: 0xac100064 encap: VXLAN
 

Leaf4# 
Leaf4# !
Leaf4# 
Leaf4# show nve interface nve 1 detail
Interface: nve1, State: Up, encapsulation: VXLAN
 VPC Capability: VPC-VIP-Only [not-notified]
 Local Router MAC: 5002.8500.1b08
 Host Learning Mode: Control-Plane
 Source-Interface: loopback0 (primary: 172.16.0.14, secondary: 0.0.0.0)
 Source Interface State: Up
 Virtual RMAC Advertisement: No
 NVE Flags: 
 Interface Handle: 0x49000001
 Source Interface hold-down-time: 180
 Source Interface hold-up-time: 30
 Remaining hold-down time: 0 seconds
 Virtual Router MAC: N/A
 Interface state: nve-intf-add-complete
 ESI multihoming delay-restore time: 30 seconds
 ESI multihoming delay-restore time left: 0 seconds

Leaf4# show nve ethernet-segment summary 
ESI                              Parent interface   ES State  
------------------------------   ------------------ ----------
0312.3412.3412.3400.000a         port-channel10     Up         
Leaf4# 
Leaf4# 
Leaf4# 
Leaf4# show nve ethernet-segment

ESI: 0312.3412.3412.3400.000a
   Parent interface: port-channel10
  ES State: Up 
  Port-channel state: Up
  NVE Interface: nve1 
   NVE State: Up 
   Host Learning Mode: control-plane
  Active Vlans: 10 
   DF Vlans:  
   Active VNIs: 10000 
  CC failed for VLANs:  
  VLAN CC timer: no-timer 
  Number of ES members: 2 
  My ordinal: 1 
  DF timer start time: 00:00:00 
  Config State: config-applied 
  DF List: 172.16.0.13 172.16.0.14  
  ES route added to L2RIB: True
  EAD/ES routes added to L2RIB: True
  EAD/EVI route timer age: not running 
----------------------------------------

Leaf4# sh mac address-table 
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
C   10     0000.0000.0011   dynamic  0         F      F    nve1(172.16.0.100)
*   10     0000.0000.0013   dynamic  0         F      F    Po10
*   77     5002.8500.1b08   static   -         F      F    Vlan77
*   77     5002.8600.1b08   static   -         F      F    nve1(172.16.0.100)
*   77     5002.8900.1b08   static   -         F      F    nve1(172.16.0.13)
G    -     0001.0001.0001   static   -         F      F    sup-eth1(R)
G    -     5002.8500.1b08   static   -         F      F    sup-eth1(R)
G   10     5002.8500.1b08   static   -         F      F    sup-eth1(R)
G   77     5002.8500.1b08   static   -         F      F    sup-eth1(R)
```
=============================================================================
```
Client1#ping 10.0.0.13
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.13, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 402/489/573 ms
Client1#ping 20.0.0.12
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 20.0.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 243/355/708 ms
Client1#sh ip aro
Client1#sh ip arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  10.0.0.11               -   0000.0000.0011  ARPA   Port-channel1
Internet  10.0.0.13             235   0000.0000.0013  ARPA   Port-channel1
Internet  10.0.0.254              9   0001.0001.0001  ARPA   Port-channel1

=======================================================================================

Client2#ping 10.0.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.11, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/5/9 ms
Client2#ping 10.0.0.13
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.13, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 8/9/11 ms
Client2#sh ip arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  20.0.0.12               -   0000.0000.0012  ARPA   Port-channel1
Internet  20.0.0.254              8   0001.0001.0001  ARPA   Port-channel1

=======================================================================================

Client3#ping 10.0.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.11, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 5/6/10 ms
Client3#ping 20.0.0.12
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 20.0.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 8/10/12 ms
Client3#sh ip arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  10.0.0.11             242   0000.0000.0011  ARPA   Port-channel1
Internet  10.0.0.13               -   0000.0000.0013  ARPA   Port-channel1
Internet  10.0.0.254             13   0001.0001.0001  ARPA   Port-channel1
```
========================================================================================

 В рамках проверки отказоустойчивости были погашены: на Client1 - Gi2(Leaf2), Client3 - Gi1 (Leaf3). Также для проверки
 маршрутизации через peerlink на Leaf1 погашены Eth1/6, Eth1/7 в сторону Spine1 и Spine2.

```
Client1#sh ip int brief 
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       unassigned      YES unset  up                    up      
GigabitEthernet2       unassigned      YES unset  administratively down down    
GigabitEthernet3       unassigned      YES unset  administratively down down    
GigabitEthernet4       unassigned      YES unset  administratively down down    
Port-channel1          10.0.0.11       YES manual up                    up      
Client1#ping 10.0.0.13
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.13, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/10/15 ms
Client1#ping 20.0.0.12
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 20.0.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms

Client3#sh ip int br
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       unassigned      YES unset  administratively down down    
GigabitEthernet2       unassigned      YES unset  up                    up      
GigabitEthernet3       unassigned      YES unset  administratively down down    
GigabitEthernet4       unassigned      YES unset  administratively down down    
Port-channel1          10.0.0.13       YES manual up                    up      
Client3#ping 10.0.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.11, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 8/11/17 ms
Client3#ping 20.0.0.12
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 20.0.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 7/9/14 ms

```

