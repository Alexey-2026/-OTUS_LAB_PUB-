# Настройка iBGP для Underlay сети

## Цель работы
Настроить iBGP для Underlay сети.

<img src="https://github.com/user-attachments/assets/dbbdefb0-dc44-4500-b74f-02a4d5f61335" width="500" style="max-width: 100%; height: auto;">

## План адресации

### Underlay сеть (линки /31)

| Линк | Spine интерфейс | IP Spine | Leaf интерфейс | IP Leaf |
|------|----------------|----------|----------------|---------|
| Spine1-Leaf1 | Eth1/1 | 172.16.10.1 | Eth1/6 | 172.16.10.0 |
| Spine1-Leaf2 | Eth1/2 | 172.16.10.3 | Eth1/6 | 172.16.10.2 |
| Spine1-Leaf3 | Eth1/3 | 172.16.10.5 | Eth1/6 | 172.16.10.4 |
| Spine2-Leaf1 | Eth1/1 | 172.16.10.7 | Eth1/7 | 172.16.10.6 |
| Spine2-Leaf2 | Eth1/2 | 172.16.10.9 | Eth1/7 | 172.16.10.8 |
| Spine2-Leaf3 | Eth1/3 | 172.16.10.11 | Eth1/7 | 172.16.10.10 |

### Loopback адреса

| Устройство | Loopback0 IP |
|------------|--------------|
| Spine-1 | 172.16.0.111/32 |
| Spine-2 | 172.16.0.112/32 |
| Leaf-1 | 172.16.0.11/32 |
| Leaf-2 | 172.16.0.12/32 |
| Leaf-3 | 172.16.0.13/32 |

### Серверные подсети

| Leaf | VLAN | Подсеть | Шлюз |
|------|------|---------|------|
| Leaf-1 | 10 | 192.168.1.0/24 | 192.168.1.254 |
| Leaf-2 | 10 | 192.168.1.0/24 | 192.168.1.254 |
| Leaf-3 | 10 | 192.168.1.0/24 | 192.168.1.254 |
| Leaf-3 | 20 | 192.168.2.0/24 | 192.168.2.254 |

---

## Конфигурация оборудования (iBGP)

### Spine-1
```
hostname Spine1
!
feature bgp
feature bfd
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
interface Ethernet1/3
  description Link_To_Leaf2
  bfd
  no ip redirects
  ip address 172.16.10.5/31
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
    timers 3 9
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
      send-community
      send-community extended
      route-reflector-client
      next-hop-self all
  neighbor 172.16.10.0
    inherit peer LEAFs
    description To_Leaf1
  neighbor 172.16.10.2
    inherit peer LEAFs
    description To_Leaf2
  neighbor 172.16.10.4
    inherit peer LEAFs
    description To_Leaf3
!

text
```
### Spine-2
```
hostname Spine2
!
feature bgp
feature bfd
!
bfd interval 100 min_rx 100 multiplier 5
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
!
interface Loopback0
ip address 172.16.0.112 255.255.255.255
!
interface Ethernet1/1
  description Link_To_Leaf1
  bfd
  no ip redirects
  ip address 172.16.10.7/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/2
  description Link_To_Leaf2
  bfd
  no ip redirects
  ip address 172.16.10.9/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/3
  description Link_To_Leaf3
  bfd
  no ip redirects
  ip address 172.16.10.11/31
  no ipv6 redirects
  no shutdown
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
    timers 3 9
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
      send-community
      send-community extended
      route-reflector-client
      next-hop-self all
  neighbor 172.16.10.6
    inherit peer LEAFs
    description To_Leaf1
  neighbor 172.16.10.8
    inherit peer LEAFs
    description To_Leaf2
  neighbor 172.16.10.10
    inherit peer LEAFs
    description To_Leaf3
!

```

### Leaf-1
```
hostname Leaf1
!
feature bgp
feature bfd
feature interface-vlan
!
bfd interval 100 min_rx 100 multiplier 5
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
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
interface Ethernet1/7
  description Link_To_Spine2
  bfd
  no ip redirects
  ip address 172.16.10.6/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/1
description Server_Network
switchport
switchport access vlan 10
no shutdown
!
interface Vlan10
description Server_Network
ip address 192.168.1.254 255.255.255.0
no shutdown
!
router bgp 65501
  router-id 172.16.0.11
  address-family ipv4 unicast
    redistribute direct route-map RM_RED_FOR_BGP
    maximum-paths 4
    maximum-paths ibgp 4
  template peer SPINEs
    bfd
    remote-as 65501
    timers 3 9
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
      send-community
      send-community extended
  neighbor 172.16.10.1
    inherit peer SPINEs
    description To_Spine1
  neighbor 172.16.10.7
    inherit peer SPINEs
    description To_Spine2
!
```

### Leaf-2
```
hostname Leaf2
!
feature bgp
feature bfd
feature interface-vlan
!
bfd interval 100 min_rx 100 multiplier 5
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
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

interface Ethernet1/7
  description Link_To_Spine2
  bfd
  no ip redirects
  ip address 172.16.10.8/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/1
description Server_Network
switchport
switchport access vlan 10
no shutdown
!
interface Vlan10
description Server_Network
ip address 192.168.1.254 255.255.255.0
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
    timers 3 9
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
      send-community
      send-community extended
  neighbor 172.16.10.3
    inherit peer SPINEs
    description ---To_Spine1---
  neighbor 172.16.10.9
    inherit peer SPINEs
    description ---To_Spine2---
!
```

### Leaf-3
```
hostname Leaf3
!
feature bgp
feature bfd
feature interface-vlan
!
bfd interval 100 min_rx 100 multiplier 5
!
route-map RM_RED_FOR_BGP permit 10
  match interface loopback0 
!
interface Loopback0
ip address 172.16.0.13 255.255.255.255
!
interface Ethernet1/6
  description Link_To_Spine1
  bfd
  no ip redirects
  ip address 172.16.10.4/31
  no ipv6 redirects
  no shutdown

interface Ethernet1/7
  description Link_To_Spine2
  bfd
  no ip redirects
  ip address 172.16.10.10/31
  no ipv6 redirects
  no shutdown
!
interface Ethernet1/1
description Server_Network
switchport
switchport access vlan 10
no shutdown
!
interface Ethernet1/2
description Server_Network
switchport
switchport access vlan 20
no shutdown
!
interface Vlan10
description Server_Network
ip address 192.168.1.254 255.255.255.0
no shutdown
!
interface Vlan20
description Server_Network
ip address 192.168.2.254 255.255.255.0
no shutdown
!
router bgp 65501
  router-id 172.16.0.13
  address-family ipv4 unicast
    redistribute direct route-map RM_RED_FOR_BGP
    maximum-paths 4
    maximum-paths ibgp 4
  template peer SPINEs
    bfd
    remote-as 65501
    timers 3 9
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
      send-community
      send-community extended
  neighbor 172.16.10.5
    inherit peer SPINEs
    description ----To_Spine1----
  neighbor 172.16.10.11
    inherit peer SPINEs
    description ----To_Spine2----
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
    *via 172.16.10.0, [200/0], 00:18:18, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 1/0
    *via 172.16.10.2, [200/0], 00:53:08, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 1/0
    *via 172.16.10.4, [200/0], 00:52:44, bgp-65501, internal, tag 65501
172.16.0.111/32, ubest/mbest: 2/0, attached
    *via 172.16.0.111, Lo0, [0/0], 06:46:51, local
    *via 172.16.0.111, Lo0, [0/0], 06:46:51, direct
172.16.10.0/31, ubest/mbest: 1/0, attached
    *via 172.16.10.1, Eth1/1, [0/0], 00:53:10, direct
172.16.10.1/32, ubest/mbest: 1/0, attached
    *via 172.16.10.1, Eth1/1, [0/0], 00:53:10, local
172.16.10.2/31, ubest/mbest: 1/0, attached
    *via 172.16.10.3, Eth1/2, [0/0], 00:53:10, direct
172.16.10.3/32, ubest/mbest: 1/0, attached
    *via 172.16.10.3, Eth1/2, [0/0], 00:53:10, local
172.16.10.4/31, ubest/mbest: 1/0, attached
    *via 172.16.10.5, Eth1/3, [0/0], 00:53:10, direct
172.16.10.5/32, ubest/mbest: 1/0, attached
    *via 172.16.10.5, Eth1/3, [0/0], 00:53:10, local
  
Spine1# sh bgp ipv4 unicast summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.111, local AS number 65501
BGP table version is 9, IPv4 Unicast config peers 3, capable peers 3
4 network entries and 4 paths using 976 bytes of memory
BGP attribute entries [3/516], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.0     4 65501    1071    1068        9    0    0 00:53:28 1         
172.16.10.2     4 65501    1071    1069        9    0    0 00:53:28 1         
172.16.10.4     4 65501    1062    1062        9    0    0 00:53:05 1         
Spine1# sh bgp ipv4 unicast 
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 9, Local Router ID is 172.16.0.111
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>i172.16.0.11/32     172.16.10.0              0        100          0 ?
*>i172.16.0.12/32     172.16.10.2              0        100          0 ?
*>i172.16.0.13/32     172.16.10.4              0        100          0 i
*>r172.16.0.111/32    0.0.0.0                  0        100      32768 ?
   
Spine1# sh bgp ipv4 unicast detail 
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 172.16.0.11/32, version 9
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.0 (metric 0) from 172.16.10.0 (172.16.0.11)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 advertised to peers:
    172.16.10.2        172.16.10.4    
BGP routing table entry for 172.16.0.12/32, version 6
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.2 (metric 0) from 172.16.10.2 (172.16.0.12)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.4    
BGP routing table entry for 172.16.0.13/32, version 8
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.4 (metric 0) from 172.16.10.4 (172.16.0.13)
      Origin IGP, MED 0, localpref 100, weight 0

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2    
BGP routing table entry for 172.16.0.111/32, version 2
Paths: (1 available, best #1)
Flags: (0x080002) (high32 00000000) on xmit-list, is not in urib
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: redist, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    0.0.0.0 (metric 0) from 0.0.0.0 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 32768

  Path-id 1 advertised to peers:
    172.16.10.0        172.16.10.2        172.16.10.4    

Spine1#   ping 172.16.0.11
PING 172.16.0.11 (172.16.0.11): 56 data bytes
64 bytes from 172.16.0.11: icmp_seq=0 ttl=254 time=219.098 ms
64 bytes from 172.16.0.11: icmp_seq=1 ttl=254 time=97.474 ms
64 bytes from 172.16.0.11: icmp_seq=2 ttl=254 time=128.236 ms
64 bytes from 172.16.0.11: icmp_seq=3 ttl=254 time=256.094 ms
64 bytes from 172.16.0.11: icmp_seq=4 ttl=254 time=141.361 ms

--- 172.16.0.11 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 97.474/168.452/256.094 ms
Spine1#   ping 172.16.0.12
PING 172.16.0.12 (172.16.0.12): 56 data bytes
64 bytes from 172.16.0.12: icmp_seq=0 ttl=254 time=112.254 ms
64 bytes from 172.16.0.12: icmp_seq=1 ttl=254 time=107.02 ms
64 bytes from 172.16.0.12: icmp_seq=2 ttl=254 time=147.734 ms
64 bytes from 172.16.0.12: icmp_seq=3 ttl=254 time=206.65 ms
64 bytes from 172.16.0.12: icmp_seq=4 ttl=254 time=141.872 ms

--- 172.16.0.12 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 107.02/143.105/206.65 ms
Spine1#   ping 172.16.0.13
PING 172.16.0.13 (172.16.0.13): 56 data bytes
64 bytes from 172.16.0.13: icmp_seq=0 ttl=254 time=111.375 ms
64 bytes from 172.16.0.13: icmp_seq=1 ttl=254 time=77.11 ms
64 bytes from 172.16.0.13: icmp_seq=2 ttl=254 time=6.181 ms
64 bytes from 172.16.0.13: icmp_seq=3 ttl=254 time=5.363 ms
64 bytes from 172.16.0.13: icmp_seq=4 ttl=254 time=3.743 ms

--- 172.16.0.13 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 3.743/40.754/111.375 ms

=======================================================================

Spine2# sh ip route 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

172.16.0.11/32, ubest/mbest: 1/0
    *via 172.16.10.6, [200/0], 00:23:03, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 1/0
    *via 172.16.10.8, [200/0], 07:09:20, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 1/0
    *via 172.16.10.10, [200/0], 07:09:17, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 2/0, attached
    *via 172.16.0.112, Lo0, [0/0], 17:14:38, local
    *via 172.16.0.112, Lo0, [0/0], 17:14:38, direct
172.16.10.6/31, ubest/mbest: 1/0, attached
    *via 172.16.10.7, Eth1/1, [0/0], 07:09:32, direct
172.16.10.7/32, ubest/mbest: 1/0, attached
    *via 172.16.10.7, Eth1/1, [0/0], 07:09:32, local
172.16.10.8/31, ubest/mbest: 1/0, attached
    *via 172.16.10.9, Eth1/2, [0/0], 07:09:29, direct
172.16.10.9/32, ubest/mbest: 1/0, attached
    *via 172.16.10.9, Eth1/2, [0/0], 07:09:29, local
172.16.10.10/31, ubest/mbest: 1/0, attached
    *via 172.16.10.11, Eth1/3, [0/0], 07:09:29, direct
172.16.10.11/32, ubest/mbest: 1/0, attached
    *via 172.16.10.11, Eth1/3, [0/0], 07:09:29, local

Spine2# sh bgp ipv4 unicast summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.112, local AS number 65501
BGP table version is 65, IPv4 Unicast config peers 3, capable peers 3
4 network entries and 4 paths using 976 bytes of memory
BGP attribute entries [3/516], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.6     4 65501    1035    1044       65    0    0 16:55:12 1         
172.16.10.8     4 65501   14468   14689       65    0    0 07:10:02 1         
172.16.10.10    4 65501   11840   11858       65    0    0 07:09:59 1         
Spine2# sh bgp ipv4 unicast 
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 65, Local Router ID is 172.16.0.112
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>i172.16.0.11/32     172.16.10.6              0        100          0 ?
*>i172.16.0.12/32     172.16.10.8              0        100          0 ?
*>i172.16.0.13/32     172.16.10.10             0        100          0 i
*>r172.16.0.112/32    0.0.0.0                  0        100      32768 ?

Spine2# sh bgp ipv4 unicast detail 
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 172.16.0.11/32, version 65
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.6 (metric 0) from 172.16.10.6 (172.16.0.11)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 advertised to peers:
    172.16.10.8        172.16.10.10   
BGP routing table entry for 172.16.0.12/32, version 55
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.8 (metric 0) from 172.16.10.8 (172.16.0.12)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.10   
BGP routing table entry for 172.16.0.13/32, version 58
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.10 (metric 0) from 172.16.10.10 (172.16.0.13)
      Origin IGP, MED 0, localpref 100, weight 0

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8    
BGP routing table entry for 172.16.0.112/32, version 10
Paths: (1 available, best #1)
Flags: (0x080002) (high32 00000000) on xmit-list, is not in urib
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: redist, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    0.0.0.0 (metric 0) from 0.0.0.0 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 32768

  Path-id 1 advertised to peers:
    172.16.10.6        172.16.10.8        172.16.10.10   

Spine2#  ping 172.16.0.11
PING 172.16.0.11 (172.16.0.11): 56 data bytes
64 bytes from 172.16.0.11: icmp_seq=0 ttl=254 time=194.892 ms
64 bytes from 172.16.0.11: icmp_seq=1 ttl=254 time=285.772 ms
64 bytes from 172.16.0.11: icmp_seq=2 ttl=254 time=139.709 ms
64 bytes from 172.16.0.11: icmp_seq=3 ttl=254 time=170.43 ms
64 bytes from 172.16.0.11: icmp_seq=4 ttl=254 time=124.601 ms

--- 172.16.0.11 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 124.601/183.08/285.772 ms
Spine2#  ping 172.16.0.12
PING 172.16.0.12 (172.16.0.12): 56 data bytes
64 bytes from 172.16.0.12: icmp_seq=0 ttl=254 time=66.309 ms
64 bytes from 172.16.0.12: icmp_seq=1 ttl=254 time=84.5 ms
64 bytes from 172.16.0.12: icmp_seq=2 ttl=254 time=50.368 ms
64 bytes from 172.16.0.12: icmp_seq=3 ttl=254 time=59.205 ms
64 bytes from 172.16.0.12: icmp_seq=4 ttl=254 time=37.275 ms

--- 172.16.0.12 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 37.275/59.531/84.5 ms
Spine2#  ping 172.16.0.13
PING 172.16.0.13 (172.16.0.13): 56 data bytes
64 bytes from 172.16.0.13: icmp_seq=0 ttl=254 time=118.03 ms
64 bytes from 172.16.0.13: icmp_seq=1 ttl=254 time=175.168 ms
64 bytes from 172.16.0.13: icmp_seq=2 ttl=254 time=154.819 ms
64 bytes from 172.16.0.13: icmp_seq=3 ttl=254 time=215.572 ms
64 bytes from 172.16.0.13: icmp_seq=4 ttl=254 time=90.482 ms

--- 172.16.0.13 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 90.482/150.814/215.572 ms

========================================================================

Leaf1# sh ip route 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

172.16.0.11/32, ubest/mbest: 2/0, attached
    *via 172.16.0.11, Lo0, [0/0], 17:04:28, local
    *via 172.16.0.11, Lo0, [0/0], 17:04:28, direct
172.16.0.12/32, ubest/mbest: 2/0
    *via 172.16.10.1, [200/0], 01:01:39, bgp-65501, internal, tag 65501
    *via 172.16.10.7, [200/0], 00:59:37, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 2/0
    *via 172.16.10.1, [200/0], 01:01:15, bgp-65501, internal, tag 65501
    *via 172.16.10.7, [200/0], 00:59:37, bgp-65501, internal, tag 65501
172.16.0.111/32, ubest/mbest: 1/0
    *via 172.16.10.1, [200/0], 01:01:39, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 1/0
    *via 172.16.10.7, [200/0], 08:12:36, bgp-65501, internal, tag 65501
172.16.10.0/31, ubest/mbest: 1/0, attached
    *via 172.16.10.0, Eth1/6, [0/0], 17:01:36, direct
172.16.10.0/32, ubest/mbest: 1/0, attached
    *via 172.16.10.0, Eth1/6, [0/0], 17:01:36, local
172.16.10.6/31, ubest/mbest: 1/0, attached
    *via 172.16.10.6, Eth1/7, [0/0], 17:01:27, direct
172.16.10.6/32, ubest/mbest: 1/0, attached
    *via 172.16.10.6, Eth1/7, [0/0], 17:01:27, local

Leaf1# sh bgp ipv4 unicast summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.11, local AS number 65501
BGP table version is 70, IPv4 Unicast config peers 2, capable peers 2
5 network entries and 7 paths using 1460 bytes of memory
BGP attribute entries [6/1032], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [4/16]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.1     4 65501    1894    1870       70    0    0 01:02:06 3         
172.16.10.7     4 65501    1081    1031       70    0    0 16:58:43 3         
Leaf1# sh bgp ipv4 unicast 
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 70, Local Router ID is 172.16.0.11
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>r172.16.0.11/32     0.0.0.0                  0        100      32768 ?
*>i172.16.0.12/32     172.16.10.1              0        100          0 ?
*|i                   172.16.10.7              0        100          0 ?
*>i172.16.0.13/32     172.16.10.1              0        100          0 i
*|i                   172.16.10.7              0        100          0 i
*>i172.16.0.111/32    172.16.10.1              0        100          0 ?
*>i172.16.0.112/32    172.16.10.7              0        100          0 ?

Leaf1# sh bgp ipv4 unicast detail 
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 172.16.0.11/32, version 70
Paths: (1 available, best #1)
Flags: (0x080002) (high32 00000000) on xmit-list, is not in urib
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: redist, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    0.0.0.0 (metric 0) from 0.0.0.0 (172.16.0.11)
      Origin incomplete, MED 0, localpref 100, weight 32768

  Path-id 1 advertised to peers:
    172.16.10.1        172.16.10.7    
BGP routing table entry for 172.16.0.12/32, version 68
Paths: (2 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.1 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, multipa
th, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.7 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.13/32, version 69
Paths: (2 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.1 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin IGP, MED 0, localpref 100, weight 0
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, multipa
th, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.7 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin IGP, MED 0, localpref 100, weight 0
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.111/32, version 65
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.1 (metric 0) from 172.16.10.1 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.112/32, version 53
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.7 (metric 0) from 172.16.10.7 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 not advertised to any peer

Leaf1#   ping 172.16.0.111
PING 172.16.0.111 (172.16.0.111): 56 data bytes
64 bytes from 172.16.0.111: icmp_seq=0 ttl=254 time=139.747 ms
64 bytes from 172.16.0.111: icmp_seq=1 ttl=254 time=113.859 ms
64 bytes from 172.16.0.111: icmp_seq=2 ttl=254 time=97.13 ms
64 bytes from 172.16.0.111: icmp_seq=3 ttl=254 time=187.082 ms
64 bytes from 172.16.0.111: icmp_seq=4 ttl=254 time=266.911 ms

--- 172.16.0.111 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 97.13/160.945/266.911 ms
Leaf1#   ping 172.16.0.112
PING 172.16.0.112 (172.16.0.112): 56 data bytes
64 bytes from 172.16.0.112: icmp_seq=0 ttl=254 time=92.307 ms
64 bytes from 172.16.0.112: icmp_seq=1 ttl=254 time=8.943 ms
64 bytes from 172.16.0.112: icmp_seq=2 ttl=254 time=5.339 ms
64 bytes from 172.16.0.112: icmp_seq=3 ttl=254 time=6.44 ms
64 bytes from 172.16.0.112: icmp_seq=4 ttl=254 time=9.725 ms

--- 172.16.0.112 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 5.339/24.55/92.307 ms
Leaf1# ping 172.16.0.12 source-interface loopback 0
PING 172.16.0.12 (172.16.0.12): 56 data bytes
64 bytes from 172.16.0.12: icmp_seq=0 ttl=253 time=9.591 ms
64 bytes from 172.16.0.12: icmp_seq=1 ttl=253 time=5.876 ms
64 bytes from 172.16.0.12: icmp_seq=2 ttl=253 time=4.279 ms
64 bytes from 172.16.0.12: icmp_seq=3 ttl=253 time=6.189 ms
64 bytes from 172.16.0.12: icmp_seq=4 ttl=253 time=4.008 ms

--- 172.16.0.12 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 4.008/5.988/9.591 ms
Leaf1# ping 172.16.0.13 source-interface loopback 0
PING 172.16.0.13 (172.16.0.13): 56 data bytes
64 bytes from 172.16.0.13: icmp_seq=0 ttl=253 time=14.726 ms
64 bytes from 172.16.0.13: icmp_seq=1 ttl=253 time=5.342 ms
64 bytes from 172.16.0.13: icmp_seq=2 ttl=253 time=4.558 ms
64 bytes from 172.16.0.13: icmp_seq=3 ttl=253 time=5.355 ms
64 bytes from 172.16.0.13: icmp_seq=4 ttl=253 time=7.16 ms

--- 172.16.0.13 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 4.558/7.428/14.726 ms

=============================================================================

Leaf2# sh ip route 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

172.16.0.11/32, ubest/mbest: 2/0
    *via 172.16.10.3, [200/0], 00:32:24, bgp-65501, internal, tag 65501
    *via 172.16.10.9, [200/0], 00:32:24, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 2/0, attached
    *via 172.16.0.12, Lo0, [0/0], 16:35:17, local
    *via 172.16.0.12, Lo0, [0/0], 16:35:17, direct
172.16.0.13/32, ubest/mbest: 2/0
    *via 172.16.10.3, [200/0], 01:06:50, bgp-65501, internal, tag 65501
    *via 172.16.10.9, [200/0], 01:05:12, bgp-65501, internal, tag 65501
172.16.0.111/32, ubest/mbest: 1/0
    *via 172.16.10.3, [200/0], 01:07:14, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 1/0
    *via 172.16.10.9, [200/0], 07:18:41, bgp-65501, internal, tag 65501
172.16.10.2/31, ubest/mbest: 1/0, attached
    *via 172.16.10.2, Eth1/6, [0/0], 16:32:55, direct
172.16.10.2/32, ubest/mbest: 1/0, attached
    *via 172.16.10.2, Eth1/6, [0/0], 16:32:55, local
172.16.10.8/31, ubest/mbest: 1/0, attached
    *via 172.16.10.8, Eth1/7, [0/0], 16:32:45, direct
172.16.10.8/32, ubest/mbest: 1/0, attached
    *via 172.16.10.8, Eth1/7, [0/0], 16:32:45, local
192.168.1.0/24, ubest/mbest: 1/0, attached
    *via 192.168.1.254, Vlan10, [0/0], 12:00:19, direct
192.168.1.254/32, ubest/mbest: 1/0, attached
    *via 192.168.1.254, Vlan10, [0/0], 12:00:19, local

Leaf2# sh bgp ipv4 unicast summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.12, local AS number 65501
BGP table version is 33, IPv4 Unicast config peers 2, capable peers 2
5 network entries and 7 paths using 1460 bytes of memory
BGP attribute entries [6/1032], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [4/16]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.3     4 65501    7432    7410       33    0    0 01:07:31 3         
172.16.10.9     4 65501   14691   14649       33    0    0 07:18:59 3         
Leaf2# sh bgp ipv4 unicast 
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 33, Local Router ID is 172.16.0.12
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>i172.16.0.11/32     172.16.10.3              0        100          0 ?
*|i                   172.16.10.9              0        100          0 ?
*>r172.16.0.12/32     0.0.0.0                  0        100      32768 ?
*>i172.16.0.13/32     172.16.10.3              0        100          0 i
*|i                   172.16.10.9              0        100          0 i
*>i172.16.0.111/32    172.16.10.3              0        100          0 ?
*>i172.16.0.112/32    172.16.10.9              0        100          0 ?

Leaf2# sh bgp ipv4 unicast detail 
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 172.16.0.11/32, version 33
Paths: (2 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.3 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, multipa
th, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.9 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.12/32, version 4
Paths: (1 available, best #1)
Flags: (0x080002) (high32 00000000) on xmit-list, is not in urib
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: redist, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    0.0.0.0 (metric 0) from 0.0.0.0 (172.16.0.12)
      Origin incomplete, MED 0, localpref 100, weight 32768

  Path-id 1 advertised to peers:
    172.16.10.3        172.16.10.9    
BGP routing table entry for 172.16.0.13/32, version 32
Paths: (2 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.3 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin IGP, MED 0, localpref 100, weight 0
      Originator: 172.16.0.13 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, multipa
th, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.9 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin IGP, MED 0, localpref 100, weight 0
      Originator: 172.16.0.13 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.111/32, version 28
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.3 (metric 0) from 172.16.10.3 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.112/32, version 24
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.9 (metric 0) from 172.16.10.9 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 not advertised to any peer

Leaf2# ping 172.16.0.111
PING 172.16.0.111 (172.16.0.111): 56 data bytes
64 bytes from 172.16.0.111: icmp_seq=0 ttl=254 time=3.715 ms
64 bytes from 172.16.0.111: icmp_seq=1 ttl=254 time=2.387 ms
64 bytes from 172.16.0.111: icmp_seq=2 ttl=254 time=2.647 ms
64 bytes from 172.16.0.111: icmp_seq=3 ttl=254 time=2.164 ms
64 bytes from 172.16.0.111: icmp_seq=4 ttl=254 time=2.604 ms

--- 172.16.0.111 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 2.164/2.703/3.715 ms
Leaf2# ping 172.16.0.112
PING 172.16.0.112 (172.16.0.112): 56 data bytes
64 bytes from 172.16.0.112: icmp_seq=0 ttl=254 time=82.702 ms
64 bytes from 172.16.0.112: icmp_seq=1 ttl=254 time=66.199 ms
64 bytes from 172.16.0.112: icmp_seq=2 ttl=254 time=193.48 ms
64 bytes from 172.16.0.112: icmp_seq=3 ttl=254 time=164.573 ms
64 bytes from 172.16.0.112: icmp_seq=4 ttl=254 time=144.444 ms

--- 172.16.0.112 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 66.199/130.279/193.48 ms
  
Leaf2# ping 172.16.0.11 source-interface loopback 0
PING 172.16.0.11 (172.16.0.11): 56 data bytes
64 bytes from 172.16.0.11: icmp_seq=0 ttl=253 time=221.89 ms
64 bytes from 172.16.0.11: icmp_seq=1 ttl=253 time=158.7 ms
64 bytes from 172.16.0.11: icmp_seq=2 ttl=253 time=145.863 ms
64 bytes from 172.16.0.11: icmp_seq=3 ttl=253 time=197.294 ms
64 bytes from 172.16.0.11: icmp_seq=4 ttl=253 time=165.963 ms

--- 172.16.0.11 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 145.863/177.941/221.89 ms

Leaf2# ping 172.16.0.13 source-interface loopback 0
PING 172.16.0.13 (172.16.0.13): 56 data bytes
64 bytes from 172.16.0.13: icmp_seq=0 ttl=253 time=340.606 ms
64 bytes from 172.16.0.13: icmp_seq=1 ttl=253 time=400.169 ms
64 bytes from 172.16.0.13: icmp_seq=2 ttl=253 time=375.463 ms
64 bytes from 172.16.0.13: icmp_seq=3 ttl=253 time=513.062 ms
64 bytes from 172.16.0.13: icmp_seq=4 ttl=253 time=228.857 ms

--- 172.16.0.13 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 228.857/371.631/513.062 ms

======================================================================
Leaf3# sh ip route 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

172.16.0.11/32, ubest/mbest: 2/0
    *via 172.16.10.5, [200/0], 00:21:47, bgp-65501, internal, tag 65501
    *via 172.16.10.11, [200/0], 00:21:47, bgp-65501, internal, tag 65501
172.16.0.12/32, ubest/mbest: 2/0
    *via 172.16.10.5, [200/0], 00:21:47, bgp-65501, internal, tag 65501
    *via 172.16.10.11, [200/0], 00:21:47, bgp-65501, internal, tag 65501
172.16.0.13/32, ubest/mbest: 2/0, attached
    *via 172.16.0.13, Lo0, [0/0], 09:12:33, local
    *via 172.16.0.13, Lo0, [0/0], 09:12:33, direct
172.16.0.111/32, ubest/mbest: 1/0
    *via 172.16.10.5, [200/0], 00:21:47, bgp-65501, internal, tag 65501
172.16.0.112/32, ubest/mbest: 1/0
    *via 172.16.10.11, [200/0], 00:21:47, bgp-65501, internal, tag 65501
172.16.10.4/31, ubest/mbest: 1/0, attached
    *via 172.16.10.4, Eth1/6, [0/0], 10:11:55, direct
172.16.10.4/32, ubest/mbest: 1/0, attached
    *via 172.16.10.4, Eth1/6, [0/0], 10:11:55, local
172.16.10.10/31, ubest/mbest: 1/0, attached
    *via 172.16.10.10, Eth1/7, [0/0], 10:11:11, direct
172.16.10.10/32, ubest/mbest: 1/0, attached
    *via 172.16.10.10, Eth1/7, [0/0], 10:11:11, local

Leaf3# sh bgp ipv4 unicast summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 172.16.0.13, local AS number 65501
BGP table version is 43, IPv4 Unicast config peers 2, capable peers 2
5 network entries and 7 paths using 1460 bytes of memory
BGP attribute entries [6/1032], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [4/16]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.10.5     4 65501    4871    4846       43    0    0 01:10:15 3         
172.16.10.11    4 65501   12128   12082       43    0    0 07:22:03 3         
Leaf3# sh bgp ipv4 unicast 
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 43, Local Router ID is 172.16.0.13
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>i172.16.0.11/32     172.16.10.5              0        100          0 ?
*|i                   172.16.10.11             0        100          0 ?
*>i172.16.0.12/32     172.16.10.5              0        100          0 ?
*|i                   172.16.10.11             0        100          0 ?
*>r172.16.0.13/32     0.0.0.0                  0        100      32768 i
*>i172.16.0.111/32    172.16.10.5              0        100          0 ?
*>i172.16.0.112/32    172.16.10.11             0        100          0 ?

Leaf3# sh bgp ipv4 unicast detail 
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 172.16.0.11/32, version 39
Paths: (2 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.5 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.11 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, multipa
th, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.11 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.11 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.12/32, version 40
Paths: (2 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.5 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.12 Cluster list: 172.16.0.111 

  Path type: internal, path is valid, not best reason: Neighbor Address, multipa
th, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.11 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0
      Originator: 172.16.0.12 Cluster list: 172.16.0.112 

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.13/32, version 41
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
BGP routing table entry for 172.16.0.111/32, version 42
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.5 (metric 0) from 172.16.10.5 (172.16.0.111)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 not advertised to any peer
BGP routing table entry for 172.16.0.112/32, version 43
Paths: (1 available, best #1)
Flags: (0x8008001a) (high32 00000000) on xmit-list, is in urib, is best urib rou
te, is in HW
Multipath: eBGP iBGP

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: NONE, path sourced internal to AS
    172.16.10.11 (metric 0) from 172.16.10.11 (172.16.0.112)
      Origin incomplete, MED 0, localpref 100, weight 0

  Path-id 1 not advertised to any peer

Leaf3# ping 172.16.0.111
PING 172.16.0.111 (172.16.0.111): 56 data bytes
64 bytes from 172.16.0.111: icmp_seq=0 ttl=254 time=3.436 ms
64 bytes from 172.16.0.111: icmp_seq=1 ttl=254 time=2.847 ms
64 bytes from 172.16.0.111: icmp_seq=2 ttl=254 time=3.114 ms
64 bytes from 172.16.0.111: icmp_seq=3 ttl=254 time=4.3 ms
64 bytes from 172.16.0.111: icmp_seq=4 ttl=254 time=3.004 ms

--- 172.16.0.111 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 2.847/3.34/4.3 ms
Leaf3# ping 172.16.0.112
PING 172.16.0.112 (172.16.0.112): 56 data bytes
64 bytes from 172.16.0.112: icmp_seq=0 ttl=254 time=3.395 ms
64 bytes from 172.16.0.112: icmp_seq=1 ttl=254 time=2.52 ms
64 bytes from 172.16.0.112: icmp_seq=2 ttl=254 time=2.001 ms
64 bytes from 172.16.0.112: icmp_seq=3 ttl=254 time=2.17 ms
64 bytes from 172.16.0.112: icmp_seq=4 ttl=254 time=2.074 ms

--- 172.16.0.112 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 2.001/2.432/3.395 ms
Leaf3# ping 172.16.0.11 source-interface loopback 0
PING 172.16.0.11 (172.16.0.11): 56 data bytes
64 bytes from 172.16.0.11: icmp_seq=0 ttl=253 time=18.59 ms
64 bytes from 172.16.0.11: icmp_seq=1 ttl=253 time=4.541 ms
64 bytes from 172.16.0.11: icmp_seq=2 ttl=253 time=4.612 ms
64 bytes from 172.16.0.11: icmp_seq=3 ttl=253 time=5.032 ms
64 bytes from 172.16.0.11: icmp_seq=4 ttl=253 time=3.939 ms

--- 172.16.0.11 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 3.939/7.342/18.59 ms
Leaf3# ping 172.16.0.12 source-interface loopback 0
PING 172.16.0.12 (172.16.0.12): 56 data bytes
64 bytes from 172.16.0.12: icmp_seq=0 ttl=253 time=7.362 ms
64 bytes from 172.16.0.12: icmp_seq=1 ttl=253 time=6.18 ms
64 bytes from 172.16.0.12: icmp_seq=2 ttl=253 time=7.918 ms
64 bytes from 172.16.0.12: icmp_seq=3 ttl=253 time=7.696 ms
64 bytes from 172.16.0.12: icmp_seq=4 ttl=253 time=9.11 ms
