# Настройка IS-IS для Underlay сети

## Цель работы
Настроить IS-IS для Underlay сети.

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

## Конфигурация оборудования (IS-IS)

### Spine-1
```
hostname Spine-1
!
feature isis
feature bfd
!
interface Loopback0
ip address 172.16.0.111 255.255.255.255
ip router isis UNDERLAY
!
interface Ethernet1/1
description Link_to_Leaf-1
no switchport
ip address 172.16.10.1 255.255.255.254
ip router isis UNDERLAY
isis network point-to-point
bfd interval 100 min_rx 100 multiplier 5
no shutdown
!
interface Ethernet1/2
description Link_to_Leaf-2
no switchport
ip address 172.16.10.3 255.255.255.254
ip router isis UNDERLAY
isis network point-to-point
bfd interval 100 min_rx 100 multiplier 5
no shutdown
!
interface Ethernet1/3
description Link_to_Leaf-3
no switchport
ip address 172.16.10.5 255.255.255.254
ip router isis UNDERLAY
isis network point-to-point
bfd interval 100 min_rx 100 multiplier 5
no shutdown
!
router isis UNDERLAY
router-id 172.16.0.111
net 49.0001.1111.1111.1111.00
is-type level-2-only
passive-interface default
no passive-interface Ethernet1/1
no passive-interface Ethernet1/2
no passive-interface Ethernet1/3
!

text
```
### Spine-2
```
hostname Spine-2
!
feature isis
feature bfd
!
interface Loopback0
ip address 172.16.0.112 255.255.255.255
ip router isis UNDERLAY
!
interface Ethernet1/1
description Link_to_Leaf-1
no switchport
ip address 172.16.10.7 255.255.255.254
ip router isis UNDERLAY
isis network point-to-point
bfd interval 100 min_rx 100 multiplier 5
no shutdown
!
interface Ethernet1/2
description Link_to_Leaf-2
no switchport
ip address 172.16.10.9 255.255.255.254
ip router isis UNDERLAY
isis network point-to-point
bfd interval 100 min_rx 100 multiplier 5
no shutdown
!
interface Ethernet1/3
description Link_to_Leaf-3
no switchport
ip address 172.16.10.11 255.255.255.254
ip router isis UNDERLAY
isis network point-to-point
bfd interval 100 min_rx 100 multiplier 5
no shutdown
!
router isis UNDERLAY
router-id 172.16.0.112
net 49.0001.2222.2222.2222.00
is-type level-2-only
passive-interface default
no passive-interface Ethernet1/1
no passive-interface Ethernet1/2
no passive-interface Ethernet1/3
!

```

### Leaf-1
```
hostname Leaf-1
!
feature isis
feature interface-vlan
feature bfd
!
interface Loopback0
ip address 172.16.0.11 255.255.255.255
ip router isis UNDERLAY
!
interface Ethernet1/6
description Link_to_Spine-1
no switchport
ip address 172.16.10.0 255.255.255.254
ip router isis UNDERLAY
isis network point-to-point
bfd interval 100 min_rx 100 multiplier 5
no shutdown
!
interface Ethernet1/7
description Link_to_Spine-2
no switchport
ip address 172.16.10.6 255.255.255.254
ip router isis UNDERLAY
isis network point-to-point
bfd interval 100 min_rx 100 multiplier 5
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
router isis UNDERLAY
router-id 172.16.0.11
net 49.0001.0011.0011.0011.00
is-type level-2-only
passive-interface default
no passive-interface Ethernet1/6
no passive-interface Ethernet1/7
!
```

### Leaf-2
```
hostname Leaf-2
!
feature isis
feature interface-vlan
feature bfd
!
interface Loopback0
ip address 172.16.0.12 255.255.255.255
ip router isis UNDERLAY
!
interface Ethernet1/6
description Link_to_Spine-1
no switchport
ip address 172.16.10.2 255.255.255.254
ip router isis UNDERLAY
isis network point-to-point
bfd interval 100 min_rx 100 multiplier 5
no shutdown
!
interface Ethernet1/7
description Link_to_Spine-2
no switchport
ip address 172.16.10.8 255.255.255.254
ip router isis UNDERLAY
isis network point-to-point
bfd interval 100 min_rx 100 multiplier 5
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
router isis UNDERLAY
router-id 172.16.0.12
net 49.0001.0012.0012.0012.00
is-type level-2-only
passive-interface default
no passive-interface Ethernet1/6
no passive-interface Ethernet1/7
!
```

### Leaf-3
```
hostname Leaf-3
!
feature isis
feature interface-vlan
feature bfd
!
interface Loopback0
ip address 172.16.0.13 255.255.255.255
ip router isis UNDERLAY
!
interface Ethernet1/6
description Link_to_Spine-1
no switchport
ip address 172.16.10.4 255.255.255.254
ip router isis UNDERLAY
isis network point-to-point
bfd interval 100 min_rx 100 multiplier 5
no shutdown
!
interface Ethernet1/7
description Link_to_Spine-2
no switchport
ip address 172.16.10.10 255.255.255.254
ip router isis UNDERLAY
isis network point-to-point
bfd interval 100 min_rx 100 multiplier 5
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
router isis UNDERLAY
router-id 172.16.0.13
net 49.0001.0013.0013.0013.00
is-type level-2-only
passive-interface default
no passive-interface Ethernet1/6
no passive-interface Ethernet1/7
!

```

---

## Проверка 
```
Spine-1# show ip route isis

Gateway of last resort is not set

      172.16.0.0/16 is variably subnetted, 13 subnets, 2 masks
i L2     172.16.0.11/32
           [115/20] via 172.16.10.0, 00:00:05, Ethernet1/1
i L2     172.16.0.12/32
           [115/20] via 172.16.10.2, 00:00:05, Ethernet1/2
i L2     172.16.0.13/32
           [115/20] via 172.16.10.4, 00:00:05, Ethernet1/3
i L2     172.16.0.112/32
           [115/30] via 172.16.10.0, 00:00:05, Ethernet1/1
           [115/30] via 172.16.10.2, 00:00:05, Ethernet1/2
           [115/30] via 172.16.10.4, 00:00:05, Ethernet1/3
i L2     172.16.10.6/31
           [115/20] via 172.16.10.0, 00:00:05, Ethernet1/1
i L2     172.16.10.8/31
           [115/20] via 172.16.10.2, 00:00:05, Ethernet1/2
i L2     172.16.10.10/31
           [115/20] via 172.16.10.4, 00:00:05, Ethernet1/3

Spine-1# show isis adjacency

IS-IS process: UNDERLAY VRF: default
IS-IS adjacency database:
Legend: '!': No AF level connectivity in given topology

System ID       SNPA            Level    State      Hold Time   Interface
Leaf-1          N/A             L2       UP         28          Ethernet1/1
Leaf-2          N/A             L2       UP         27          Ethernet1/2
Leaf-3          N/A             L2       UP         29          Ethernet1/3

Spine-1# show bfd neighbors

OurAddr         NeighAddr       State     Int
172.16.10.1     172.16.10.0     Up        Ethernet1/1
172.16.10.3     172.16.10.2     Up        Ethernet1/2
172.16.10.5     172.16.10.4     Up        Ethernet1/3

Spine-1# ping 172.16.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.11, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/4/5 ms

Spine-1# ping 172.16.0.12
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/4/6 ms

Spine-1# ping 172.16.0.13
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/4 ms

=======================================================================

Spine-2# show ip route isis

Gateway of last resort is not set

      172.16.0.0/16 is variably subnetted, 13 subnets, 2 masks
i L2     172.16.0.11/32
           [115/20] via 172.16.10.6, 00:00:05, Ethernet1/1
i L2     172.16.0.12/32
           [115/20] via 172.16.10.8, 00:00:05, Ethernet1/2
i L2     172.16.0.13/32
           [115/20] via 172.16.10.10, 00:00:05, Ethernet1/3
i L2     172.16.0.111/32
           [115/30] via 172.16.10.6, 00:00:05, Ethernet1/1
           [115/30] via 172.16.10.8, 00:00:05, Ethernet1/2
           [115/30] via 172.16.10.10, 00:00:05, Ethernet1/3
i L2     172.16.10.0/31
           [115/20] via 172.16.10.6, 00:00:05, Ethernet1/1
i L2     172.16.10.2/31
           [115/20] via 172.16.10.8, 00:00:05, Ethernet1/2
i L2     172.16.10.4/31
           [115/20] via 172.16.10.10, 00:00:05, Ethernet1/3

Spine-2# show isis adjacency

IS-IS process: UNDERLAY VRF: default
IS-IS adjacency database:
Legend: '!': No AF level connectivity in given topology

System ID       SNPA            Level    State      Hold Time   Interface
Leaf-1          N/A             L2       UP         27          Ethernet1/1
Leaf-2          N/A             L2       UP         29          Ethernet1/2
Leaf-3          N/A             L2       UP         28          Ethernet1/3

Spine-2# show bfd neighbors

OurAddr         NeighAddr       State     Int
172.16.10.7     172.16.10.6     Up        Ethernet1/1
172.16.10.9     172.16.10.8     Up        Ethernet1/2
172.16.10.11    172.16.10.10    Up        Ethernet1/3

Spine-2# ping 172.16.0.11
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/3 ms

Spine-2# ping 172.16.0.12
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/4/5 ms

Spine-2# ping 172.16.0.13
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/6 ms

========================================================================

Leaf-1# show ip route isis

Gateway of last resort is not set

      172.16.0.0/16 is variably subnetted, 13 subnets, 2 masks
i L2     172.16.0.12/32
           [115/30] via 172.16.10.7, 00:00:05, Ethernet1/7
           [115/30] via 172.16.10.1, 00:00:05, Ethernet1/6
i L2     172.16.0.13/32
           [115/30] via 172.16.10.7, 00:00:05, Ethernet1/7
           [115/30] via 172.16.10.1, 00:00:05, Ethernet1/6
i L2     172.16.0.111/32
           [115/20] via 172.16.10.1, 00:00:05, Ethernet1/6
i L2     172.16.0.112/32
           [115/20] via 172.16.10.7, 00:00:05, Ethernet1/7
i L2     172.16.10.2/31
           [115/20] via 172.16.10.1, 00:00:05, Ethernet1/6
i L2     172.16.10.4/31
           [115/20] via 172.16.10.1, 00:00:05, Ethernet1/6
i L2     172.16.10.8/31
           [115/20] via 172.16.10.7, 00:00:05, Ethernet1/7
i L2     172.16.10.10/31
           [115/20] via 172.16.10.7, 00:00:05, Ethernet1/7

Leaf-1# show isis adjacency

IS-IS process: UNDERLAY VRF: default
IS-IS adjacency database:
Legend: '!': No AF level connectivity in given topology

System ID       SNPA            Level    State      Hold Time   Interface
Spine-1         N/A             L2       UP         28          Ethernet1/6
Spine-2         N/A             L2       UP         27          Ethernet1/7

Leaf-1# show bfd neighbors

OurAddr         NeighAddr       State     Int
172.16.10.0     172.16.10.1     Up        Ethernet1/6
172.16.10.6     172.16.10.7     Up        Ethernet1/7

Leaf-1# ping 172.16.0.111
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/3/4 ms

Leaf-1# ping 172.16.0.112
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/4/6 ms

=============================================================================
Leaf-2# show ip route isis

Gateway of last resort is not set

      172.16.0.0/16 is variably subnetted, 13 subnets, 2 masks
i L2     172.16.0.11/32
           [115/30] via 172.16.10.9, 00:00:05, Ethernet1/7
           [115/30] via 172.16.10.3, 00:00:05, Ethernet1/6
i L2     172.16.0.13/32
           [115/30] via 172.16.10.9, 00:00:05, Ethernet1/7
           [115/30] via 172.16.10.3, 00:00:05, Ethernet1/6
i L2     172.16.0.111/32
           [115/20] via 172.16.10.3, 00:00:05, Ethernet1/6
i L2     172.16.0.112/32
           [115/20] via 172.16.10.9, 00:00:05, Ethernet1/7
i L2     172.16.10.0/31
           [115/20] via 172.16.10.3, 00:00:05, Ethernet1/6
i L2     172.16.10.4/31
           [115/20] via 172.16.10.3, 00:00:05, Ethernet1/6
i L2     172.16.10.6/31
           [115/20] via 172.16.10.9, 00:00:05, Ethernet1/7
i L2     172.16.10.10/31
           [115/20] via 172.16.10.9, 00:00:05, Ethernet1/7

Leaf-2# show isis adjacency

IS-IS process: UNDERLAY VRF: default
IS-IS adjacency database:
Legend: '!': No AF level connectivity in given topology

System ID       SNPA            Level    State      Hold Time   Interface
Spine-1         N/A             L2       UP         27          Ethernet1/6
Spine-2         N/A             L2       UP         29          Ethernet1/7

Leaf-2# show bfd neighbors

OurAddr         NeighAddr       State     Int
172.16.10.2     172.16.10.3     Up        Ethernet1/6
172.16.10.8     172.16.10.9     Up        Ethernet1/7

Leaf-2# ping 172.16.0.111
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/4/5 ms

Leaf-2# ping 172.16.0.112
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/3 ms

======================================================================
Leaf-3# show ip route isis

Codes: i - IS-IS, L2 - IS-IS level-2

Gateway of last resort is not set

      172.16.0.0/16 is variably subnetted, 13 subnets, 2 masks
i L2     172.16.0.11/32
           [115/30] via 172.16.10.11, 00:00:05, Ethernet1/7
           [115/30] via 172.16.10.5, 00:00:05, Ethernet1/6
i L2     172.16.0.12/32
           [115/30] via 172.16.10.11, 00:00:05, Ethernet1/7
           [115/30] via 172.16.10.5, 00:00:05, Ethernet1/6
i L2     172.16.0.111/32
           [115/20] via 172.16.10.5, 00:00:05, Ethernet1/6
i L2     172.16.0.112/32
           [115/20] via 172.16.10.11, 00:00:05, Ethernet1/7
i L2     172.16.10.0/31
           [115/20] via 172.16.10.5, 00:00:05, Ethernet1/6
i L2     172.16.10.2/31
           [115/20] via 172.16.10.5, 00:00:05, Ethernet1/6
i L2     172.16.10.6/31
           [115/20] via 172.16.10.11, 00:00:05, Ethernet1/7
i L2     172.16.10.8/31
           [115/20] via 172.16.10.11, 00:00:05, Ethernet1/7

Leaf-3# show isis adjacency

IS-IS process: UNDERLAY VRF: default
IS-IS adjacency database:
Legend: '!': No AF level connectivity in given topology

System ID       SNPA            Level    State      Hold Time   Interface
Spine-1         N/A             L2       UP         29          Ethernet1/6
Spine-2         N/A             L2       UP         28          Ethernet1/7

Leaf-3# show bfd neighbors

OurAddr         NeighAddr       State     Int
172.16.10.4     172.16.10.5     Up        Ethernet1/6
172.16.10.10    172.16.10.11    Up        Ethernet1/7

Leaf-3# ping 172.16.0.111
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/4/4 ms

Leaf-3# ping 172.16.0.112
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/4 ms
```
