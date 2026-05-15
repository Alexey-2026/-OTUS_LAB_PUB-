# Настройка VxLAN L3VNI

## Цель работы
Настроить маршрутизацию в рамках Overlay между клиентами.

<img src="https://github.com/user-attachments/assets/23ab6f5a-7a70-4a3a-8085-843593989844" width="300" style="max-width: 100%; height: auto;">



## План адресации

### Underlay сеть (линки /31)

| Линк | Spine интерфейс | IP Spine | Leaf интерфейс | IP Leaf |
|------|----------------|----------|----------------|---------|
| Spine1-Leaf1 | Eth1/1 | 172.16.10.1 | Eth1/6 | 172.16.10.0 |
| Spine1-Leaf2 | Eth1/2 | 172.16.10.3 | Eth1/6 | 172.16.10.2 |


### Loopback адреса

| Устройство | Loopback0 IP |
|------------|--------------|
| Spine-1 | 172.16.0.111/32 |
| Leaf-1 | 172.16.0.11/32 |
| Leaf-2 | 172.16.0.12/32 |


### Серверные подсети

| Leaf | VLAN | Подсеть | Шлюз |
|------|------|---------|------|
| Leaf-1 | 10 | 10.0.0.0/24 | 10.0.0.254 |
| Leaf-1 | 20 | 20.0.0.0/24 | 20.0.0.254 |
| Leaf-2 | 10 | 10.0.0.0/24 | 10.0.0.254 |

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
bfd interval 100 min_rx 100 multiplier 5
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
!
interface loopback0
  ip address 172.16.0.111/32
!
interface Ethernet1/1
  description Link_To_Leaf1
  bfd
  no ip redirects
  ip address 172.16.10.1/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/2
  description Link_To_Leaf2
  bfd
  no ip redirects
  ip address 172.16.10.3/31
  no ipv6 redirects
  no shutdown
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
!
```

### Leaf-1
```
hostname Leaf1
!
nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature bfd
feature nv overlay
!
bfd interval 100 min_rx 100 multiplier 5
!
fabric forwarding anycast-gateway-mac 0001.0001.0001
!
vlan 1,10,20,77
vlan 10
  name SEVERS_VLAN10
  vn-segment 10000
vlan 20
  name SEVERS_VLAN20
  vn-segment 20000
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
  ip address 10.0.0.254/24
  fabric forwarding mode anycast-gateway
!
interface Vlan20
  description ----SERVERS_VLAN20----
  no shutdown
  vrf member VRF_A
  ip address 20.0.0.254/24
  fabric forwarding mode anycast-gateway
!
interface Vlan77
  description ####L3VNI_770000####
  no shutdown
  vrf member VRF_A
  ip forward
!
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  global ingress-replication protocol bgp
  member vni 10000
  member vni 20000
  member vni 770000 associate-vrf
!
interface Loopback0
  ip address 172.16.0.11 255.255.255.255
!
interface Ethernet1/1
  description TO_CLIENT_VLAN10
  switchport
  switchport access vlan 10
  no shutdown
!
interface Ethernet1/2
  description TO_CLIENT_VLAN20
  switchport
  switchport access vlan 20
  no shutdown
!
interface Ethernet1/6
  description Link_To_Spine1
  bfd
  no ip redirects
  ip address 172.16.10.0/31
  no ipv6 redirects
  no shutdown
!
router bgp 65501
  router-id 172.16.0.11
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
  neighbor 172.16.10.1
    inherit peer SPINEs
    description To_Spine1
evpn
  vni 10000 l2
    rd auto
    route-target import 65501:10000
    route-target export 65501:10000
  vni 20000 l2
    rd auto
    route-target import 65501:20000
    route-target export 65501:20000
!
```

### Leaf-2
```
hostname Leaf2
!
nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature bfd
feature nv overlay
!
bfd interval 100 min_rx 100 multiplier 5
!
fabric forwarding anycast-gateway-mac 0001.0001.0001
!
vlan 1,10,77
vlan 10
  name FOR_CLIENT
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
  ip address 10.0.0.254/24
  fabric forwarding mode anycast-gateway
!
interface Vlan77
  description ####L3VNI_770000####
  no shutdown
  vrf member VRF_A
  ip forward
!
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  global ingress-replication protocol bgp
  member vni 10000
  member vni 770000 associate-vrf
!
interface Loopback0
ip address 172.16.0.12 255.255.255.255
!
interface Ethernet1/1
  description TO_CLIENT_VLAN10
  switchport
  switchport access vlan 10
  no shutdown
!
interface Ethernet1/6
  description Link_To_Spine1
  bfd
  no ip redirects
  ip address 172.16.10.2/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/1
description Server_Network
switchport
switchport access vlan 10
no shutdown
!
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
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 172.16.10.3
    inherit peer SPINEs
    description ---To_Spine1---
evpn
  vni 10000 l2
    rd auto
    route-target import 65501:10000
    route-target export 65501:10000
!
```

---

## Проверка 
```
Spine1# sh ip route 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

172.16.0.11/32, ubest/mbest: 1/0
    *via 172.16.10.0, [200/0], 01:24:36, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 1/0
    *via 172.16.10.2, [200/0], 01:45:06, bgp-65501, internal, tag 65501
172.16.0.111/32, ubest/mbest: 2/0, attached
    *via 172.16.0.111, Lo0, [0/0], 3d12h, local
    *via 172.16.0.111, Lo0, [0/0], 3d12h, direct
172.16.10.0/31, ubest/mbest: 1/0, attached
    *via 172.16.10.1, Eth1/1, [0/0], 3d06h, direct
172.16.10.1/32, ubest/mbest: 1/0, attached
    *via 172.16.10.1, Eth1/1, [0/0], 3d06h, local
172.16.10.2/31, ubest/mbest: 1/0, attached
    *via 172.16.10.3, Eth1/2, [0/0], 3d06h, direct
172.16.10.3/32, ubest/mbest: 1/0, attached
    *via 172.16.10.3, Eth1/2, [0/0], 3d06h, local
172.16.10.4/31, ubest/mbest: 1/0, attached
    *via 172.16.10.5, Eth1/3, [0/0], 3d06h, direct
172.16.10.5/32, ubest/mbest: 1/0, attached
    *via 172.16.10.5, Eth1/3, [0/0], 3d06h, local

Spine1# show bgp ip
ipv4   ipv6   
Spine1# show bgp ipv4 unicast summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.111, local AS number 65501
BGP table version is 48, IPv4 Unicast config peers 3, capable peers 2
3 network entries and 3 paths using 732 bytes of memory
BGP attribute entries [2/344], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.0     4 65501    1696    1690       48    0    0 01:24:56 1         
172.16.10.2     4 65501    2109    2100       48    0    0 01:45:26 1         
Spine1# show bgp ipv4 unicast 
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 48, Local Router ID is 172.16.0.111
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>i172.16.0.11/32     172.16.10.0              0        100          0 ?
*>i172.16.0.12/32     172.16.10.2              0        100          0 ?
*>r172.16.0.111/32    0.0.0.0                  0        100      32768 ?

Spine1# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 93, Local Router ID is 172.16.0.111
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.11                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.11                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.11]/88
                      172.16.0.11                       100          0 i

Route Distinguisher: 172.16.0.11:32787
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.11                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.11                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.11]/88
                      172.16.0.11                       100          0 i

Route Distinguisher: 172.16.0.12:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.12                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.12                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.12]/88
                      172.16.0.12                       100          0 i

Spine1# show bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216,
 version 76
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.2    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/2
72, version 91
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8600.1
b08

  Path-id 1 advertised to peers:
    172.16.10.2    

Spine1# show bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216,
 version 78
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000
      Extcommunity: RT:65501:20000 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.2    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/2
72, version 88
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 ENCAP:8 Router MAC:5002.8600.1
b08

  Path-id 1 advertised to peers:
    172.16.10.2    

Spine1# show bgp l2vpn evpn 0000.0000.0013
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216,
 version 84
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.12 (metric 0) from 172.16.10.2 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.0    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/2
72, version 93
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path sourced internal to AS
    172.16.0.12 (metric 0) from 172.16.10.2 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8400.1
b08

  Path-id 1 advertised to peers:
    172.16.10.0  

=======================================================================

Leaf1# sh bgp ipv4 unicast summary

BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.11, local AS number 65501
BGP table version is 123, IPv4 Unicast config peers 2, capable peers 1
3 network entries and 3 paths using 732 bytes of memory
BGP attribute entries [3/516], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [1/4]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.1     4 65501   93901   93897      123    0    0 01:27:25 2         
172.16.10.7     4 65501   26474   26420        0    0    0 00:42:32 Idle
    
Leaf1# sh bgp ipv4 unicast

BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 123, Local Router ID is 172.16.0.11
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>r172.16.0.11/32     0.0.0.0                  0        100      32768 ?
*>i172.16.0.12/32     172.16.10.1              0        100          0 ?
*>i172.16.0.111/32    172.16.10.1              0        100          0 ?

Leaf1# sh bgp l2vpn evpn summary

BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 172.16.0.11, local AS number 65501
BGP table version is 48, L2VPN EVPN config peers 2, capable peers 1
6 network entries and 6 paths using 1224 bytes of memory
BGP attribute entries [5/860], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [1/4]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.1     4 65501   93906   93902       48    0    0 01:27:40 2         
    
Leaf1# sh bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 116, Local Router ID is 172.16.0.11
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.11                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.12                       100          0 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.11                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.12                       100          0 i
*>l[3]:[0]:[32]:[172.16.0.11]/88
                      172.16.0.11                       100      32768 i
*>i[3]:[0]:[32]:[172.16.0.12]/88
                      172.16.0.12                       100          0 i

Route Distinguisher: 172.16.0.11:32787    (L2VNI 20000)
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.11                       100      32768 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.11                       100      32768 i
*>l[3]:[0]:[32]:[172.16.0.11]/88
                      172.16.0.11                       100      32768 i

Route Distinguisher: 172.16.0.12:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.12                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.12                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.12]/88
                      172.16.0.12                       100          0 i

Route Distinguisher: 172.16.0.11:3    (L3VNI 770000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.12                       100          0 i

Leaf1# sh bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216,
 version 95
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.11 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.1    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/2
72, version 113
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.11 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8600.1
b08

  Path-id 1 advertised to peers:
    172.16.10.1    

Leaf1# sh bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32787    (L2VNI 20000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216,
 version 96
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.11 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 20000
      Extcommunity: RT:65501:20000 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.1    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/2
72, version 111
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.11 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 ENCAP:8 Router MAC:5002.8600.1
b08

  Path-id 1 advertised to peers:
    172.16.10.1    

Leaf1# sh bgp l2vpn evpn 0000.0000.0013
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216,
 version 107
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[
0]:[0.0.0.0]/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.12 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/2
72, version 116
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[
32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.12 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8400.1
b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216,
 version 106
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.12 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/2
72, version 114
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 3 destination(s)
             Imported paths list: VRF_A L2-10000 L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.12 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8400.1
b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.11:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/2
72, version 115
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0013]:[
32]:[10.0.0.13]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.12 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8400.1
b08
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf1# sh bgp l2vpn evpn route-type 3
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
BGP routing table entry for [3]:[0]:[32]:[172.16.0.11]/88, version 97
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.11 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.11

  Path-id 1 advertised to peers:
    172.16.10.1    
BGP routing table entry for [3]:[0]:[32]:[172.16.0.12]/88, version 101
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.12:32777:[3]:[0]:[32]:[172.16.0.12]/88 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.12 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.12

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.11:32787    (L2VNI 20000)
BGP routing table entry for [3]:[0]:[32]:[172.16.0.11]/88, version 98
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.11 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin IGP, MED not set, localpref 100, weight 32768
      Extcommunity: RT:65501:20000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 20000, Tunnel Id: 172.16.0.11

  Path-id 1 advertised to peers:
    172.16.10.1    

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.12]/88, version 90
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.12 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.12

  Path-id 1 not advertised to any peer

Leaf1# sh nve peers
Interface Peer-IP                                 State LearnType Uptime   Route
r-Mac       
--------- --------------------------------------  ----- --------- -------- -----
------------
nve1      172.16.0.12                             Up    CP        02:02:58 5002.
8400.1b08   

Leaf1# sh nve interface nve 1 detail
Interface: nve1, State: Up, encapsulation: VXLAN
 VPC Capability: VPC-VIP-Only [not-notified]
 Local Router MAC: 5002.8600.1b08
 Host Learning Mode: Control-Plane
 Source-Interface: loopback0 (primary: 172.16.0.11, secondary: 0.0.0.0)
 Source Interface State: Up
 Virtual RMAC Advertisement: No
 NVE Flags: 
 Interface Handle: 0x49000001
 Source Interface hold-down-time: 180
 Source Interface hold-up-time: 30
 Remaining hold-down time: 0 seconds
 Virtual Router MAC: N/A
 Interface state: nve-intf-add-complete

Leaf1# sh mac address-table 
Legend: 
* - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
age - seconds since last seen,+ - primary entry using vPC Peer-Link,
(T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
*   10     0000.0000.0011   dynamic  0         F      F    Eth1/1
C   10     0000.0000.0013   dynamic  0         F      F    nve1(172.16.0.12)
*   20     0000.0000.0012   dynamic  0         F      F    Eth1/2
*   77     5002.8400.1b08   static   -         F      F    nve1(172.16.0.12)
*   77     5002.8600.1b08   static   -         F      F    Vlan77
G    -     0001.0001.0001   static   -         F      F    sup-eth1(R)
G    -     5002.8600.1b08   static   -         F      F    sup-eth1(R)
G   10     5002.8600.1b08   static   -         F      F    sup-eth1(R)
G   20     5002.8600.1b08   static   -         F      F    sup-eth1(R)
G   77     5002.8600.1b08   static   -         F      F    sup-eth1(R)

Leaf1# sh ip rout vrf vrF_A 
IP Route Table for VRF "VRF_A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.0.0.0/24, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 01:41:19, direct
10.0.0.11/32, ubest/mbest: 1/0, attached
    *via 10.0.0.11, Vlan10, [190/0], 01:40:42, hmm
10.0.0.13/32, ubest/mbest: 1/0
    *via 172.16.0.12%default, [200/0], 01:21:01, bgp-65501, internal, tag 65501,
 segid: 770000 tunnelid: 0xac10000c encap: VXLAN
 
10.0.0.254/32, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 01:41:19, local
20.0.0.0/24, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 02:10:49, direct
20.0.0.12/32, ubest/mbest: 1/0, attached
    *via 20.0.0.12, Vlan20, [190/0], 02:10:47, hmm
20.0.0.254/32, ubest/mbest: 1/0, attached
    *via 20.0.0.254, Vlan20, [0/0], 02:10:49, local


=============================================================================

Leaf2# sh bgp ipv4 unicast summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.12, local AS number 65501
BGP table version is 84, IPv4 Unicast config peers 2, capable peers 1
3 network entries and 3 paths using 732 bytes of memory
BGP attribute entries [3/516], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [1/4]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.3     4 65501   99522   99502       84    0    0 02:00:40 2         
 
Leaf2# sh bgp ipv4 unicast
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 84, Local Router ID is 172.16.0.12
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>i172.16.0.11/32     172.16.10.3              0        100          0 ?
*>r172.16.0.12/32     0.0.0.0                  0        100      32768 ?
*>i172.16.0.111/32    172.16.10.3              0        100          0 ?

Leaf2# sh bgp l2vpn evpn summary
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 172.16.0.12, local AS number 65501
BGP table version is 42, L2VPN EVPN config peers 2, capable peers 1
6 network entries and 6 paths using 1224 bytes of memory
BGP attribute entries [5/860], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [1/4]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.3     4 65501   99532   99511       42    0    0 02:01:09 2         
    
Leaf2# sh bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 90, Local Router ID is 172.16.0.12
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.11                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.11                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.11]/88
                      172.16.0.11                       100          0 i

Route Distinguisher: 172.16.0.11:32787
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.11                       100          0 i

Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.11                       100          0 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.16.0.12                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.11                       100          0 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.16.0.12                       100      32768 i
*>i[3]:[0]:[32]:[172.16.0.11]/88
                      172.16.0.11                       100          0 i
*>l[3]:[0]:[32]:[172.16.0.12]/88
                      172.16.0.12                       100      32768 i

Route Distinguisher: 172.16.0.12:3    (L3VNI 770000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.16.0.11                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.16.0.11                       100          0 i

Leaf2#  sh bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216,
 version 69
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/2
72, version 83
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 3 destination(s)
             Imported paths list: L2-10000 L3-770000 VRF_A
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8600.1
b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216,
 version 72
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[
0]:[0.0.0.0]/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/2
72, version 84
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[
32]:[10.0.0.11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8600.1
b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/2
72, version 88
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32777:[2]:[0]:[0]:[48]:[0000.0000.0011]:[
32]:[10.0.0.11]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8600.1
b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf2#  sh bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/2
72, version 86
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 2 destination(s)
             Imported paths list: VRF_A L3-770000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 ENCAP:8 Router MAC:5002.8600.1
b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:3    (L3VNI 770000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/2
72, version 89
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32787:[2]:[0]:[0]:[48]:[0000.0000.0012]:[
32]:[20.0.0.12]/272 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 20000 770000
      Extcommunity: RT:7777:770000 RT:65501:20000 ENCAP:8 Router MAC:5002.8600.1
b08
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Leaf2#  sh bgp l2vpn evpn 0000.0000.0013
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216,
 version 76
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.12 (metric 0) from 0.0.0.0 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8

  Path-id 1 advertised to peers:
    172.16.10.3    
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/2
72, version 90
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.12 (metric 0) from 0.0.0.0 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10000 770000
      Extcommunity: RT:7777:770000 RT:65501:10000 ENCAP:8 Router MAC:5002.8400.1
b08

  Path-id 1 advertised to peers:
    172.16.10.3    

Leaf2#  sh bgp l2vpn evpn route-type 3
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.11]/88, version 70
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10000
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.11

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
BGP routing table entry for [3]:[0]:[32]:[172.16.0.11]/88, version 73
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 172.16.0.11:32777:[3]:[0]:[32]:[172.16.0.11]/88 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.11 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.11

  Path-id 1 not advertised to any peer
BGP routing table entry for [3]:[0]:[32]:[172.16.0.12]/88, version 5
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    172.16.0.12 (metric 0) from 0.0.0.0 (172.16.0.12)
      Origin IGP, MED not set, localpref 100, weight 32768
      Extcommunity: RT:65501:10000 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10000, Tunnel Id: 172.16.0.12

  Path-id 1 advertised to peers:
    172.16.10.3    

Leaf2# sh nve peers
Interface Peer-IP                                 State LearnType Uptime   Route
r-Mac       
--------- --------------------------------------  ----- --------- -------- -----
------------
nve1      172.16.0.11                             Up    CP        02:07:43 5002.
8600.1b08   

Leaf2# sh nve interface nve 1 detai
Interface: nve1, State: Up, encapsulation: VXLAN
 VPC Capability: VPC-VIP-Only [not-notified]
 Local Router MAC: 5002.8400.1b08
 Host Learning Mode: Control-Plane
 Source-Interface: loopback0 (primary: 172.16.0.12, secondary: 0.0.0.0)
 Source Interface State: Up
 Virtual RMAC Advertisement: No
 NVE Flags: 
 Interface Handle: 0x49000001
 Source Interface hold-down-time: 180
 Source Interface hold-up-time: 30
 Remaining hold-down time: 0 seconds
 Virtual Router MAC: N/A
 Interface state: nve-intf-add-complete

Leaf2# sh mac address-table 
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
C   10     0000.0000.0011   dynamic  0         F      F    nve1(172.16.0.11)
*   10     0000.0000.0013   dynamic  0         F      F    Eth1/1
*   77     5002.8400.1b08   static   -         F      F    Vlan77
*   77     5002.8600.1b08   static   -         F      F    nve1(172.16.0.11)
G    -     0001.0001.0001   static   -         F      F    sup-eth1(R)
G    -     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G   10     5002.8400.1b08   static   -         F      F    sup-eth1(R)
G   77     5002.8400.1b08   static   -         F      F    sup-eth1(R)
Leaf2# sh ip route vrf vrF_A 
IP Route Table for VRF "VRF_A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.0.0.0/24, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 01:25:33, direct
10.0.0.11/32, ubest/mbest: 1/0
    *via 172.16.0.11%default, [200/0], 01:27:42, bgp-65501, internal, tag 65501,
 segid: 770000 tunnelid: 0xac10000b encap: VXLAN
 
10.0.0.13/32, ubest/mbest: 1/0, attached
    *via 10.0.0.13, Vlan10, [190/0], 01:25:19, hmm
10.0.0.254/32, ubest/mbest: 1/0, attached
    *via 10.0.0.254, Vlan10, [0/0], 01:25:33, local
20.0.0.12/32, ubest/mbest: 1/0
    *via 172.16.0.11%default, [200/0], 01:27:42, bgp-65501, internal, tag 65501,
 segid: 770000 tunnelid: 0xac10000b encap: VXLAN

=================================================================

Client1#ping 10.0.0.13
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.13, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 5/5/7 ms

Client1#ping 20.0.0.12
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 20.0.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 5/5/8 ms

Client1#sh ip arp 
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  10.0.0.11               -   0000.0000.0011  ARPA   GigabitEthernet0/0
Internet  10.0.0.13              10   0000.0000.0013  ARPA   GigabitEthernet0/0
Internet  10.0.0.254             18   0001.0001.0001  ARPA   GigabitEthernet0/0

Client2#ping 10.0.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.11, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 64/91/119 ms

Client2#ping 10.0.0.13
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.13, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 149/322/484 ms

Client2#sh ip arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  20.0.0.12               -   0000.0000.0012  ARPA   GigabitEthernet0/0
Internet  20.0.0.254              1   0001.0001.0001  ARPA   GigabitEthernet0/0

Client3#ping 10.0.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.11, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 7/8/13 ms

Client3#ping 20.0.0.12
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 20.0.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 10/11/15 ms

Client3#sh ip arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  10.0.0.11              13   0000.0000.0011  ARPA   GigabitEthernet0/0
Internet  10.0.0.13               -   0000.0000.0013  ARPA   GigabitEthernet0/0
Internet  10.0.0.254             14   0001.0001.0001  ARPA   GigabitEthernet0/0
