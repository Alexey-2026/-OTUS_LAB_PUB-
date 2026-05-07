# Настройка VxLAN L2VNI


## Цель работы
Настроить Overlay на основе VxLAN EVPN для L2 связанности между клиентами.

<!-- <img src="https://github.com/user-attachments/assets/dbbdefb0-dc44-4500-b74f-02a4d5f61335" width="500" style="max-width: 100%; height: auto;">-->
<img src="https://github.com/user-attachments/assets/6315567c-1512-4870-bc4c-68dd34bd23a2" width="300" style="max-width: 100%; height: auto;">

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
| Leaf-2 | 10 | 10.0.0.0/24 | 10.0.0.254 |

---

## Конфигурация оборудования (VXLAN L2VNI)

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
vlan 1,10
vlan 10
  name FOR_CLIENT
  vn-segment 10000
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
!
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  global ingress-replication protocol bgp
  member vni 10000
!
interface Loopback0
  ip address 172.16.0.11 255.255.255.255
!
interface Ethernet1/6
  description Link_To_Spine1
  bfd
  no ip redirects
  ip address 172.16.10.0/31
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
vlan 1,10
vlan 10
  name FOR_CLIENT
  vn-segment 10000
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
!
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  global ingress-replication protocol bgp
  member vni 10000
!
interface Loopback0
ip address 172.16.0.12 255.255.255.255
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

Spine1# show bgp l2vpn evpn summary 
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 172.16.0.111, local AS number 65501
BGP table version is 40, L2VPN EVPN config peers 3, capable peers 2
4 network entries and 4 paths using 976 bytes of memory
BGP attribute entries [3/516], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.0     4 65501    1703    1697       40    0    0 01:25:16 2         
172.16.10.2     4 65501    2115    2107       40    0    0 01:45:46 2         
Spine1# show bgp l2vpn evpn 
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 40, Local Router ID is 172.16.0.111
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.11                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.11]/88
                      172.16.0.11                       100          0 i

Route Distinguisher: 172.16.0.12:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.12                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.12]/88
                      172.16.0.12                       100          0 i

Spine1# show bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216,
 version 29
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

Spine1# show bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216,
 version 30
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
172.16.10.7     4 65501   26474   26420        0    0    0 00:42:47 Idle
    
Leaf1# sh bgp l2vpn evpn

BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 48, Local Router ID is 172.16.0.11
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.11                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.12                       100          0 i
*>l[3]:[0]:[32]:[172.16.0.11]/88
                      172.16.0.11                       100      32768 i
*>i[3]:[0]:[32]:[172.16.0.12]/88
                      172.16.0.12                       100          0 i

Route Distinguisher: 172.16.0.12:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.12                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.12]/88
                      172.16.0.12                       100          0 i

Leaf1# sh bgp l2vpn evpn 0000.0000.0011

BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216,
 version 25
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

Leaf1# sh bgp l2vpn evpn 0000.0000.0012

BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216,
 version 48
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 172.16.0.12:32777:[2]:[0]:[0]:[48]:[0000.0000.0012]:[
0]:[0.0.0.0]/216 
  AS-Path: NONE, path sourced internal to AS
    172.16.0.12 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10000
      Extcommunity: RT:65501:10000 ENCAP:8
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path-id 1 not advertised to any peer

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216,
 version 46
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

Leaf1# sh bgp l2vpn evpn route-type 3

BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777    (L2VNI 10000)
BGP routing table entry for [3]:[0]:[32]:[172.16.0.11]/88, version 5
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
BGP routing table entry for [3]:[0]:[32]:[172.16.0.12]/88, version 47
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

Route Distinguisher: 172.16.0.12:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.12]/88, version 45
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
nve1      172.16.0.12                             Up    CP        01:29:21 n/a  

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
C   10     0000.0000.0012   dynamic  0         F      F    nve1(172.16.0.12)
G    -     5002.8600.1b08   static   -         F      F    sup-eth1(R)

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
172.16.10.9     4 65501  105982  105944        0    0    0 00:55:05 Idle     
Leaf2# 
Leaf2# 
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
172.16.10.9     4 65501  105982  105944        0    0    0 00:55:34 Idle     
Leaf2# sh bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 42, Local Router ID is 172.16.0.12
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 172.16.0.11:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.11                       100          0 i
*>i[3]:[0]:[32]:[172.16.0.11]/88
                      172.16.0.11                       100          0 i

Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.16.0.11                       100          0 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.16.0.12                       100      32768 i
*>i[3]:[0]:[32]:[172.16.0.11]/88
                      172.16.0.11                       100          0 i
*>l[3]:[0]:[32]:[172.16.0.12]/88
                      172.16.0.12                       100      32768 i

Leaf2# sh bgp l2vpn evpn 0000.0000.0011
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216,
 version 40
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

Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216,
 version 42
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

Leaf2# sh bgp l2vpn evpn 0000.0000.0012
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.12:32777    (L2VNI 10000)
BGP routing table entry for [2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216,
 version 26
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

Leaf2# sh bgp l2vpn evpn route-type 3
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 172.16.0.11:32777
BGP routing table entry for [3]:[0]:[32]:[172.16.0.11]/88, version 39
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
BGP routing table entry for [3]:[0]:[32]:[172.16.0.11]/88, version 41
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
nve1      172.16.0.11                             Up    CP        01:41:19 n/a  
            

Leaf2# 
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
*   10     0000.0000.0012   dynamic  0         F      F    Eth1/1
G    -     5002.8400.1b08   static   -         F      F    sup-eth1(R)

=================================================================

Client1#ping 10.0.0.12
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 211/320/491 ms

Client1#sh ip arp 
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  10.0.0.11               -   0000.0000.0011  ARPA   GigabitEthernet0/0
Internet  10.0.0.12              91   0000.0000.0012  ARPA   GigabitEthernet0/0

Client2#ping 10.0.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.11, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 200/289/512 ms

Client2#sh ip arp 
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  10.0.0.11              91   0000.0000.0011  ARPA   GigabitEthernet0/0
Internet  10.0.0.12               -   0000.0000.0012  ARPA   GigabitEthernet0/0
```
