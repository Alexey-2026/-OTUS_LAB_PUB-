# OSPF UNDERLAY
Цель работы: Hастроить OSPF для Underlay сети.

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
|------------|-------------|
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

## Конфигурация оборудования

### Spine-1

```
hostname Spine-1
!
feature ospf
!
interface Loopback0
 ip address 172.16.0.111 255.255.255.255
 ip router ospf UNDERLAY area 0
!
router ospf UNDERLAY
 router-id 172.16.0.111
!
interface Ethernet1/1
 description Link_to_Leaf-1
 no switchport
 ip address 172.16.10.1 255.255.255.254
 ip router ospf UNDERLAY area 0
 no shutdown
!
interface Ethernet1/2
 description Link_to_Leaf-2
 no switchport
 ip address 172.16.10.3 255.255.255.254
 ip router ospf UNDERLAY area 0
 no shutdown
!
interface Ethernet1/3
 no switchport
 description Link_to_Leaf-3
 ip address 172.16.10.5 255.255.255.254
 ip router ospf UNDERLAY area 0
 no shutdown
!
```

### Spine-2

```
hostname Spine-2
!
feature ospf
!
interface Loopback0
 ip address 172.16.0.112 255.255.255.255
 ip router ospf UNDERLAY area 0
!
router ospf UNDERLAY
 router-id 172.16.0.112
!
interface Ethernet1/1
 description Link_to_Leaf-1
 no switchport
 ip address 172.16.10.7 255.255.255.254
 ip router ospf UNDERLAY area 0
 no shutdown
!
interface Ethernet1/2
 description Link_to_Leaf-2
 no switchport
 ip address 172.16.10.9 255.255.255.254
 ip router ospf UNDERLAY area 0
 no shutdown
!
interface Ethernet1/3
 description Link_to_Leaf-3
 no switchport
 ip address 172.16.10.11 255.255.255.254
 ip router ospf UNDERLAY area 0
 no shutdown
!
```

### Leaf-1

```
hostname Leaf-1
!
feature ospf
feature interface-vlan 
!
interface Loopback0
 ip address 172.16.0.11 255.255.255.255
 ip router ospf UNDERLAY area 0
!
router ospf UNDERLAY
 router-id 172.16.0.11
!
interface Ethernet1/6
 description Link_to_Spine-1
 no switchport
 ip address 172.16.10.0 255.255.255.254
 ip router ospf UNDERLAY area 0
 no shutdown
!
interface Ethernet1/7
 description Link_to_Spine-2
 no switchport
 ip address 172.16.10.6 255.255.255.254
 ip router ospf UNDERLAY area 0
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

```

### Leaf-2

```
hostname Leaf-2
!
feature ospf
feature interface-vlan 
!
interface Loopback0
 ip address 172.16.0.12 255.255.255.255
 ip router ospf UNDERLAY area 0
!
router ospf UNDERLAY
 router-id 172.16.0.12
!
interface Ethernet1/6
 description Link_to_Spine-1
 no switchport
 ip address 172.16.10.2 255.255.255.254
 ip router ospf UNDERLAY area 0
 no shutdown
!
interface Ethernet1/7
 description Link_to_Spine-2
 no switchport
 ip address 172.16.10.8 255.255.255.254
 ip router ospf UNDERLAY area 0
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

```

### Leaf-3

```
hostname Leaf-3
!
feature ospf
feature interface-vlan 
!
interface Loopback0
 ip address 172.16.0.13 255.255.255.255
 ip router ospf UNDERLAY area 0
!
router ospf UNDERLAY
 router-id 172.16.0.13
!
interface Ethernet1/6
 description Link_to_Spine-1
 no switchport
 ip address 172.16.10.4 255.255.255.254
 ip router ospf UNDERLAY area 0
 no shutdown
!
interface Ethernet1/7
 description Link_to_Spine-2
 no switchport
 ip address 172.16.10.10 255.255.255.254
 ip router ospf UNDERLAY area 0
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
```

### **Проверка**
```
Spine1#sh ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.0.13       1   FULL/BDR        00:00:36    172.16.10.4     Ethernet1/3
172.16.0.12       1   FULL/BDR        00:00:32    172.16.10.2     Ethernet1/2
172.16.0.11       1   FULL/BDR        00:00:38    172.16.10.0     Ethernet1/1
Spine1#ping 172.16.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.11, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/5 ms
Spine1#ping 172.16.0.12
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/6 ms
Spine1#ping 172.16.0.13
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.13, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/2/4 ms

=======================================================================

Spine2#sh ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.0.13       1   FULL/BDR        00:00:31    172.16.10.10    Ethernet1/3
172.16.0.12       1   FULL/BDR        00:00:34    172.16.10.8     Ethernet1/2
172.16.0.11       1   FULL/BDR        00:00:36    172.16.10.6     Ethernet1/1
Spine2#ping 172.16.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.11, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/4 ms
Spine2#ping 172.16.0.12
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/5 ms
Spine2#ping 172.16.0.13
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.13, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/3/9 ms

=======================================================================

Leaf1#sh ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.0.112      1   FULL/DR         00:00:37    172.16.10.7     Ethernet1/7
172.16.0.111      1   FULL/DR         00:00:38    172.16.10.1     Ethernet1/6
Leaf1#       
Leaf1#ping 172.16.0.111
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.111, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/3/3 ms
Leaf1#ping 172.16.0.112
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.112, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/4/7 ms

===========================================================================

Leaf2#sh ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.0.112      1   FULL/DR         00:00:39    172.16.10.9     Ethernet1/7
172.16.0.111      1   FULL/DR         00:00:34    172.16.10.3     Ethernet1/6
Leaf2#
Leaf2#ping 172.16.0.111
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.111, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/4 ms
Leaf2#ping 172.16.0.112
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.112, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/2/3 ms

===============================================================================

Leaf3#sh ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.16.0.112      1   FULL/DR         00:00:30    172.16.10.11    Ethernet1/7
172.16.0.111      1   FULL/DR         00:00:33    172.16.10.5     Ethernet1/6
Leaf3#
Leaf3#ping 172.16.0.111
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.111, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/2/4 ms
Leaf3#ping 172.16.0.112
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.112, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/5 ms
```
