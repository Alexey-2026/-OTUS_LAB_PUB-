# Комбинированная модель пересылки BUM-трафика (PIM + Ingress Replication) в архитектуре VXLAN EVPN Multi-Site

### Цель работы
Реализовать комбинированную модель пересылки BUM-трафика (PIM + Ingress Replication) в архитектуре VXLAN EVPN Multi-Site.

<img src="https://github.com/user-attachments/assets/c6936e33-662d-44dd-a4f1-0d13465b4bf5" width="700" style="max-width: 100%; height: auto;">



## План адресации

### Underlay сеть (линки /31)

| Линк | Spine интерфейс | IP Spine | Leaf интерфейс | IP Leaf |
|------|----------------|----------|----------------|---------|
|SuperSpine-Spine1| Eth1/1 | 172.16.111.0 | Eth1/4 | 172.16.111.1 | 
|SuperSpine-Spine2| Eth1/2 | 172.16.112.0 | Eth1/4 | 172.16.112.1 | 
|SuperSpine-Spine3| Eth1/3 | 172.16.113.0 | Eth1/4 | 172.16.113.1 | 
| Spine1-Leaf1 | Eth1/1 | 172.16.11.1 | Eth1/1 | 172.16.11.0 |
| Spine1-Leaf2 | Eth1/2 | 172.16.21.1 | Eth1/1 | 172.16.21.0 |
| Spine1-Leaf3 | Eth1/3 | 172.16.31.1 | Eth1/1 | 172.16.31.0 |
| Spine2-Leaf1 | Eth1/1 | 172.16.12.1 | Eth1/2 | 172.16.12.0 |
| Spine2-Leaf2 | Eth1/2 | 172.16.22.1 | Eth1/2 | 172.16.22.0 |
| Spine2-Leaf3 | Eth1/3 | 172.16.32.1 | Eth1/2 | 172.16.32.0 |
| Spine3-Leaf21 | Eth1/1 | 172.16.211.1 | Eth1/1 | 172.16.211.0 |



### Loopback адреса

| Устройство | Loopback0 IP |
|------------|--------------|
| SuperSpine | 172.16.0.254/32 | 
| Spine-1 | 172.16.0.111/32 |
| Spine-2 | 172.16.0.112/32 |
| Spine-3 | 172.16.0.113/32 |
| Leaf-1 | 172.16.0.11/32 |
| Leaf-2 | 172.16.0.12/32 |
| Leaf-3 | 172.16.0.13/32 |
| Leaf-21 | 172.16.0.21/32 |


### Серверные подсети

| Leaf | VLAN | Подсеть | Шлюз |
|------|------|---------|------|
| Leaf-1 | 10 | 10.0.0.0/24 | 10.0.0.254 |
| Leaf-1 | 20 | 20.0.0.0/24 | 20.0.0.254 |
| Leaf-2 | 10 | 10.0.0.0/24 | 10.0.0.254 |
| Leaf-2 | 20 | 20.0.0.0/24 | 20.0.0.254 |
| Leaf-3 | 10 | 10.0.0.0/24 | 10.0.0.254 |
| Leaf-21 | 10 | 10.0.0.0/24 | 10.0.0.254 |
| Leaf-21 | 20 | 20.0.0.0/24 | 20.0.0.254 |

---

## Конфигурация оборудования 

### SuperSpine
```
hostname SS

nv overlay evpn
feature ospf
feature bgp
feature bfd

no password strength-check
username admin password 5 $5$GPICGJ$o8/mMwoLudJKqd5ReF2UxZMlWgJYSszqe23qmJGTbA7 
 role network-admin
no ip domain-lookup
copp profile strict
bfd interval 100 min_rx 100 multiplier 5
configure maintenance profile normal-mode
  interface Ethernet1/1-3
    no shutdown
  sleep instance 1 1000
  router ospf UNDERLAY
    no isolate
  sleep instance 2 1200
  router bgp 65254
    address-family l2vpn evpn
      nexthop trigger-delay critical 1 non-critical 1
configure maintenance profile maintenance-mode
  interface Ethernet1/1-3
    shutdown
  router ospf UNDERLAY
    isolate
  sleep instance 1 100
  router bgp 65254
    address-family l2vpn evpn
      no nexthop trigger-delay
configure terminal



route-map NH_UNCH permit 10
  set ip next-hop unchanged
key chain OSPF_KC
  key 1
    key-string 7 070c285f4d06
    accept-lifetime 00:00:00 May 01 2026  infinite
    send-lifetime 00:00:00 May 01 2026  infinite
    cryptographic-algorithm HMAC-SHA-256


interface Ethernet1/1
  description ----Spine1_Eth1/4----
  no switchport
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.111.0/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf UNDERLAY area 0.0.0.0
  no shutdown

interface Ethernet1/2
  description ----Spine2_Eth1/4----
  no switchport
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.112.0/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf UNDERLAY area 0.0.0.0
  no shutdown

interface Ethernet1/3
  description ----Spine3_Eth1/1----
  no switchport
  mtu 9216
  no bfd echo
  no bfd ipv6 echo
  no ip redirects
  ip address 172.16.113.0/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf UNDERLAY area 0.0.0.0
  no shutdown

interface loopback0
  description ----Router-ID_and_BGP_Peering----
  ip address 172.0.0.254/32
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
icam monitor scale

line console
line vty
router ospf UNDERLAY
  bfd
  router-id 172.0.0.254
  passive-interface default
router bgp 65254
  router-id 172.0.0.254
  address-family ipv4 unicast
    maximum-paths 4
  address-family l2vpn evpn
    maximum-paths 4
    retain route-target all
  neighbor 99.99.99.111
    remote-as 65001
    update-source loopback0
    ebgp-multihop 5
    address-family ipv4 unicast
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
      route-map NH_UNCH out
  neighbor 99.99.99.112
    remote-as 65001
    update-source loopback0
    ebgp-multihop 5
    address-family ipv4 unicast
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
      route-map NH_UNCH out
  neighbor 99.99.99.113
    remote-as 65002
    update-source loopback0
    ebgp-multihop 5
    address-family ipv4 unicast
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
      route-map NH_UNCH out
```
### Spine1
```
hostname Spine1
vdc Spine1 id 1
  limit-resource vlan minimum 16 maximum 4094
  limit-resource vrf minimum 2 maximum 4096
  limit-resource port-channel minimum 0 maximum 511
  limit-resource u4route-mem minimum 248 maximum 248
  limit-resource u6route-mem minimum 96 maximum 96
  limit-resource m4route-mem minimum 58 maximum 58
  limit-resource m6route-mem minimum 8 maximum 8

nv overlay evpn
feature ospf
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature bfd
feature nv overlay
evpn multisite border-gateway 1
  delay-restore time 1000
  df-election time 1.0
  split-horizon per-site

no password strength-check
username admin password 5 $5$MMBDLO$iCwV/84fjiHnz2cGP212y77erlTuWU7GNZ.JA6nvoz3 
 role network-admin
no ip domain-lookup
copp profile strict
bfd interval 100 min_rx 100 multiplier 5
advertise evpn multicast

ip pim rp-address 172.0.0.1 group-list 225.0.0.0/24
ip pim ssm range 232.0.0.0/8
ip pim anycast-rp 172.0.0.1 172.0.0.111
ip pim anycast-rp 172.0.0.1 172.0.0.112
vlan 1,10,20,77
vlan 10
  name SERVERS_V10
  vn-segment 10000
vlan 20
  name SERVERS_V20
  vn-segment 20000
vlan 77
  name L3VNI
  vn-segment 770000

ip prefix-list PL_Lo99 seq 5 permit 99.99.99.111/32 
route-map INOSPFDCI permit 10
  match ip address prefix-list PL_Lo99 
route-map RMFORBGP permit 10
key chain OSPF_KC
  key 1
    key-string 7 070c285f4d06
    accept-lifetime 00:00:00 May 01 2026  infinite
    send-lifetime 00:00:00 May 01 2026  infinite
    cryptographic-algorithm HMAC-SHA-256
vrf context VRF_A
  vni 770000
  ip pim ssm range 232.0.0.0/8
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn

interface Vlan77
  description #####L3VNI_770000#####
  no shutdown
  vrf member VRF_A
  ip forward
  ip pim sparse-mode

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback99
  multisite border-gateway interface loopback100
  member vni 10000
    multisite ingress-replication
    mcast-group 225.0.0.10
  member vni 20000
    multisite ingress-replication
    ingress-replication protocol bgp
  member vni 770000 associate-vrf

interface Ethernet1/1
  description ----Leaf1_Eth1/1----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.11.1/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown
  evpn multisite fabric-tracking

interface Ethernet1/2
  description ----Leaf2_Eth1/2----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.21.1/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown
  evpn multisite fabric-tracking

interface Ethernet1/3
  description ----Leaf3_Eth1/3----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.31.1/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown
  evpn multisite fabric-tracking

interface Ethernet1/4
  description ----SuperSpine_Eth1/1----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.111.1/31
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf DCI area 0.0.0.0
  ip ospf bfd
  no shutdown
  evpn multisite dci-tracking

interface loopback0
  description #####Router-ID_and_BGP_Peering#####
  ip address 172.0.0.111/32
  ip ospf network point-to-point
  ip router ospf underlay area 0.0.0.0
  ip pim sparse-mode

interface loopback1
  description #####RP_ANYCAST_Address#####
  ip address 172.0.0.1/32
  ip ospf network point-to-point
  ip router ospf underlay area 0.0.0.0
  ip pim sparse-mode

interface loopback99
  description #####NVE_Source_Interface_PIP#####
  ip address 99.99.99.111/32
  ip router ospf underlay area 0.0.0.0

interface loopback100
  description #####MULTISITE_BGW_Anycast_VIP#####
  ip address 100.0.0.1/32
icam monitor scale

line console
line vty
router ospf DCI
  bfd
  router-id 172.0.0.111
  redistribute direct route-map INOSPFDCI
  passive-interface default
router ospf underlay
  bfd
  router-id 172.0.0.111
  passive-interface default
router bgp 65001
  router-id 172.0.0.111
  address-family ipv4 unicast
    network 100.0.0.1/32
    maximum-paths 2
  address-family l2vpn evpn
    maximum-paths 4
    retain route-target all
  template peer LEAFs
    bfd
    remote-as 65001
    update-source loopback0
    timers 3 9
    address-family ipv4 unicast
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  neighbor 172.0.0.11
    inherit peer LEAFs
  neighbor 172.0.0.12
    inherit peer LEAFs
  neighbor 172.0.0.13
    inherit peer LEAFs
  neighbor 172.0.0.112
    bfd
    remote-as 65001
    update-source loopback0
    timers 3 9
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  neighbor 172.0.0.254
    bfd
    remote-as 65254
    update-source loopback99
    ebgp-multihop 5
    timers 3 9
    peer-type fabric-external
    address-family ipv4 unicast
      send-community
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
  vrf VRF_A
    graceful-restart stalepath-time 1800
    bestpath as-path multipath-relax
    address-family ipv4 unicast
      redistribute direct route-map RMFORBGP
      maximum-paths 2
      maximum-paths ibgp 2
evpn
  vni 10000 l2
    rd auto
    route-target import 9999:10000
    route-target export 9999:10000
  vni 20000 l2
    rd auto
    route-target import 9999:20000
    route-target export 9999:20000
```
### Spine2
```
hostname Spine2
vdc Spine2 id 1
  limit-resource vlan minimum 16 maximum 4094
  limit-resource vrf minimum 2 maximum 4096
  limit-resource port-channel minimum 0 maximum 511
  limit-resource u4route-mem minimum 248 maximum 248
  limit-resource u6route-mem minimum 96 maximum 96
  limit-resource m4route-mem minimum 58 maximum 58
  limit-resource m6route-mem minimum 8 maximum 8

nv overlay evpn
feature ospf
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature bfd
feature nv overlay
evpn multisite border-gateway 1
  delay-restore time 1000
  df-election time 1.0
  split-horizon per-site

no password strength-check
username admin password 5 $5$JHEEJL$9u9dJh7OhX4qvKJLN0JYQ55VVbIZMg052jJaK1arjv0 
 role network-admin
no ip domain-lookup
copp profile strict
bfd interval 100 min_rx 100 multiplier 5
advertise evpn multicast

ip pim rp-address 172.0.0.1 group-list 225.0.0.0/24
ip pim ssm range 232.0.0.0/8
ip pim anycast-rp 172.0.0.1 172.0.0.111
ip pim anycast-rp 172.0.0.1 172.0.0.112
vlan 1,10,20,77
vlan 10
  name SERVERS_V10
  vn-segment 10000
vlan 20
  name SERVERS_V20
  vn-segment 20000
vlan 77
  name L3VNI
  vn-segment 770000

ip prefix-list PL_Lo99 seq 5 permit 99.99.99.112/32 
route-map INOSPFDCI permit 10
  match ip address prefix-list PL_Lo99 
route-map RMFORBGP permit 10
key chain OSPF_KC
  key 1
    key-string 7 070c285f4d06
    accept-lifetime 00:00:00 May 01 2026  infinite
    send-lifetime 00:00:00 May 01 2026  infinite
    cryptographic-algorithm HMAC-SHA-256
vrf context VRF_A
  vni 770000
  ip pim ssm range 232.0.0.0/8
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn

interface Vlan77
  description ####L3VNI_770000####
  no shutdown
  vrf member VRF_A
  ip forward
  ip pim sparse-mode

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback99
  multisite border-gateway interface loopback100
  member vni 10000
    multisite ingress-replication
    mcast-group 225.0.0.10
  member vni 20000
    multisite ingress-replication
    ingress-replication protocol bgp
  member vni 770000 associate-vrf

interface Ethernet1/1
  description ----Leaf1_Eth1/2----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.12.1/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown
  evpn multisite fabric-tracking

interface Ethernet1/2
  description ----Leaf2_Eth1/2----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.22.1/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown
  evpn multisite fabric-tracking

interface Ethernet1/3
  description ----Leaf3_Eth1/2----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.32.1/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown
  evpn multisite fabric-tracking

interface Ethernet1/4
  description ----SuperSpine_Eth1/2----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.112.1/31
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf DCI area 0.0.0.0
  ip ospf bfd
  no shutdown
  evpn multisite dci-tracking

interface loopback0
  description #####Router-ID_and_BGP_Peering#####
  ip address 172.0.0.112/32
  ip ospf network point-to-point
  ip router ospf underlay area 0.0.0.0
  ip pim sparse-mode

interface loopback1
  description #####RP_ANYCAST_Address#####
  ip address 172.0.0.1/32
  ip ospf network point-to-point
  ip router ospf underlay area 0.0.0.0
  ip pim sparse-mode

interface loopback99
  description #####NVE_Source_Interface_PIP#####
  ip address 99.99.99.112/32
  ip router ospf underlay area 0.0.0.0

interface loopback100
  description #####MULTISITE_BGW_Anycast_VIP#####
  ip address 100.0.0.1/32
icam monitor scale

line console
line vty
router ospf DCI
  bfd
  router-id 172.0.0.112
  redistribute direct route-map INOSPFDCI
  passive-interface default
router ospf underlay
  bfd
  router-id 172.0.0.112
  passive-interface default
router bgp 65001
  router-id 172.0.0.112
  address-family ipv4 unicast
    network 100.0.0.1/32
    maximum-paths 2
  address-family l2vpn evpn
    maximum-paths 4
    retain route-target all
  template peer LEAFs
    bfd
    remote-as 65001
    update-source loopback0
    timers 3 9
    address-family ipv4 unicast
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  neighbor 172.0.0.11
    inherit peer LEAFs
  neighbor 172.0.0.12
    inherit peer LEAFs
  neighbor 172.0.0.13
    inherit peer LEAFs
  neighbor 172.0.0.111
    bfd
    remote-as 65001
    update-source loopback0
    timers 3 9
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  neighbor 172.0.0.254
    bfd
    remote-as 65254
    update-source loopback99
    ebgp-multihop 3
    timers 3 9
    peer-type fabric-external
    address-family ipv4 unicast
      send-community
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
  vrf VRF_A
    graceful-restart stalepath-time 1800
    bestpath as-path multipath-relax
    address-family ipv4 unicast
      redistribute direct route-map RMFORBGP
      maximum-paths 2
      maximum-paths ibgp 2
evpn
  vni 10000 l2
    rd auto
    route-target import 9999:10000
    route-target export 9999:10000
  vni 20000 l2
    rd auto
    route-target import 9999:20000
    route-target export 9999:20000
```
### Spine3
```
hostname Spine3
vdc Spine3 id 1
  limit-resource vlan minimum 16 maximum 4094
  limit-resource vrf minimum 2 maximum 4096
  limit-resource port-channel minimum 0 maximum 511
  limit-resource u4route-mem minimum 248 maximum 248
  limit-resource u6route-mem minimum 96 maximum 96
  limit-resource m4route-mem minimum 58 maximum 58
  limit-resource m6route-mem minimum 8 maximum 8

nv overlay evpn
feature ospf
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature bfd
feature nv overlay
evpn multisite border-gateway 2
  delay-restore time 1000
  df-election time 1.0
  split-horizon per-site

no password strength-check
username admin password 5 $5$KGBBMO$k0n1Ey4AJ2KiRHxP.H3mjor4/n7FbmyFzlMJM5ulBb/ 
 role network-admin
no ip domain-lookup
copp profile strict
bfd interval 100 min_rx 100 multiplier 5
advertise evpn multicast

ip pim rp-address 172.0.0.2 group-list 225.0.0.0/24
ip pim ssm range 232.0.0.0/8
ip pim anycast-rp 172.0.0.2 172.0.0.113
vlan 1,10,20,77
vlan 10
  name SERVERS_V10
  vn-segment 10000
vlan 20
  name SERVERS_V20
  vn-segment 20000
vlan 77
  name L3VNI
  vn-segment 770000

ip prefix-list PL_Lo99 seq 5 permit 99.99.99.113/32 
route-map INOSPFDCI permit 10
  match ip address prefix-list PL_Lo99 
route-map RM_FORBGP permit 10
key chain OSPF_KC
  key 1
    key-string 7 070c285f4d06
    accept-lifetime 00:00:00 May 01 2026  infinite
    send-lifetime 00:00:00 May 01 2026  infinite
    cryptographic-algorithm HMAC-SHA-256
vrf context VRF_A
  vni 770000
  ip pim ssm range 232.0.0.0/8
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
vrf context management

interface Vlan77
  description ####L3VNI_770000####
  no shutdown
  vrf member VRF_A
  ip forward
  ip pim sparse-mode

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback99
  multisite border-gateway interface loopback100
  member vni 10000
    multisite ingress-replication
    mcast-group 225.0.0.10
  member vni 20000
    multisite ingress-replication
    ingress-replication protocol bgp
  member vni 770000 associate-vrf

interface Ethernet1/1
  description ----Leaf21_Eth1/1----
  no switchport
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.211.0/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown
  evpn multisite fabric-tracking

interface Ethernet1/4
  description ----SuperSpine_Eth1/3----
  no switchport
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.113.1/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf DCI area 0.0.0.0
  ip ospf bfd
  no shutdown
  evpn multisite dci-tracking

interface loopback0
  description #####Router-ID_and_BGP_Peering#####
  ip address 172.0.0.113/32
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode

interface loopback1
  description ####RP_ANYCAST_Address#####
  ip address 172.0.0.2/32
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode

interface loopback99
  description #####NVE_Source_Interface_PIP#####
  ip address 99.99.99.113/32
  ip router ospf UNDERLAY area 0.0.0.0

interface loopback100
  description #####MULTISITE_BGW_Anycast_VIP#####
  ip address 100.0.0.2/32
icam monitor scale

line console
line vty
router ospf DCI
  bfd
  router-id 172.0.0.113
  redistribute direct route-map INOSPFDCI
  passive-interface default
router ospf UNDERLAY
  bfd
  router-id 172.0.0.113
  passive-interface default
router bgp 65002
  router-id 172.0.0.113
  address-family ipv4 unicast
    network 100.0.0.2/32
    maximum-paths 2
  address-family l2vpn evpn
    maximum-paths 2
    retain route-target all
  neighbor 172.0.0.21
    remote-as 65002
    update-source loopback0
    timers 3 9
    address-family ipv4 unicast
      send-community
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  neighbor 172.0.0.254
    bfd
    remote-as 65254
    update-source loopback99
    ebgp-multihop 5
    timers 3 9
    peer-type fabric-external
    address-family ipv4 unicast
      send-community
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
  vrf VRF_A
    graceful-restart stalepath-time 1800
    bestpath as-path multipath-relax
    address-family ipv4 unicast
      redistribute direct route-map RM_FORBGP
      maximum-paths 2
      maximum-paths ibgp 2
evpn
  vni 10000 l2
    rd auto
    route-target import 9999:10000
    route-target export 9999:10000
  vni 20000 l2
    rd auto
    route-target import 9999:20000
    route-target export 9999:20000
```
### Leaf11
```
hostname Leaf11
vdc Leaf11 id 1
  limit-resource vlan minimum 16 maximum 4094
  limit-resource vrf minimum 2 maximum 4096
  limit-resource port-channel minimum 0 maximum 511
  limit-resource u4route-mem minimum 248 maximum 248
  limit-resource u6route-mem minimum 96 maximum 96
  limit-resource m4route-mem minimum 58 maximum 58
  limit-resource m6route-mem minimum 8 maximum 8

cfs eth distribute
nv overlay evpn
feature ospf
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature vpc
feature bfd
feature nv overlay

no password strength-check
username admin password 5 $5$MJNECA$WJtwrcRSha9sJUpHyTddAGw2k7RnjkLWiO5U08cdFD4 
 role network-admin
no ip domain-lookup
copp profile strict
bfd interval 100 min_rx 100 multiplier 5

fabric forwarding anycast-gateway-mac 0001.0001.0001
ip pim rp-address 172.0.0.1 group-list 225.0.0.0/24
ip pim ssm range 232.0.0.0/8
vlan 1,10,20,77,3900
vlan 10
  name SERVERS_V10
  vn-segment 10000
vlan 20
  name SERVERS_V20
  vn-segment 20000
vlan 77
  name L3VNI
  vn-segment 770000
vlan 3900
  name BACKUP_VLAN_ROUTING

route-map RM_BGP_REDISTR_OSPF permit 10
key chain OSPF_KC
  key 1
    key-string 7 070c285f4d06
    accept-lifetime 00:00:00 May 01 2026  infinite
    send-lifetime 00:00:00 May 01 2026  infinite
    cryptographic-algorithm HMAC-SHA-256
vrf context VRF_A
  vni 770000
  ip pim ssm range 232.0.0.0/8
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
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
vlan configuration 10


interface Vlan1
  no ip redirects
  no ipv6 redirects

interface Vlan10
  description ----SERVERS_VLAN10----
  no shutdown
  vrf member VRF_A
  ip address 10.0.0.254/24
  fabric forwarding mode anycast-gateway

interface Vlan20
  description ----SERVERS_VLAN20----
  no shutdown
  vrf member VRF_A
  ip address 20.0.0.254/24
  fabric forwarding mode anycast-gateway

interface Vlan77
  description ####L3VNI_770000####
  no shutdown
  vrf member VRF_A
  ip forward
  ip pim sparse-mode

interface Vlan3900
  description VPC_L3_Peering_VXLAN
  no shutdown
  mtu 9216
  no ip redirects
  ip address 172.16.39.0/31
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd

interface port-channel1
  description ----VPC_PEER_TO_LEAF12----
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77,3900
  spanning-tree port type network
  vpc peer-link

interface port-channel10
  description ----VPC_EP1----
  switchport
  switchport access vlan 10
  mtu 9216
  vpc 11

interface port-channel20
  description ----VPC_EP2----
  switchport
  switchport access vlan 20
  mtu 9216
  vpc 12

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 10000
    suppress-arp
    mcast-group 225.0.0.10
  member vni 20000
    ingress-replication protocol bgp
  member vni 770000 associate-vrf

interface Ethernet1/1
  description ----Spine1_Eth1/1----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.11.0/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown

interface Ethernet1/2
  description ----Spine2_Eth1/1----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.12.0/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown

interface Ethernet1/4
  description ----To_Server_VLAN10----
  switchport
  switchport access vlan 10
  mtu 9216
  channel-group 10 mode active
  no shutdown

interface Ethernet1/5
  description ----To_Server_VLAN20----
  switchport
  switchport access vlan 20
  mtu 9216
  channel-group 20 mode active
  no shutdown

interface Ethernet1/6
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77,3900
  channel-group 1 mode active
  no shutdown

interface Ethernet1/7
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77,3900
  channel-group 1 mode active
  no shutdown

interface mgmt0
  vrf member management
  ip address 192.168.1.0/31

interface loopback0
  description #####Router_ID_and_BGP_Peering#####
  ip address 172.0.0.11/32
  ip address 172.0.0.100/24 secondary
  ip ospf network point-to-point
  ip router ospf underlay area 0.0.0.0
  ip pim sparse-mode
icam monitor scale

line console
line vty
router ospf underlay
  bfd
  router-id 172.0.0.11
  passive-interface default
router bgp 65001
  router-id 172.0.0.100
  bestpath as-path multipath-relax
  address-family ipv4 unicast
    maximum-paths 2
    maximum-paths ibgp 2
  neighbor 172.0.0.111
    remote-as 65001
    update-source loopback0
    address-family ipv4 unicast
      send-community
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 172.0.0.112
    remote-as 65001
    update-source loopback0
    address-family ipv4 unicast
      send-community
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
  vrf VRF_A
    bestpath as-path multipath-relax
    address-family ipv4 unicast
      redistribute direct route-map RM_BGP_REDISTR_OSPF
      maximum-paths 2
      maximum-paths ibgp 2
evpn
  vni 10000 l2
    rd auto
    route-target import 9999:10000
    route-target export 9999:10000
  vni 20000 l2
    rd auto
    route-target import 9999:20000
    route-target export 9999:20000
```
### Leaf12
```
  
hostname Leaf12
vdc Leaf12 id 1
  limit-resource vlan minimum 16 maximum 4094
  limit-resource vrf minimum 2 maximum 4096
  limit-resource port-channel minimum 0 maximum 511
  limit-resource u4route-mem minimum 248 maximum 248
  limit-resource u6route-mem minimum 96 maximum 96
  limit-resource m4route-mem minimum 58 maximum 58
  limit-resource m6route-mem minimum 8 maximum 8

cfs eth distribute
nv overlay evpn
feature ospf
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature vpc
feature bfd
feature nv overlay

no password strength-check
username admin password 5 $5$CLDLGF$U4hsZpMQXFCC4wmC6dVRVqBko6wOUz6/FqeoAlvOdbD 
 role network-admin
no ip domain-lookup
copp profile strict
bfd interval 100 min_rx 100 multiplier 5

fabric forwarding anycast-gateway-mac 0001.0001.0001
ip pim rp-address 172.0.0.1 group-list 225.0.0.0/24
ip pim ssm range 232.0.0.0/8
vlan 1,10,20,77,3900
vlan 10
  name SERVERS_V10
  vn-segment 10000
vlan 20
  name SERVERS_V20
  vn-segment 20000
vlan 77
  name L3VNI
  vn-segment 770000
vlan 3900
  name BACKUP_VLAN_ROUTING

spanning-tree vlan 10,20,77 priority 8192
route-map RM_BGP_REDISTR_OSPF permit 10
key chain OSPF_KC
  key 1
    key-string 7 070c285f4d06
    accept-lifetime 00:00:00 May 01 2026  infinite
    send-lifetime 00:00:00 May 01 2026  infinite
    cryptographic-algorithm HMAC-SHA-256
vrf context VRF_A
  vni 770000
  ip pim ssm range 232.0.0.0/8
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
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
vlan configuration 10


interface Vlan1
  no ip redirects
  no ipv6 redirects

interface Vlan10
  description ----SERVERS_VLAN10----
  no shutdown
  vrf member VRF_A
  ip address 10.0.0.254/24
  fabric forwarding mode anycast-gateway

interface Vlan20
  description ----SERVERS_VLAN20----
  no shutdown
  vrf member VRF_A
  ip address 20.0.0.254/24
  fabric forwarding mode anycast-gateway

interface Vlan77
  description ####L3VNI_770000####
  no shutdown
  vrf member VRF_A
  ip forward
  ip pim sparse-mode

interface Vlan3900
  description VPC_L3_Peering_VXLAN
  no shutdown
  mtu 9216
  no ip redirects
  ip address 172.16.39.1/31
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd

interface port-channel1
  description ----VPC_PEER_TO_LEAF12----
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77,3900
  spanning-tree port type network
  vpc peer-link

interface port-channel10
  description ----VPC_EP1----
  switchport
  switchport access vlan 10
  mtu 9216
  vpc 11

interface port-channel20
  description ----VPC_EP2----
  switchport
  switchport access vlan 20
  mtu 9216
  vpc 12

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 10000
    suppress-arp
    mcast-group 225.0.0.10
  member vni 20000
    ingress-replication protocol bgp
  member vni 770000 associate-vrf

interface Ethernet1/1
  description ----Spine1_Eth1/2----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.21.0/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown

interface Ethernet1/2
  description ----Spine2_Eth1/2----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.22.0/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown

interface Ethernet1/4
  description ----To_Server_VLAN10----
  switchport
  switchport access vlan 10
  mtu 9216
  channel-group 10 mode active
  no shutdown

interface Ethernet1/5
  description ----To_Server_VLAN20----
  switchport
  switchport access vlan 20
  mtu 9216
  channel-group 20 mode active
  no shutdown

interface Ethernet1/6
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77,3900
  channel-group 1 mode active
  no shutdown

interface Ethernet1/7
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 10,20,77,3900
  channel-group 1 mode active
  no shutdown

interface mgmt0
  vrf member management
  ip address 192.168.1.1/31

interface loopback0
  description #####Router_ID_and_BGP_Peering#####
  ip address 172.0.0.12/32
  ip address 172.0.0.100/24 secondary
  ip ospf network point-to-point
  ip router ospf underlay area 0.0.0.0
  ip pim sparse-mode
icam monitor scale

line console
line vty
router ospf underlay
  bfd
  router-id 172.0.0.12
  passive-interface default
router bgp 65001
  router-id 172.0.0.100
  bestpath as-path multipath-relax
  address-family ipv4 unicast
    maximum-paths 2
    maximum-paths ibgp 2
  neighbor 172.0.0.111
    remote-as 65001
    update-source loopback0
    address-family ipv4 unicast
      send-community
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 172.0.0.112
    remote-as 65001
    update-source loopback0
    address-family ipv4 unicast
      send-community
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
  vrf VRF_A
    bestpath as-path multipath-relax
    address-family ipv4 unicast
      redistribute direct route-map RM_BGP_REDISTR_OSPF
      maximum-paths 2
      maximum-paths ibgp 2
evpn
  vni 10000 l2
    rd auto
    route-target import 9999:10000
    route-target export 9999:10000
  vni 20000 l2
    rd auto
    route-target import 9999:20000
    route-target export 9999:20000
```
### Leaf13
```
 
hostname Leaf13
vdc Leaf13 id 1
  limit-resource vlan minimum 16 maximum 4094
  limit-resource vrf minimum 2 maximum 4096
  limit-resource port-channel minimum 0 maximum 511
  limit-resource u4route-mem minimum 248 maximum 248
  limit-resource u6route-mem minimum 96 maximum 96
  limit-resource m4route-mem minimum 58 maximum 58
  limit-resource m6route-mem minimum 8 maximum 8

nv overlay evpn
feature ospf
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature bfd
feature nv overlay

no password strength-check
username admin password 5 $5$OMALEM$dfuB8ezS7K.4GOK1FoP2r.I.CZQAvkS1QUdep8QIB77 
 role network-admin
no ip domain-lookup
copp profile strict
bfd interval 100 min_rx 100 multiplier 5

fabric forwarding anycast-gateway-mac 0001.0001.0001
ip pim rp-address 172.0.0.1 group-list 225.0.0.0/24
ip pim ssm range 232.0.0.0/8
vlan 1,10,77
vlan 10
  name SERVERS
  vn-segment 10000
vlan 77
  name L3VNI
  vn-segment 770000

route-map RM_FORBGP permit 10
key chain OSPF_KC
  key 1
    key-string 7 070c285f4d06
    accept-lifetime 00:00:00 May 01 2026  infinite
    send-lifetime 00:00:00 May 01 2026  infinite
    cryptographic-algorithm HMAC-SHA-256
vrf context VRF_A
  vni 770000
  ip pim ssm range 232.0.0.0/8
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
vrf context management
hardware access-list tcam region racl 1024
hardware access-list tcam region span 0
hardware access-list tcam region rp-ipv6-qos 0
hardware access-list tcam region arp-ether 256 double-wide
vlan configuration 10

interface Vlan10
  description ----SERVERS_VLAN10----
  no shutdown
  vrf member VRF_A
  ip address 10.0.0.254/24
  fabric forwarding mode anycast-gateway

interface Vlan77
  description ####L3VNI_770000#### 
  no shutdown
  vrf member VRF_A
  ip forward
  ip pim sparse-mode

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 10000
    suppress-arp
    mcast-group 225.0.0.10
  member vni 770000 associate-vrf

interface Ethernet1/1
  description ----Spine1_Eth1/3----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.31.0/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown

interface Ethernet1/2
  description ----To_Server_VLAN10----
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.32.0/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf underlay area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown

interface Ethernet1/3
  switchport
  switchport access vlan 10
  mtu 9216
  no shutdown

interface loopback0
  description #####Router_ID#####
  ip address 172.0.0.13/32
  ip router ospf underlay area 0.0.0.0
  ip pim sparse-mode
icam monitor scale

line console
line vty
router ospf underlay
  bfd
  router-id 172.0.0.13
  passive-interface default
router bgp 65001
  router-id 172.0.0.13
  bestpath as-path multipath-relax
  address-family ipv4 unicast
    maximum-paths 2
    maximum-paths ibgp 2
  neighbor 172.0.0.111
    remote-as 65001
    update-source loopback0
    address-family ipv4 unicast
      send-community
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 172.0.0.112
    remote-as 65001
    update-source loopback0
    address-family ipv4 unicast
      send-community
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
  vrf VRF_A
    bestpath as-path multipath-relax
    address-family ipv4 unicast
      redistribute direct route-map RM_FORBGP
      maximum-paths 2
      maximum-paths ibgp 2
evpn
  vni 10000 l2
    rd auto
    route-target import 9999:10000
    route-target export 9999:10000
```
### Leaf21
```
 
hostname Leaf21
vdc Leaf21 id 1
  limit-resource vlan minimum 16 maximum 4094
  limit-resource vrf minimum 2 maximum 4096
  limit-resource port-channel minimum 0 maximum 511
  limit-resource u4route-mem minimum 248 maximum 248
  limit-resource u6route-mem minimum 96 maximum 96
  limit-resource m4route-mem minimum 58 maximum 58
  limit-resource m6route-mem minimum 8 maximum 8

nv overlay evpn
feature ospf
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature bfd
feature nv overlay

no password strength-check
username admin password 5 $5$CEIAGP$1mk5/JIsaFyy4Jx/7NeH.ErS4s0JrimF6KNYiS/hUmC 
 role network-admin
ip domain-lookup
copp profile strict
bfd interval 100 min_rx 100 multiplier 5

fabric forwarding anycast-gateway-mac 0001.0001.0001
ip pim rp-address 172.0.0.2 group-list 225.0.0.0/24
ip pim ssm range 232.0.0.0/8
vlan 1,10,20,77
vlan 10
  name SERVERS_V10
  vn-segment 10000
vlan 20
  name SERVERS_V20
  vn-segment 20000
vlan 77
  name L3VNI
  vn-segment 770000

route-map RM_FORBGP permit 10
key chain OSPF_KC
  key 1
    key-string 7 070c285f4d06
    accept-lifetime 00:00:00 May 01 2026  infinite
    send-lifetime 00:00:00 May 01 2026  infinite
    cryptographic-algorithm HMAC-SHA-256
vrf context VRF_A
  vni 770000
  ip pim ssm range 232.0.0.0/8
  rd auto
  address-family ipv4 unicast
    route-target import 7777:770000
    route-target import 7777:770000 evpn
    route-target export 7777:770000
    route-target export 7777:770000 evpn
vrf context management
hardware access-list tcam region racl 1024
hardware access-list tcam region span 0
hardware access-list tcam region rp-ipv6-qos 0
hardware access-list tcam region arp-ether 256 double-wide
vlan configuration 10
  ip igmp snooping querier 172.0.0.21


interface Vlan1

interface Vlan10
  description ----SERVERS_VLAN10----
  no shutdown
  vrf member VRF_A
  ip address 10.0.0.254/24
  fabric forwarding mode anycast-gateway

interface Vlan20
  description ----SERVERS_VLAN20----
  no shutdown
  vrf member VRF_A
  ip address 20.0.0.254/24
  fabric forwarding mode anycast-gateway

interface Vlan77
  description ####L3VNI_770000####
  no shutdown
  vrf member VRF_A
  ip forward
  ip pim sparse-mode

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 10000
    suppress-arp
    mcast-group 225.0.0.10
  member vni 20000
    suppress-arp
    ingress-replication protocol bgp
  member vni 770000 associate-vrf

interface Ethernet1/1
  description ----Spine3_Eth1/2----
  no switchport
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  no ip redirects
  ip address 172.16.211.1/31
  no ipv6 redirects
  ip ospf authentication message-digest
  ip ospf authentication key-chain OSPF_KC
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf UNDERLAY area 0.0.0.0
  ip ospf bfd
  ip pim sparse-mode
  no shutdown


interface Ethernet1/4
  description ----To_Servers_VLAN10----
  switchport access vlan 10

interface Ethernet1/5
  description ----To_Servers_VLAN20----
  switchport access vlan 20

interface loopback0
  description #####Router_ID#####
  ip address 172.0.0.21/32
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
icam monitor scale

line console
line vty
router ospf UNDERLAY
  bfd
  router-id 172.0.0.21
  passive-interface default
router bgp 65002
  router-id 172.0.0.21
  address-family ipv4 unicast
  neighbor 172.0.0.113
    remote-as 65002
    update-source loopback0
    address-family ipv4 unicast
      send-community
      send-community extended
    address-family l2vpn evpn
      send-community
      send-community extended
  vrf VRF_A
    bestpath as-path multipath-relax
    address-family ipv4 unicast
      redistribute direct route-map RM_FORBGP
      maximum-paths 2
      maximum-paths ibgp 2
evpn
  vni 10000 l2
    rd auto
    route-target import 9999:10000
    route-target export 9999:10000
  vni 20000 l2
    rd auto
    route-target import 9999:20000
    route-target export 9999:20000
```

## Проверки 
