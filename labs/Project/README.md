# Комбинированная модель пересылки BUM-трафика (PIM + Ingress Replication) в архитектуре VXLAN EVPN Multi-Site

## Описание проекта

В рамках данного проекта реализована **многосайтовая архитектура EVPN VXLAN** с использованием комбинированной модели пересылки BUM-трафика. Решение сочетает два подхода:

* **PIM (Protocol Independent Multicast)** — для рассылки BUM-трафика внутри сайтов в сегменте VLAN 10.
* **Ingress Replication** — для BUM-трафика VLAN 20 внутри сайтов, а также для всех межсайтовых взаимодействий между Site 1 и Site 2.

### Как работает пересылка BUM-трафика в EVPN Multi-Site?

Ключевая особенность технологии EVPN Multi-Site:

* **Внутри сайта (intra-site):** Метод репликации определяется настройками на Leaf. В данном проекте для VLAN 10 используется PIM (на обоих сайтах), для VLAN 20 — Ingress Replication (на обоих сайтах).
* **Между сайтами (inter-site):** BUM-трафик передаётся **только через Ingress Replication** — это обязательное требование технологии Multi-Site. Согласно официальной документации Cisco, **до версии NX-OS 10.2(2)F между DCI-пирами поддерживается только Ingress Replication**.

На пограничных шлюзах (BGW) команда `multisite ingress-replication` включает именно этот механизм межсайтовой репликации. Она определяет метод репликации BUM-трафика для расширения Layer 2 VNI между сайтами.

> **Источник:** Официальная документация Cisco NX-OS VXLAN Configuration Guide, раздел "Configure VNI dual mode":  
> *"Defines the Multi-Site BUM replication method for extending the Layer 2 VNI"*  
> `switch(config-if-nve-vni)# multisite ingress-replication`  
> — [Cisco NX-OS VXLAN Configuration Guide, Release 10.4(x), стр. 15-16](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/104x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-104x/m_configuring_multisite_93x.pdf#page=15)

### Зачем нужна такая комбинация?

Данная конфигурация показывает два сценария:

1. **Для VLAN 10** — демонстрация того, как PIM, используемый внутри каждого сайта, стыкуется с обязательным Ingress Replication на границе сайта. Это типичный сценарий миграции или сосуществования legacy-мультикаста с  EVPN Multi-Site архитектурой.

2. **Для VLAN 20** — сценарий, где Ingress Replication используется везде (и внутри сайтов, и между ними).

**Важное замечание про PIM:** PIM сложнее в траблшутинге (требует проверки RPF, mroute, состояния (*,G) и (S,G)). Ingress Replication проще: при проблемах достаточно проверить BGP EVPN (маршруты типа 3, route-target) и связность NVE-пиров. Ingress Replication проще в настройке и диагностике, поэтому он является наиболее часто используемым методом в современных ЦОД. 

## Схема сети

<img src="https://github.com/user-attachments/assets/c6936e33-662d-44dd-a4f1-0d13465b4bf5" width="700" style="max-width: 100%; height: auto;">

*Рисунок 1 — Топология VXLAN EVPN Multi-Site с двумя сайтами и SuperSpine*

### Архитектура и топология

* **SuperSpine** — единая точка подключения для всех сайтов, выполняет маршрутизацию между AS 65001 (Site 1) и AS 65002 (Site 2) через eBGP.
* **Сайт 1 (AS 65001)**: три Spine (Spine1/2/3) и три Leaf (Leaf11, Leaf12, Leaf13). Spine1 и Spine2 объединены в **Anycast-RP (172.0.0.1)** и являются пограничными шлюзами сайта (BGW) с VIP-адресом 100.0.0.1. Leaf11 и Leaf12 связаны в **vPC**.
* **Сайт 2 (AS 65002)**: один Spine3 (также BGW с VIP 100.0.0.2) и один leaf Leaf21.
* **Underlay**: OSPF (processes `underlay` для сайтов и `DCI` для SuperSpine) с BFD. Адресация P2P (/31), Loopback 0 — для идентификации устройств и BGP-пиринга.
* **Overlay**: VXLAN с EVPN-сигнализацией. Маршруты между сайтами распространяются через SuperSpine с сохранением следующего перехода (route-map `NH_UNCH`).

### Цели работы

1. Реализовать **мультисайтовую EVPN-фабрику** с двумя независимыми AS.
2. Настроить **Anycast-RP и PIM SM** для доставки BUM-трафика внутри VLAN 10 (мультикаст-группа 225.0.0.10).
3. Настроить **Ingress Replication** для VLAN 20 (на уровне BGP EVPN).
4. Обеспечить **взаимодействие между сайтами** через SuperSpine с транзитом как L2-, так и L3-маршрутов EVPN.
5. Обеспечить резервирование и отказоустойчивость за счет vPC и Anycast-шлюзов.

## Адресация

### Underlay (линки /31)

<details>
<summary><b>Таблица линков между устройствами</b></summary>

| Линк | Интерфейс Spine | IP Spine | Интерфейс Leaf | IP Leaf |
|------|----------------|-----------|----------------|---------|
| SuperSpine–Spine1 | Eth1/1 | 172.16.111.0 | Eth1/4 | 172.16.111.1 |
| SuperSpine–Spine2 | Eth1/2 | 172.16.112.0 | Eth1/4 | 172.16.112.1 |
| SuperSpine–Spine3 | Eth1/3 | 172.16.113.0 | Eth1/4 | 172.16.113.1 |
| Spine1–Leaf1 (Leaf11) | Eth1/1 | 172.16.11.1 | Eth1/1 | 172.16.11.0 |
| Spine1–Leaf2 (Leaf12) | Eth1/2 | 172.16.21.1 | Eth1/1 | 172.16.21.0 |
| Spine1–Leaf3 (Leaf13) | Eth1/3 | 172.16.31.1 | Eth1/1 | 172.16.31.0 |
| Spine2–Leaf1 (Leaf11) | Eth1/1 | 172.16.12.1 | Eth1/2 | 172.16.12.0 |
| Spine2–Leaf2 (Leaf12) | Eth1/2 | 172.16.22.1 | Eth1/2 | 172.16.22.0 |
| Spine2–Leaf3 (Leaf13) | Eth1/3 | 172.16.32.1 | Eth1/2 | 172.16.32.0 |
| Spine3–Leaf21 | Eth1/1 | 172.16.211.0 | Eth1/1 | 172.16.211.1 |

</details>

### Loopback-адреса

<details>
<summary><b>Таблица Loopback0</b></summary>

| Устройство | Loopback0 IP |
|------------|--------------|
| SuperSpine | 172.0.0.254/32 |
| Spine-1 | 172.0.0.111/32 |
| Spine-2 | 172.0.0.112/32 |
| Spine-3 | 172.0.0.113/32 |
| Leaf-11 | 172.0.0.11/32 |
| Leaf-12 | 172.0.0.12/32 |
| Leaf-13 | 172.0.0.13/32 |
| Leaf-21 | 172.0.0.21/32 |

</details>

### Серверные VLAN и шлюзы

<details>
<summary><b>Подсети и Anycast-шлюзы</b></summary>

| Leaf | VLAN | Подсеть | Шлюз (Anycast) |
|------|------|---------|----------------|
| Leaf-11, Leaf-12, Leaf-13, Leaf-21 | 10 | 10.0.0.0/24 | 10.0.0.254 |
| Leaf-11, Leaf-12, Leaf-21 | 20 | 20.0.0.0/24 | 20.0.0.254 |

> **Примечание:** Leaf13 подключен только к VLAN 10. Для VLAN 20 на Leaf13 используется Ingress Replication без мультикаста.

</details>

## Конфигурация оборудования

### SuperSpine (маршрутизатор между сайтами)

<details>
<summary><b>Конфигурация SuperSpine</b></summary>

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
</details>

### Spine1 (BGW для Site1, Anycast-RP 172.0.0.1)

<details> 
<summary><b>Конфигурация Spine1</b></summary>
  
```
  hostname Spine1

nv overlay evpn
feature ospf
feature bgp
feature pim
feature msdp
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
bfd interval 100 min_rx 100 multiplier 5
advertise evpn multicast
configure maintenance profile normal-mode
  interface Ethernet1/1-4
    no shutdown
  sleep instance 1 1000
  router ospf underlay
    no isolate
  router ospf DCI
    no isolate
  sleep instance 2 1200
  router bgp 65001
    address-family l2vpn evpn
      nexthop trigger-delay critical 1 non-critical 1
configure maintenance profile maintenance-mode
  interface Ethernet1/1-4
    shutdown
  router ospf DCI
    no isolate
  router ospf underlay
    isolate
  sleep instance 1 100
  router bgp 65001
    address-family l2vpn evpn
      no nexthop trigger-delay

ip pim rp-address 172.0.0.1 group-list 225.0.0.0/24
ip pim ssm range 232.0.0.0/8
ip pim anycast-rp 172.0.0.1 172.0.0.111
ip pim anycast-rp 172.0.0.1 172.0.0.112
ip msdp originator-id loopback99
ip msdp peer 99.99.99.112 connect-source loopback99
ip msdp peer 99.99.99.113 connect-source loopback99

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
</details>

### Spine2 (второй BGW для Site1 и Anycast-RP)
<details>
<summary><b>Конфигурация Spine2</b></summary>
  
```
 hostname Spine2

nv overlay evpn
feature ospf
feature bgp
feature pim
feature msdp
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
bfd interval 100 min_rx 100 multiplier 5
advertise evpn multicast
configure maintenance profile normal-mode
  interface Ethernet1/1-4
    no shutdown
  sleep instance 1 1000
  router ospf underlay
    no isolate
  router ospf DCI
    no isolate
  sleep instance 2 1200
  router bgp 65001
    address-family l2vpn evpn
      nexthop trigger-delay critical 1 non-critical 1
configure maintenance profile maintenance-mode
  interface Ethernet1/1-4
    shutdown
  router ospf DCI
    no isolate
  router ospf underlay
    isolate
  sleep instance 1 100
  router bgp 65001
    address-family l2vpn evpn
      no nexthop trigger-delay

ip pim rp-address 172.0.0.1 group-list 225.0.0.0/24
ip pim ssm range 232.0.0.0/8
ip pim anycast-rp 172.0.0.1 172.0.0.111
ip pim anycast-rp 172.0.0.1 172.0.0.112
ip msdp originator-id loopback99
ip msdp peer 99.99.99.111 connect-source loopback99
ip msdp peer 99.99.99.113 connect-source loopback99

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
</details>

### Spine3 (BGW для Site2, Anycast-RP 172.0.0.1)
<details> 
<summary><b>Конфигурация Spine3</b></summary>
  
```
 hostname Spine3

nv overlay evpn
feature ospf
feature bgp
feature pim
feature msdp
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
bfd interval 100 min_rx 100 multiplier 5
advertise evpn multicast
configure maintenance profile normal-mode
  interface Ethernet1/1-4
    no shutdown
  sleep instance 1 1000
  router ospf UNDERLAY
    no isolate
  router ospf DCI
    no isolate
  sleep instance 2 1200
  router bgp 65001
    address-family l2vpn evpn
      nexthop trigger-delay critical 1 non-critical 1
configure maintenance profile maintenance-mode
  interface Ethernet1/1-4
    shutdown
  router ospf DCI
    no isolate
  router ospf UNDERLAY
    isolate
  sleep instance 1 100
  router bgp 65001
    address-family l2vpn evpn
      no nexthop trigger-delay

ip pim rp-address 172.0.0.1 group-list 225.0.0.0/24
ip pim ssm range 232.0.0.0/8
ip pim anycast-rp 172.0.0.1 172.0.0.113
ip msdp originator-id loopback99
ip msdp peer 99.99.99.111 connect-source loopback99
ip msdp peer 99.99.99.112 connect-source loopback99

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
  ip address 172.0.0.1/32
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
</details>

### Leaf11 (vPC-пара с Anycast-шлюзами)
<details> 
<summary><b>Конфигурация Leaf11 </b></summary>
  
```
hostname Leaf11

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
bfd interval 100 min_rx 100 multiplier 5
configure maintenance profile normal-mode
  interface Ethernet1/1-2
    no shutdown
  sleep instance 1 1000
  router ospf underlay
    no isolate
  router ospf DCI
    no isolate
  sleep instance 2 1200
  router bgp 65001
    address-family l2vpn evpn
      nexthop trigger-delay critical 1 non-critical 1
configure maintenance profile maintenance-mode
  interface Ethernet1/1-2
    shutdown
  router ospf underlay
    isolate
  sleep instance 1 100
  router bgp 65001
    address-family l2vpn evpn
      no nexthop trigger-delay

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
</details>

### Leaf12 (vPC-пара с Anycast-шлюзами)
<details> 
<summary><b>Конфигурация Leaf12</b></summary>
  
```
hostname Leaf12

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
bfd interval 100 min_rx 100 multiplier 5
configure maintenance profile normal-mode
  interface Ethernet1/1-2
    no shutdown
  sleep instance 1 1000
  router ospf underlay
    no isolate
  router ospf DCI
    no isolate
  sleep instance 2 1200
  router bgp 65001
    address-family l2vpn evpn
      nexthop trigger-delay critical 1 non-critical 1
configure maintenance profile maintenance-mode
  interface Ethernet1/1-2
    shutdown
  router ospf underlay
    isolate
  sleep instance 1 100
  router bgp 65001
    address-family l2vpn evpn
      no nexthop trigger-delay

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
</details>

### Leaf13 (одиночный Leaf, только VLAN 10)
<details> 
<summary><b>Конфигурация Leaf13</b></summary>
  
```
hostname Leaf13

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
bfd interval 100 min_rx 100 multiplier 5
configure maintenance profile normal-mode
  interface Ethernet1/1-2
    no shutdown
  sleep instance 1 1000
  router ospf underlay
    no isolate
  router ospf DCI
    no isolate
  sleep instance 2 1200
  router bgp 65001
    address-family l2vpn evpn
      nexthop trigger-delay critical 1 non-critical 1
configure maintenance profile maintenance-mode
  interface Ethernet1/1-2
    shutdown
  router ospf underlay
    isolate
  sleep instance 1 100
  router bgp 65001
    address-family l2vpn evpn
      no nexthop trigger-delay


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
</details>

### Leaf21 (Leaf в Site2, обе VLAN)
<details> 
<summary><b>Конфигурация Leaf21</b></summary>
  
```
hostname Leaf21

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
bfd interval 100 min_rx 100 multiplier 5
configure maintenance profile normal-mode
  interface Ethernet1/1
    no shutdown
  sleep instance 1 1000
  router ospf UNDERLAY
    no isolate
  router ospf DCI
    no isolate
  sleep instance 2 1200
  router bgp 65001
    address-family l2vpn evpn
      nexthop trigger-delay critical 1 non-critical 1
configure maintenance profile maintenance-mode
  interface Ethernet1/1
    shutdown
  router ospf UNDERLAY
    isolate
  sleep instance 1 100
  router bgp 65001
    address-family l2vpn evpn
      no nexthop trigger-delay

fabric forwarding anycast-gateway-mac 0001.0001.0001
ip pim rp-address 172.0.0.1 group-list 225.0.0.0/24
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
</details>

## Диагностические команды

Ниже приведены основные `show`-команды для проверки работоспособности каждой из используемых технологий. 

### VPC (Leaf11 и Leaf12)
Для связки Leaf11-Leaf12 проверяем состояние vPC.

```
Leaf11# show vpc brief
Legend:
(*) - local vPC is down, forwarding via vPC peer-link

vPC domain id                     : 10  
Peer status                       : peer adjacency formed ok      
vPC keep-alive status             : peer is alive                 
Configuration consistency status  : success 
Per-vlan consistency status       : success                       
Type-2 consistency status         : success 
vPC role                          : secondary, operational primary
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
```

```
Leaf12# show vpc brief 
Legend:
                (*) - local vPC is down, forwarding via vPC peer-link

vPC domain id                     : 10  
Peer status                       : peer adjacency formed ok      
vPC keep-alive status             : peer is alive                 
Configuration consistency status  : success 
Per-vlan consistency status       : success                       
Type-2 consistency status         : success 
vPC role                          : primary, operational secondary
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
```

### Underlay (OSPF + BFD)

Проверка соседства OSPF на всех интерфейсах — первый шаг. Если Underlay не работает, EVPN и VXLAN работать не смогут.

### OSPF соседи (должны быть в состоянии FULL)
```
Spine1# show ip ospf neighbor
 OSPF Process ID underlay VRF default
 Total number of neighbors: 3
 Neighbor ID     Pri State            Up Time  Address         Interface
 172.0.0.11        1 FULL/ -          1w2d     172.16.11.0     Eth1/1 
 172.0.0.12        1 FULL/ -          1w2d     172.16.21.0     Eth1/2 
 172.0.0.13        1 FULL/ -          1w2d     172.16.31.0     Eth1/3 
 OSPF Process ID DCI VRF default
 Total number of neighbors: 1
 Neighbor ID     Pri State            Up Time  Address         Interface
 172.0.0.254       1 FULL/ -          4d20h    172.16.111.0    Eth1/4

Spine2# show ip ospf neighbor
 OSPF Process ID underlay VRF default
 Total number of neighbors: 3
 Neighbor ID     Pri State            Up Time  Address         Interface
 172.0.0.11        1 FULL/ -          00:01:34 172.16.12.0     Eth1/1 
 172.0.0.12        1 FULL/ -          00:01:26 172.16.22.0     Eth1/2 
 172.0.0.13        1 FULL/ -          00:01:33 172.16.32.0     Eth1/3 
 OSPF Process ID DCI VRF default
 Total number of neighbors: 1
 Neighbor ID     Pri State            Up Time  Address         Interface
 172.0.0.254       1 FULL/ -          3d09h    172.16.112.0    Eth1/4

Spine3# show ip ospf neighbor
 OSPF Process ID UNDERLAY VRF default
 Total number of neighbors: 1
 Neighbor ID     Pri State            Up Time  Address         Interface
 172.0.0.21        1 FULL/ -          4d21h    172.16.211.1    Eth1/1 
 OSPF Process ID DCI VRF default
 Total number of neighbors: 1
 Neighbor ID     Pri State            Up Time  Address         Interface
 172.0.0.254       1 FULL/ -          4d21h    172.16.113.0    Eth1/4 
```

### Маршруты OSPF (Loopback'и всех устройств должны быть в таблице)
```
Leaf11# show ip route ospf
99.99.99.111/32, ubest/mbest: 1/0
    *via 172.16.11.1, Eth1/1, [110/41], 1w2d, ospf-underlay, intra
99.99.99.112/32, ubest/mbest: 1/0
    *via 172.16.12.1, Eth1/2, [110/41], 00:04:39, ospf-underlay, intra
172.0.0.1/32, ubest/mbest: 2/0
    *via 172.16.11.1, Eth1/1, [110/41], 1w2d, ospf-underlay, intra
    *via 172.16.12.1, Eth1/2, [110/41], 00:04:39, ospf-underlay, intra
172.0.0.12/32, ubest/mbest: 1/0
    *via 172.16.39.1, Vlan3900, [110/41], 1w2d, ospf-underlay, intra
172.0.0.13/32, ubest/mbest: 2/0
    *via 172.16.11.1, Eth1/1, [110/81], 1w2d, ospf-underlay, intra
    *via 172.16.12.1, Eth1/2, [110/81], 00:04:39, ospf-underlay, intra
172.0.0.111/32, ubest/mbest: 1/0
    *via 172.16.11.1, Eth1/1, [110/41], 1w2d, ospf-underlay, intra
172.0.0.112/32, ubest/mbest: 1/0
    *via 172.16.12.1, Eth1/2, [110/41], 00:04:39, ospf-underlay, intra
172.16.21.0/31, ubest/mbest: 2/0
    *via 172.16.11.1, Eth1/1, [110/80], 1w2d, ospf-underlay, intra
    *via 172.16.39.1, Vlan3900, [110/80], 1w2d, ospf-underlay, intra
172.16.22.0/31, ubest/mbest: 2/0
    *via 172.16.12.1, Eth1/2, [110/80], 00:04:39, ospf-underlay, intra
    *via 172.16.39.1, Vlan3900, [110/80], 1w2d, ospf-underlay, intra
172.16.31.0/31, ubest/mbest: 1/0
    *via 172.16.11.1, Eth1/1, [110/80], 1w2d, ospf-underlay, intra
172.16.32.0/31, ubest/mbest: 1/0
    *via 172.16.12.1, Eth1/2, [110/80], 00:04:39, ospf-underlay, intra

Leaf12# show ip route ospf
99.99.99.111/32, ubest/mbest: 1/0
    *via 172.16.21.1, Eth1/1, [110/41], 1w2d, ospf-underlay, intra
99.99.99.112/32, ubest/mbest: 1/0
    *via 172.16.22.1, Eth1/2, [110/41], 00:05:21, ospf-underlay, intra
172.0.0.1/32, ubest/mbest: 2/0
    *via 172.16.21.1, Eth1/1, [110/41], 1w2d, ospf-underlay, intra
    *via 172.16.22.1, Eth1/2, [110/41], 00:05:21, ospf-underlay, intra
172.0.0.11/32, ubest/mbest: 1/0
    *via 172.16.39.0, Vlan3900, [110/41], 1w2d, ospf-underlay, intra
172.0.0.13/32, ubest/mbest: 2/0
    *via 172.16.21.1, Eth1/1, [110/81], 1w2d, ospf-underlay, intra
    *via 172.16.22.1, Eth1/2, [110/81], 00:05:21, ospf-underlay, intra
172.0.0.111/32, ubest/mbest: 1/0
    *via 172.16.21.1, Eth1/1, [110/41], 1w2d, ospf-underlay, intra
172.0.0.112/32, ubest/mbest: 1/0
    *via 172.16.22.1, Eth1/2, [110/41], 00:05:21, ospf-underlay, intra
172.16.11.0/31, ubest/mbest: 2/0
    *via 172.16.21.1, Eth1/1, [110/80], 1w2d, ospf-underlay, intra
    *via 172.16.39.0, Vlan3900, [110/80], 1w2d, ospf-underlay, intra
172.16.12.0/31, ubest/mbest: 2/0
    *via 172.16.22.1, Eth1/2, [110/80], 00:05:21, ospf-underlay, intra
    *via 172.16.39.0, Vlan3900, [110/80], 1w2d, ospf-underlay, intra
172.16.31.0/31, ubest/mbest: 1/0
    *via 172.16.21.1, Eth1/1, [110/80], 1w2d, ospf-underlay, intra
172.16.32.0/31, ubest/mbest: 1/0
    *via 172.16.22.1, Eth1/2, [110/80], 00:05:21, ospf-underlay, intra

Leaf13# show ip route ospf
99.99.99.111/32, ubest/mbest: 1/0
    *via 172.16.31.1, Eth1/1, [110/41], 1w2d, ospf-underlay, intra
99.99.99.112/32, ubest/mbest: 1/0
    *via 172.16.32.1, Eth1/2, [110/41], 00:07:49, ospf-underlay, intra
172.0.0.0/24, ubest/mbest: 2/0
    *via 172.16.31.1, Eth1/1, [110/81], 1w2d, ospf-underlay, intra
    *via 172.16.32.1, Eth1/2, [110/81], 00:07:49, ospf-underlay, intra
172.0.0.1/32, ubest/mbest: 2/0
    *via 172.16.31.1, Eth1/1, [110/41], 1w2d, ospf-underlay, intra
    *via 172.16.32.1, Eth1/2, [110/41], 00:07:49, ospf-underlay, intra
172.0.0.11/32, ubest/mbest: 2/0
    *via 172.16.31.1, Eth1/1, [110/81], 1w2d, ospf-underlay, intra
    *via 172.16.32.1, Eth1/2, [110/81], 00:07:49, ospf-underlay, intra
172.0.0.12/32, ubest/mbest: 2/0
    *via 172.16.31.1, Eth1/1, [110/81], 1w2d, ospf-underlay, intra
    *via 172.16.32.1, Eth1/2, [110/81], 00:07:42, ospf-underlay, intra
172.0.0.111/32, ubest/mbest: 1/0
    *via 172.16.31.1, Eth1/1, [110/41], 1w2d, ospf-underlay, intra
172.0.0.112/32, ubest/mbest: 1/0
    *via 172.16.32.1, Eth1/2, [110/41], 00:07:49, ospf-underlay, intra
172.16.11.0/31, ubest/mbest: 1/0
    *via 172.16.31.1, Eth1/1, [110/80], 1w2d, ospf-underlay, intra
172.16.12.0/31, ubest/mbest: 1/0
    *via 172.16.32.1, Eth1/2, [110/80], 00:07:49, ospf-underlay, intra
172.16.21.0/31, ubest/mbest: 1/0
    *via 172.16.31.1, Eth1/1, [110/80], 1w2d, ospf-underlay, intra
172.16.22.0/31, ubest/mbest: 1/0
    *via 172.16.32.1, Eth1/2, [110/80], 00:07:49, ospf-underlay, intra
172.16.39.0/31, ubest/mbest: 2/0
    *via 172.16.31.1, Eth1/1, [110/120], 1w2d, ospf-underlay, intra
    *via 172.16.32.1, Eth1/2, [110/120], 00:07:49, ospf-underlay, intra

Leaf21# show ip route ospf

99.99.99.113/32, ubest/mbest: 1/0
    *via 172.16.211.0, Eth1/1, [110/41], 4d21h, ospf-UNDERLAY, intra
172.0.0.2/32, ubest/mbest: 1/0
    *via 172.16.211.0, Eth1/1, [110/41], 4d21h, ospf-UNDERLAY, intra
172.0.0.113/32, ubest/mbest: 1/0
    *via 172.16.211.0, Eth1/1, [110/41], 4d21h, ospf-UNDERLAY, intra
```
### BGP
Проверяем, что пиры установлены, а нужные маршруты (тип 2, тип 3) приходят.

<details> 
<summary><b> Leaf11</b></summary>
 
 ```
Leaf11# show bgp l2vpn evpn summary 
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 172.0.0.100, local AS number 65001
BGP table version is 8725, L2VPN EVPN config peers 2, capable peers 2
41 network entries and 59 paths using 9764 bytes of memory
BGP attribute entries [46/7912], BGP AS path entries [1/10]
BGP community entries [0/0], BGP clusterlist entries [4/16]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.0.0.111     4 65001  644382  639046     8725    0    0    2d06h 16        
172.0.0.112     4 65001 1064085 1056425     8725    0    0 00:59:38 16

Leaf11# show bgp l2vpn evpn 
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 8725, Local Router ID is 172.0.0.100
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 1:10000
* i[2]:[0]:[0]:[48]:[0000.0000.0021]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>i                   100.0.0.1                         100          0 65254 65002 i
* i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>i                   100.0.0.1                         100          0 65254 65002 i
* i[2]:[0]:[0]:[48]:[0000.0000.0021]:[32]:[10.0.0.21]/272
                      100.0.0.1                         100          0 65254 65002 i
*>i                   100.0.0.1                         100          0 65254 65002 i

Route Distinguisher: 1:20000
* i[2]:[0]:[0]:[48]:[0000.0000.0022]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>i                   100.0.0.1                         100          0 65254 65002 i
* i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>i                   100.0.0.1                         100          0 65254 65002 i
* i[2]:[0]:[0]:[48]:[0000.0000.0022]:[32]:[20.0.0.22]/272
                      100.0.0.1                         100          0 65254 65002 i
*>i                   100.0.0.1                         100          0 65254 65002 i
*>i[3]:[0]:[32]:[99.99.99.111]/88
                      99.99.99.111                      100          0 i
*>i[3]:[0]:[32]:[99.99.99.112]/88
                      99.99.99.112                      100          0 i

Route Distinguisher: 172.0.0.11:32777    (L2VNI 10000)
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.0.0.100                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.0.0.13                        100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0021]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.111                      100          0 i
*>i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.112                      100          0 i
*>i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.0.0.100                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.0.0.13                        100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0021]:[32]:[10.0.0.21]/272
                      100.0.0.1                         100          0 65254 65002 i

Route Distinguisher: 172.0.0.11:32787    (L2VNI 20000)
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.0.0.100                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0022]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.111                      100          0 i
*>i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.112                      100          0 i
*>i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.0.0.100                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0022]:[32]:[20.0.0.22]/272
                      100.0.0.1                         100          0 65254 65002 i
* i[3]:[0]:[32]:[99.99.99.111]/88
                      99.99.99.111                      100          0 i
*>i                   99.99.99.111                      100          0 i
* i[3]:[0]:[32]:[99.99.99.112]/88
                      99.99.99.112                      100          0 i
*>i                   99.99.99.112                      100          0 i
*>l[3]:[0]:[32]:[172.0.0.100]/88
                      172.0.0.100                       100      32768 i

Route Distinguisher: 172.0.0.13:3
* i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.0.0.13               0        100          0 ?
*>i                   172.0.0.13               0        100          0 ?

Route Distinguisher: 172.0.0.13:32777
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.0.0.13                        100          0 i
*>i                   172.0.0.13                        100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.0.0.13                        100          0 i
*>i                   172.0.0.13                        100          0 i

Route Distinguisher: 172.0.0.111:32777
* i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.111                      100          0 i
*>i                   99.99.99.111                      100          0 i

Route Distinguisher: 172.0.0.111:32787
* i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.111                      100          0 i
*>i                   99.99.99.111                      100          0 i
* i[3]:[0]:[32]:[99.99.99.111]/88
                      99.99.99.111                      100          0 i
*>i                   99.99.99.111                      100          0 i

Route Distinguisher: 172.0.0.112:32777
*>i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.112                      100          0 i
* i                   99.99.99.112                      100          0 i

Route Distinguisher: 172.0.0.112:32787
*>i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.112                      100          0 i
* i                   99.99.99.112                      100          0 i
* i[3]:[0]:[32]:[99.99.99.112]/88
                      99.99.99.112                      100          0 i
*>i                   99.99.99.112                      100          0 i

Route Distinguisher: 172.0.0.100:3    (L3VNI 770000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.0.0.13                        100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0021]:[32]:[10.0.0.21]/272
                      100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0022]:[32]:[20.0.0.22]/272
                      100.0.0.1                         100          0 65254 65002 i
* i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.0.0.13               0        100          0 ?
*>l                   172.0.0.100              0        100      32768 ?
*>l[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.0.0.100              0        100      32768 ?
```
</details> 

<details> 
<summary><b> Leaf12</b></summary>

```
Leaf12# show bgp l2vpn evpn summary 
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 172.0.0.100, local AS number 65001
BGP table version is 3609, L2VPN EVPN config peers 2, capable peers 2
41 network entries and 59 paths using 9764 bytes of memory
BGP attribute entries [46/7912], BGP AS path entries [1/10]
BGP community entries [0/0], BGP clusterlist entries [4/16]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.0.0.111     4 65001  262867  261338     3609    0    0 00:28:20 16        
172.0.0.112     4 65001  166199  164632     3609    0    0 01:13:21 16

Leaf12# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 3609, Local Router ID is 172.0.0.100
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 1:10000
*>i[2]:[0]:[0]:[48]:[0000.0000.0021]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
* i                   100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
* i                   100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0021]:[32]:[10.0.0.21]/272
                      100.0.0.1                         100          0 65254 65002 i
* i                   100.0.0.1                         100          0 65254 65002 i

Route Distinguisher: 1:20000
*>i[2]:[0]:[0]:[48]:[0000.0000.0022]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
* i                   100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
* i                   100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0022]:[32]:[20.0.0.22]/272
                      100.0.0.1                         100          0 65254 65002 i
* i                   100.0.0.1                         100          0 65254 65002 i
*>i[3]:[0]:[32]:[99.99.99.111]/88
                      99.99.99.111                      100          0 i
*>i[3]:[0]:[32]:[99.99.99.112]/88
                      99.99.99.112                      100          0 i

Route Distinguisher: 172.0.0.13:3
*>i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.0.0.13               0        100          0 ?
* i                   172.0.0.13               0        100          0 ?

Route Distinguisher: 172.0.0.13:32777
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.0.0.13                        100          0 i
* i                   172.0.0.13                        100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.0.0.13                        100          0 i
* i                   172.0.0.13                        100          0 i

Route Distinguisher: 172.0.0.100:32777    (L2VNI 10000)
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.0.0.100                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.0.0.13                        100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0021]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.111                      100          0 i
*>i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.112                      100          0 i
*>i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.0.0.100                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.0.0.13                        100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0021]:[32]:[10.0.0.21]/272
                      100.0.0.1                         100          0 65254 65002 i

Route Distinguisher: 172.0.0.100:32787    (L2VNI 20000)
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      172.0.0.100                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0022]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.111                      100          0 i
*>i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.112                      100          0 i
*>i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.0.0.100                       100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0022]:[32]:[20.0.0.22]/272
                      100.0.0.1                         100          0 65254 65002 i
*>i[3]:[0]:[32]:[99.99.99.111]/88
                      99.99.99.111                      100          0 i
* i                   99.99.99.111                      100          0 i
* i[3]:[0]:[32]:[99.99.99.112]/88
                      99.99.99.112                      100          0 i
*>i                   99.99.99.112                      100          0 i
*>l[3]:[0]:[32]:[172.0.0.100]/88
                      172.0.0.100                       100      32768 i

Route Distinguisher: 172.0.0.111:32777
*>i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.111                      100          0 i
* i                   99.99.99.111                      100          0 i

Route Distinguisher: 172.0.0.111:32787
*>i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.111                      100          0 i
* i                   99.99.99.111                      100          0 i
*>i[3]:[0]:[32]:[99.99.99.111]/88
                      99.99.99.111                      100          0 i
* i                   99.99.99.111                      100          0 i

Route Distinguisher: 172.0.0.112:32777
* i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.112                      100          0 i
*>i                   99.99.99.112                      100          0 i

Route Distinguisher: 172.0.0.112:32787
* i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.112                      100          0 i
*>i                   99.99.99.112                      100          0 i
* i[3]:[0]:[32]:[99.99.99.112]/88
                      99.99.99.112                      100          0 i
*>i                   99.99.99.112                      100          0 i

Route Distinguisher: 172.0.0.100:3    (L3VNI 770000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.0.0.13                        100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0021]:[32]:[10.0.0.21]/272
                      100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0022]:[32]:[20.0.0.22]/272
                      100.0.0.1                         100          0 65254 65002 i
* i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.0.0.13               0        100          0 ?
*>l                   172.0.0.100              0        100      32768 ?
*>l[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.0.0.100              0        100      32768 ?
  ```
</details>

<details> 
<summary><b> Leaf13</b></summary>
 
  ```
Leaf13# show bgp l2vpn evpn summary 
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 172.0.0.13, local AS number 65001
BGP table version is 20311, L2VPN EVPN config peers 2, capable peers 2
29 network entries and 48 paths using 7316 bytes of memory
BGP attribute entries [38/6536], BGP AS path entries [1/10]
BGP community entries [0/0], BGP clusterlist entries [4/16]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.0.0.111     4 65001  646115  642522    20311    0    0     1w2d 14        
172.0.0.112     4 65001  837586  830473    20311    0    0 01:15:13 14        

Leaf13# show bgp l2vpn evpn 
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 20311, Local Router ID is 172.0.0.13
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 1:10000
* i[2]:[0]:[0]:[48]:[0000.0000.0021]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>i                   100.0.0.1                         100          0 65254 65002 i
* i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>i                   100.0.0.1                         100          0 65254 65002 i
* i[2]:[0]:[0]:[48]:[0000.0000.0021]:[32]:[10.0.0.21]/272
                      100.0.0.1                         100          0 65254 65002 i
*>i                   100.0.0.1                         100          0 65254 65002 i

Route Distinguisher: 1:20000
* i[2]:[0]:[0]:[48]:[0000.0000.0022]:[32]:[20.0.0.22]/272
                      100.0.0.1                         100          0 65254 65002 i
*>i                   100.0.0.1                         100          0 65254 65002 i

Route Distinguisher: 172.0.0.11:32777
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.0.0.100                       100          0 i
*>i                   172.0.0.100                       100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.0.0.100                       100          0 i
*>i                   172.0.0.100                       100          0 i

Route Distinguisher: 172.0.0.11:32787
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.0.0.100                       100          0 i
*>i                   172.0.0.100                       100          0 i

Route Distinguisher: 172.0.0.13:32777    (L2VNI 10000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.0.0.100                       100          0 i
* i                   172.0.0.100                       100          0 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      172.0.0.13                        100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0021]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.111                      100          0 i
*>i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.112                      100          0 i
*>i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.0.0.100                       100          0 i
* i                   172.0.0.100                       100          0 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      172.0.0.13                        100      32768 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0021]:[32]:[10.0.0.21]/272
                      100.0.0.1                         100          0 65254 65002 i

Route Distinguisher: 172.0.0.100:3
* i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.0.0.100              0        100          0 ?
*>i                   172.0.0.100              0        100          0 ?
* i[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.0.0.100              0        100          0 ?
*>i                   172.0.0.100              0        100          0 ?

Route Distinguisher: 172.0.0.100:32777
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      172.0.0.100                       100          0 i
*>i                   172.0.0.100                       100          0 i
* i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.0.0.100                       100          0 i
*>i                   172.0.0.100                       100          0 i

Route Distinguisher: 172.0.0.100:32787
* i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.0.0.100                       100          0 i
*>i                   172.0.0.100                       100          0 i

Route Distinguisher: 172.0.0.111:32777
* i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.111                      100          0 i
*>i                   99.99.99.111                      100          0 i

Route Distinguisher: 172.0.0.112:32777
*>i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.112                      100          0 i
* i                   99.99.99.112                      100          0 i

Route Distinguisher: 172.0.0.13:3    (L3VNI 770000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      172.0.0.100                       100          0 i
* i                   172.0.0.100                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      172.0.0.100                       100          0 i
* i                   172.0.0.100                       100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0021]:[32]:[10.0.0.21]/272
                      100.0.0.1                         100          0 65254 65002 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0022]:[32]:[20.0.0.22]/272
                      100.0.0.1                         100          0 65254 65002 i
* i[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.0.0.100              0        100          0 ?
*>l                   172.0.0.13               0        100      32768 ?
*>i[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.0.0.100              0        100          0 ?
```
</details>

<details> 
<summary><b> Leaf21</b></summary>

```
Leaf21# sh bgp l2vpn evpn summary 
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 172.0.0.21, local AS number 65002
BGP table version is 3617, L2VPN EVPN config peers 1, capable peers 1
36 network entries and 36 paths using 6864 bytes of memory
BGP attribute entries [34/5848], BGP AS path entries [1/10]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.0.0.113     4 65002   28066   25781     3617    0    0    4d22h 13        

Leaf21# sh bgp l2vpn evpn 
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 3617, Local Router ID is 172.0.0.21
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 2:10000
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      100.0.0.2                         100          0 65254 65001 i

Route Distinguisher: 2:20000
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      100.0.0.2                         100          0 65254 65001 i

Route Distinguisher: 172.0.0.21:32777    (L2VNI 10000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0021]:[0]:[0.0.0.0]/216
                      172.0.0.21                        100      32768 i
*>i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.113                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      100.0.0.2                         100          0 65254 65001 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0021]:[32]:[10.0.0.21]/272
                      172.0.0.21                        100      32768 i

Route Distinguisher: 172.0.0.21:32787    (L2VNI 20000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0022]:[0]:[0.0.0.0]/216
                      172.0.0.21                        100      32768 i
*>i[2]:[0]:[0]:[48]:[5000.0400.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[5000.0500.1b08]:[0]:[0.0.0.0]/216
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.113                      100          0 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      100.0.0.2                         100          0 65254 65001 i
*>l[2]:[0]:[0]:[48]:[0000.0000.0022]:[32]:[20.0.0.22]/272
                      172.0.0.21                        100      32768 i
*>i[3]:[0]:[32]:[99.99.99.113]/88
                      99.99.99.113                      100          0 i
*>l[3]:[0]:[32]:[172.0.0.21]/88
                      172.0.0.21                        100      32768 i

Route Distinguisher: 172.0.0.113:32777
*>i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.113                      100          0 i

Route Distinguisher: 172.0.0.113:32787
*>i[2]:[0]:[0]:[48]:[5000.f300.1b08]:[0]:[0.0.0.0]/216
                      99.99.99.113                      100          0 i
*>i[3]:[0]:[32]:[99.99.99.113]/88
                      99.99.99.113                      100          0 i

Route Distinguisher: 172.0.0.21:3    (L3VNI 770000)
*>i[2]:[0]:[0]:[48]:[0000.0000.0011]:[32]:[10.0.0.11]/272
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0012]:[32]:[20.0.0.12]/272
                      100.0.0.2                         100          0 65254 65001 i
*>i[2]:[0]:[0]:[48]:[0000.0000.0013]:[32]:[10.0.0.13]/272
                      100.0.0.2                         100          0 65254 65001 i
*>l[5]:[0]:[0]:[24]:[10.0.0.0]/224
                      172.0.0.21               0        100      32768 ?
*>l[5]:[0]:[0]:[24]:[20.0.0.0]/224
                      172.0.0.21               0        100      32768 ?
```
</details>

### VXLAN (NVE)
Проверяем, что NVE-интерфейс поднят, VTEP-ы видят друг друга, VNI активны. Spine1 быд выбран DF для vlan10 и vlan20, а Spine2 - vlan77.

```
Spine1# show nve interface nve 1
Interface: nve1, State: Up, encapsulation: VXLAN
 VPC Capability: VPC-VIP-Only [not-notified]
 Local Router MAC: 5000.0400.1b08
 Host Learning Mode: Control-Plane
 Source-Interface: loopback99 (primary: 99.99.99.111, secondary: 0.0.0.0)

Spine1# show nve peers 
Interface Peer-IP                                 State LearnType Uptime   Route
r-Mac       
--------- --------------------------------------  ----- --------- -------- -----
------------
nve1      99.99.99.112                            Up    CP        01:37:55 n/a             
nve1      99.99.99.113                            Up    CP        2d03h    n/a  
nve1      100.0.0.2                               Up    CP        4d22h    0200.6400.0002   
nve1      172.0.0.13                              Up    CP        1w2d     5000.f200.1b08   
nve1      172.0.0.100                             Up    CP        2d06h    5000.0200.1b08   

Spine1# show nve vni 
Codes: CP - Control Plane        DP - Data Plane          
       UC - Unconfigured         SA - Suppress ARP        
       SU - Suppress Unknown Unicast 
       Xconn - Crossconnect      
       MS-IR - Multisite Ingress Replication
 
Interface VNI      Multicast-group   State Mode Type [BD/VRF]      Flags
--------- -------- ----------------- ----- ---- ------------------ -----
nve1      10000    225.0.0.10        Up    CP   L2 [10]            MS-IR 
nve1      20000    UnicastBGP        Up    CP   L2 [20]            MS-IR 
nve1      770000   n/a               Up    CP   L3 [VRF_A]              

Spine1# show nve vni 10000 detail 
VNI: 10000 
  NVE-Interface       : nve1
  Mcast-Addr          : 225.0.0.10
  VNI State           : Up
  Mode                : control-plane
  VNI Type            : L2 [10]
  VNI Flags           : MS-IR 
  Provision State     : vni-add-complete
  Vlan-BD             : 10
  SVI State           : n/a

Spine1# show nve vni 20000 detail 
VNI: 20000 
  NVE-Interface       : nve1
  Mcast-Addr          : UnicastBGP
  VNI State           : Up
  Mode                : control-plane
  VNI Type            : L2 [20]
  VNI Flags           : MS-IR 
  Provision State     : vni-add-complete
  Vlan-BD             : 20
  SVI State           : n/a

Spine1# sh nve ethernet-segment 

ESI: 0300.0000.0000.0100.0309
   Parent interface: nve1
  ES State: Up 
  Port-channel state: N/A
  NVE Interface: nve1 
   NVE State: Up 
   Host Learning Mode: control-plane
  Active Vlans: 1,10,20,77 
   DF Vlans: 10,20 
   Active VNIs: 10000,20000,770000 
  CC failed for VLANs:  
  VLAN CC timer: no-timer 
  Number of ES members: 2 
  My ordinal: 0 
  DF timer start time: 00:00:00 
  Config State: N/A 
  DF List: 99.99.99.111 99.99.99.112  
  ES route added to L2RIB: True
  EAD/ES routes added to L2RIB: False
  EAD/EVI route timer age: not running 
```

```
Spine2# sh nve interface nve 1
Interface: nve1, State: Up, encapsulation: VXLAN
 VPC Capability: VPC-VIP-Only [not-notified]
 Local Router MAC: 5000.0500.1b08
 Host Learning Mode: Control-Plane
 Source-Interface: loopback99 (primary: 99.99.99.112, secondary: 0.0.0.0)

Spine2# show nve peers 
Interface Peer-IP                                 State LearnType Uptime   Route
r-Mac       
--------- --------------------------------------  ----- --------- -------- -----
------------
nve1      99.99.99.111                            Up    CP        01:43:06 n/a              
nve1      99.99.99.113                            Up    CP        01:43:06 n/a             
nve1      100.0.0.2                               Up    CP        01:43:29 0200.6400.0002   
nve1      172.0.0.13                              Up    CP        01:43:29 5000.f200.1b08   
nve1      172.0.0.100                             Up    CP        01:43:29 5000.0200.1b08   

Spine2# show nve vni 
Codes: CP - Control Plane        DP - Data Plane          
       UC - Unconfigured         SA - Suppress ARP        
       SU - Suppress Unknown Unicast 
       Xconn - Crossconnect      
       MS-IR - Multisite Ingress Replication
 
Interface VNI      Multicast-group   State Mode Type [BD/VRF]      Flags
--------- -------- ----------------- ----- ---- ------------------ -----
nve1      10000    225.0.0.10        Up    CP   L2 [10]            MS-IR 
nve1      20000    UnicastBGP        Up    CP   L2 [20]            MS-IR 
nve1      770000   n/a               Up    CP   L3 [VRF_A]              

Spine2# show nve vni 10000 detail 
VNI: 10000 
  NVE-Interface       : nve1
  Mcast-Addr          : 225.0.0.10
  VNI State           : Up
  Mode                : control-plane
  VNI Type            : L2 [10]
  VNI Flags           : MS-IR 
  Provision State     : vni-add-complete
  Vlan-BD             : 10
  SVI State           : n/a

Spine2# show nve vni 20000 detail 
VNI: 20000 
  NVE-Interface       : nve1
  Mcast-Addr          : UnicastBGP
  VNI State           : Up
  Mode                : control-plane
  VNI Type            : L2 [20]
  VNI Flags           : MS-IR 
  Provision State     : vni-add-complete
  Vlan-BD             : 20
  SVI State           : n/a

Spine2# sh nve ethernet-segment 

ESI: 0300.0000.0000.0100.0309
   Parent interface: nve1
  ES State: Up 
  Port-channel state: N/A
  NVE Interface: nve1 
   NVE State: Up 
   Host Learning Mode: control-plane
  Active Vlans: 1,10,20,77 
   DF Vlans: 1,77 
   Active VNIs: 10000,20000,770000 
  CC failed for VLANs:  
  VLAN CC timer: no-timer 
  Number of ES members: 2 
  My ordinal: 1 
  DF timer start time: 00:00:00 
  Config State: N/A 
  DF List: 99.99.99.111 99.99.99.112  
  ES route added to L2RIB: True
  EAD/ES routes added to L2RIB: False
  EAD/EVI route timer age: not running 
```

```
Spine3# show nve interface nve 1
Interface: nve1, State: Up, encapsulation: VXLAN
 VPC Capability: VPC-VIP-Only [not-notified]
 Local Router MAC: 5000.f300.1b08
 Host Learning Mode: Control-Plane
 Source-Interface: loopback99 (primary: 99.99.99.113, secondary: 0.0.0.0)

Spine3# show nve peers 
Interface Peer-IP                                 State LearnType Uptime   Route
r-Mac       
--------- --------------------------------------  ----- --------- -------- -----
------------
nve1      99.99.99.111                            Up    CP        2d03h    n/a            
nve1      99.99.99.112                            Up    CP        01:50:09 n/a            
nve1      100.0.0.1                               Up    CP        4d22h    0200.6400.0001   
nve1      172.0.0.21                              Up    CP        4d22h    5000.f400.1b08   

Spine3# show nve vni 
Codes: CP - Control Plane        DP - Data Plane          
       UC - Unconfigured         SA - Suppress ARP        
       SU - Suppress Unknown Unicast 
       Xconn - Crossconnect      
       MS-IR - Multisite Ingress Replication
 
Interface VNI      Multicast-group   State Mode Type [BD/VRF]      Flags
--------- -------- ----------------- ----- ---- ------------------ -----
nve1      10000    225.0.0.10        Up    CP   L2 [10]            MS-IR 
nve1      20000    UnicastBGP        Up    CP   L2 [20]            MS-IR 
nve1      770000   n/a               Up    CP   L3 [VRF_A]              

Spine3# show nve vni 10000 detail 
VNI: 10000 
  NVE-Interface       : nve1
  Mcast-Addr          : 225.0.0.10
  VNI State           : Up
  Mode                : control-plane
  VNI Type            : L2 [10]
  VNI Flags           : MS-IR 
  Provision State     : vni-add-complete
  Vlan-BD             : 10
  SVI State           : n/a

Spine3# show nve vni 20000 detail 
VNI: 20000 
  NVE-Interface       : nve1
  Mcast-Addr          : UnicastBGP
  VNI State           : Up
  Mode                : control-plane
  VNI Type            : L2 [20]
  VNI Flags           : MS-IR 
  Provision State     : vni-add-complete
  Vlan-BD             : 20
  SVI State           : n/a
```

### VXLAN (L2RIB)
Проверяем, что MAC-адреса серверов изучены и привязаны к правильным VTEP.

```
Leaf11# show vlan 

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    
10   SERVERS_V10                      active    Po1, Po10, Eth1/4, Eth1/6
                                                Eth1/7
20   SERVERS_V20                      active    Po1, Po20, Eth1/5, Eth1/6
                                                Eth1/7
77   L3VNI                            active    Po1, Eth1/6, Eth1/7
3900 BACKUP_VLAN_ROUTING              active    Po1, Eth1/6, Eth1/7

VLAN Type         Vlan-mode
---- -----        ----------
1    enet         CE     
10   enet         CE     
20   enet         CE     
77   enet         CE     
3900 enet         CE     

Remote SPAN VLANs
-------------------------------------------------------------------------------

Primary  Secondary  Type             Ports
-------  ---------  ---------------  -------------------------------------------

Leaf11# show mac
mac        mac-list   
Leaf11# show mac address-table vlan 10
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
+   10     0000.0000.0011   dynamic  0         F      F    Po10
C   10     0000.0000.0013   dynamic  0         F      F    nve1(172.0.0.13)
C   10     0000.0000.0021   dynamic  0         F      F    nve1(100.0.0.1)
G   10     5000.0100.1b08   static   -         F      F    vPC Peer-Link(R)
G   10     5000.0200.1b08   static   -         F      F    sup-eth1(R)
Leaf11# show mac address-table vlan 20
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
+   20     0000.0000.0012   dynamic  0         F      F    Po20
C   20     0000.0000.0022   dynamic  0         F      F    nve1(100.0.0.1)
G   20     5000.0100.1b08   static   -         F      F    vPC Peer-Link(R)
G   20     5000.0200.1b08   static   -         F      F    sup-eth1(R)
```

```
Leaf13# sh vlan 

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    
10   SERVERS                          active    Eth1/3
77   L3VNI                            active    

VLAN Type         Vlan-mode
---- -----        ----------
1    enet         CE     
10   enet         CE     
77   enet         CE     

Remote SPAN VLANs
-------------------------------------------------------------------------------

Primary  Secondary  Type             Ports
-------  ---------  ---------------  -------------------------------------------

Leaf13# show mac
mac        mac-list   
Leaf13# show mac address-table vlan 10
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
C   10     0000.0000.0011   dynamic  0         F      F    nve1(172.0.0.100)
*   10     0000.0000.0013   dynamic  0         F      F    Eth1/3
C   10     0000.0000.0021   dynamic  0         F      F    nve1(100.0.0.1)
G   10     5000.f200.1b08   static   -         F      F    sup-eth1(R)
```

```
Leaf21# sh vlan 

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    
10   SERVERS_V10                      active    Eth1/4
20   SERVERS_V20                      active    Eth1/5
77   L3VNI                            active    

VLAN Type         Vlan-mode
---- -----        ----------
1    enet         CE     
10   enet         CE     
20   enet         CE     
77   enet         CE     

Remote SPAN VLANs
-------------------------------------------------------------------------------

Primary  Secondary  Type             Ports
-------  ---------  ---------------  -------------------------------------------

Leaf21# sh mac address-table vlan 10
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
C   10     0000.0000.0011   dynamic  0         F      F    nve1(100.0.0.2)
C   10     0000.0000.0013   dynamic  0         F      F    nve1(100.0.0.2)
*   10     0000.0000.0021   dynamic  0         F      F    Eth1/4
G   10     5000.f400.1b08   static   -         F      F    sup-eth1(R)

Leaf21# sh mac address-table vlan 20
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
C   20     0000.0000.0012   dynamic  0         F      F    nve1(100.0.0.2)
*   20     0000.0000.0022   dynamic  0         F      F    Eth1/5
G   20     5000.f400.1b08   static   -         F      F    sup-eth1(R)
```

### Мультикаст (PIM) — только для VLAN 10
В рамках проекта для доставки BUM-трафика внутри VLAN 10 используется PIM Sparse-Mode (PIM-SM). Все три пограничных коммутатора (Spine1, Spine2, Spine3) входят в единую группу Anycast-RP с общим виртуальным адресом 172.0.0.1. Это позволяет коммутаторам (Leaf) взаимодействовать с ближайшим RP без привязки к конкретному устройству.

Ключевые особенности конфигурации:

* RP-адрес 172.0.0.1 объявлен на всех Spine с помощью команды ip pim rp-address.
* Группа Anycast-RP настроена через ip pim anycast-rp 172.0.0.1 <уникальный IP>.
* Межсайтовый обмен информацией об активных источниках (S,G) между Spine1/Spine2 (Site 1) и Spine3 (Site 2) осуществляется через MSDP. Это позволяет Spine3 узнавать об источниках из Site 1 и наоборот.
* Команда multisite ingress-replication на пограничных шлюзах (Spine1, Spine2, Spine3) для VNI 10000 говорит, что BUM-трафик для удалённого сайта должен быть отправлен с помощью Ingress Replication.
* Для Underlay-связности MSDP-пиров используются адреса loopback99 (99.99.99.111/112/113), которые имеют маршруты через SuperSpine.

```
Leaf11# sh ip pim  neighbor 
PIM Neighbor Status for VRF "default"
Neighbor        Interface            Uptime    Expires   DR       Bidir-  BFD   
 ECMP Redirect
                                                         Priority Capable State     Capable
172.16.11.1     Ethernet1/1          1w2d      00:01:19  1        yes     n/a       no
172.16.12.1     Ethernet1/2          02:34:15  00:01:27  1        yes     n/a       no

Leaf11# 
Leaf11# sh ip pim rp 
PIM RP Status Information for VRF "default"
BSR disabled
Auto-RP disabled
BSR RP Candidate policy: None
BSR RP policy: None
Auto-RP Announce policy: None
Auto-RP Discovery policy: None

RP: 172.0.0.1, (0), 
 uptime: 6w5d   priority: 255, 
 RP-source: (local),  
 group ranges:
 225.0.0.0/24   
Leaf11# 
Leaf11# sh ip mroute 225.0.0.10
IP Multicast Routing Table for VRF "default"

(*, 225.0.0.10/32), uptime: 7w0d, nve ip pim 
  Incoming interface: Ethernet1/2, RPF nbr: 172.16.12.1
  Outgoing interface list: (count: 1)
    nve1, uptime: 7w0d, nve


(172.0.0.21/32, 225.0.0.10/32), uptime: 00:09:54, ip pim mrib 
  Incoming interface: loopback0, RPF nbr: 172.0.0.21
  Outgoing interface list: (count: 2)
    Ethernet1/2, uptime: 00:09:53, pim
    nve1, uptime: 00:09:54, mrib


(172.0.0.100/32, 225.0.0.10/32), uptime: 7w0d, nve mrib ip pim 
  Incoming interface: loopback0, RPF nbr: 172.0.0.100
  Outgoing interface list: (count: 1)
    Ethernet1/2, uptime: 2d23h, pim
```

```
Spine1(config)# sh ip mroute 225.0.0.10
IP Multicast Routing Table for VRF "default"

(*, 225.0.0.10/32), uptime: 01:15:41, nve pim ip 
  Incoming interface: loopback1, RPF nbr: 172.0.0.1
  Outgoing interface list: (count: 2)
    Ethernet1/3, uptime: 01:15:23, pim
    nve1, uptime: 01:15:41, nve


(99.99.99.111/32, 225.0.0.10/32), uptime: 01:15:41, nve mrib pim ip 
  Incoming interface: Null, RPF nbr: 0.0.0.0
  Outgoing interface list: (count: 1)
    Ethernet1/3, uptime: 01:15:23, pim


(172.0.0.13/32, 225.0.0.10/32), uptime: 01:15:11, pim mrib ip msdp 
  Incoming interface: Ethernet1/3, RPF nbr: 172.16.31.0, internal
  Outgoing interface list: (count: 2)
    Ethernet1/3, uptime: 01:04:05, pim, (RPF)
    nve1, uptime: 01:15:11, mrib


(172.0.0.21/32, 225.0.0.10/32), uptime: 00:00:31, msdp mrib ip pim 
  Incoming interface: Ethernet1/2, RPF nbr: 172.16.21.0, internal
  Outgoing interface list: (count: 1)
    nve1, uptime: 00:00:31, mrib


(172.0.0.100/32, 225.0.0.10/32), uptime: 01:15:23, pim mrib ip msdp 
  Incoming interface: Ethernet1/2, RPF nbr: 172.16.21.0, internal
  Outgoing interface list: (count: 2)
    nve1, uptime: 01:15:23, mrib
    Ethernet1/3, uptime: 01:15:23, pim
```

```
Leaf21# show ip pim neighbor 
PIM Neighbor Status for VRF "default"
Neighbor        Interface            Uptime    Expires   DR       Bidir-  BFD   
 ECMP Redirect
                                                         Priority Capable State 
    Capable
172.16.211.0    Ethernet1/1          4d23h     00:01:41  1        yes     n/a   
  no
Leaf21# 
Leaf21# sh ip pim rp
rp        rp-hash   
Leaf21# sh ip pim rp 
PIM RP Status Information for VRF "default"
BSR disabled
Auto-RP disabled
BSR RP Candidate policy: None
BSR RP policy: None
Auto-RP Announce policy: None
Auto-RP Discovery policy: None

RP: 172.0.0.2, (0), 
 uptime: 6w5d   priority: 255, 
 RP-source: (local),  
 group ranges:
 225.0.0.0/24   

Leaf21# sh ip mroute 225.0.0.10 detail 
IP Multicast Routing Table for VRF "default"

Total number of routes: 3
Total number of (*,G) routes: 1
Total number of (S,G) routes: 1
Total number of (*,G-prefix) routes: 1

(*, 225.0.0.10/32), uptime: 6w5d, nve(1) ip(0) pim(0) 
  RPF Change only
  RPF-Source: 172.0.0.2 [41/110]
  Data Created: No
  Nat Mode: Invalid
  Nat Route Type: Invalid
  VXLAN Flags
    VXLAN Encap
    VXLAN Last Hop
  Stats: 0/0 [Packets/Bytes], 0.000   bps
  Stats: Inactive Flow
  Incoming interface: Ethernet1/1, RPF nbr: 172.16.211.0
  LISP dest context id: 0  Outgoing interface list: (count: 1) (bridge-only: 0)
    nve1, uptime: 6w5d, nve

(172.0.0.21/32, 225.0.0.10/32), uptime: 6w5d, ip(0) nve(0) mrib(0) pim(1) 
  RPF-Source: 172.0.0.21 [0/0]
  Data Created: No
  Received Register stop
  Nat Mode: Invalid
```

```
Spine3# sh ip pim neighbor 
PIM Neighbor Status for VRF "default"
Neighbor        Interface            Uptime    Expires   DR       Bidir-  BFD   
 ECMP Redirect
                                                         Priority Capable State 
    Capable
172.16.211.1    Ethernet1/1          4d23h     00:01:34  1        yes     n/a   
  no
Spine3# 
Spine3# sh ip pim rp
PIM RP Status Information for VRF "default"
BSR disabled
Auto-RP disabled
BSR RP Candidate policy: None
BSR RP policy: None
Auto-RP Announce policy: None
Auto-RP Discovery policy: None

Anycast-RP 172.0.0.2 members:
  172.0.0.113*  

RP: 172.0.0.2*, (0), 
 uptime: 4d23h   priority: 255, 
 RP-source: (local),  
 group ranges:
 225.0.0.0/24   
Spine3# 
Spine3# sh ip mroute 225.0.0.10 detail 
IP Multicast Routing Table for VRF "default"

Total number of routes: 4
Total number of (*,G) routes: 1
Total number of (S,G) routes: 2
Total number of (*,G-prefix) routes: 1

(*, 225.0.0.10/32), uptime: 4d23h, pim(1) ip(0) nve(1) 
  RPF-Source: 172.0.0.2 [0/0]
  Data Created: No
  Nat Mode: Invalid
  Nat Route Type: Invalid
  VXLAN Flags
    VXLAN Encap
    VXLAN Last Hop
  Stats: 0/0 [Packets/Bytes], 0.000   bps
  Stats: Inactive Flow
  Incoming interface: loopback1, RPF nbr: 172.0.0.2
  LISP dest context id: 0  Outgoing interface list: (count: 2) (bridge-only: 0)
    nve1, uptime: 4d23h, nve
    Ethernet1/1, uptime: 4d23h, pim


(99.99.99.113/32, 225.0.0.10/32), uptime: 4d23h, mrib(0) pim(1) ip(0) nve(0) 
  RPF-Source: 99.99.99.113 [0/0]
  Data Created: No
  Nat Mode: Invalid
  Nat Route Type: Invalid
  VXLAN Flags
    VXLAN Encap
  Stats: 0/0 [Packets/Bytes], 0.000   bps
  Stats: Inactive Flow
  Incoming interface: Null, RPF nbr: 0.0.0.0
  LISP dest context id: 0  Outgoing interface list: (count: 1) (bridge-only: 0)
    Ethernet1/1, uptime: 4d23h, pim


(172.0.0.21/32, 225.0.0.10/32), uptime: 4d23h, pim(1) mrib(1) ip(0) 
  RPF-Source: 172.0.0.21 [41/110]
  Data Created: No
  Nat Mode: Invalid
  Nat Route Type: Invalid
  VXLAN Flags
    VXLAN Decap
  Stats: 0/0 [Packets/Bytes], 0.000   bps
  Stats: Inactive Flow
  Incoming interface: Ethernet1/1, RPF nbr: 172.16.211.1, internal
  LISP dest context id: 0  Outgoing interface list: (count: 2) (bridge-only: 0)
    Ethernet1/1, uptime: 2d04h, pim, (RPF)
    nve1, uptime: 4d23h, mrib
```

### Ingress Replication — только для VLAN 20
В рамках проекта для доставки BUM-трафика внутри VLAN 20 используется Ingress Replication. В отличие от PIM, этот метод не требует отдельной мультикаст-инфраструктуры и основан исключительно BGP EVPN.

Ключевые особенности конфигурации:

* На коммутаторах (Leaf11, Leaf12, Leaf21) в секции member vni 20000 настроена команда ingress-replication protocol bgp.
* Каждый Leaf автоматически анонсирует через BGP EVPN маршруты типа 3, указывая, какие VNI он обслуживает.
* На основе полученных маршрутов каждый Leaf формирует динамический список VTEP-адресов, которым необходимо реплицировать BUM-трафик для VLAN 20.
* Межсайтовое взаимодействие для VLAN 20 осуществляется через SuperSpine. На пограничных шлюзах (Spine1, Spine2, Spine3) в конфигурации VNI 20000 присутствует команда multisite ingress-replication, которая обеспечивает передачу реплицированного BUM-трафика между сайтами.

```
VNI: 20000 
  NVE-Interface       : nve1
  Mcast-Addr          : UnicastBGP
  VNI State           : Up
  Mode                : control-plane
  VNI Type            : L2 [20]
  VNI Flags           : MS-IR 
  Provision State     : vni-add-complete
  Vlan-BD             : 20
  SVI State           : n/a

Leaf11# show bgp l2vpn evpn route-type 3 | i 20000
Route Distinguisher: 1:20000
             Imported paths list: L2-20000
      Extcommunity: RT:9999:20000 ENCAP:8
        Label: 20000, Tunnel Id: 99.99.99.111
             Imported paths list: L2-20000
      Extcommunity: RT:9999:20000 ENCAP:8
        Label: 20000, Tunnel Id: 99.99.99.112
Route Distinguisher: 172.0.0.11:32787    (L2VNI 20000)
             Imported from 1:20000:[3]:[0]:[32]:[99.99.99.111]/88 
      Extcommunity: RT:9999:20000 ENCAP:8
        Label: 20000, Tunnel Id: 99.99.99.111
      Extcommunity: RT:9999:20000 ENCAP:8
        Label: 20000, Tunnel Id: 99.99.99.111
             Imported from 1:20000:[3]:[0]:[32]:[99.99.99.112]/88 
      Extcommunity: RT:9999:20000 ENCAP:8
        Label: 20000, Tunnel Id: 99.99.99.112
      Extcommunity: RT:9999:20000 ENCAP:8
        Label: 20000, Tunnel Id: 99.99.99.112
      Extcommunity: RT:9999:20000 ENCAP:8
        Label: 20000, Tunnel Id: 172.0.0.100
      Extcommunity: RT:9999:20000 ENCAP:8
        Label: 20000, Tunnel Id: 99.99.99.111
             Imported paths list: L2-20000
      Extcommunity: RT:9999:20000 ENCAP:8
        Label: 20000, Tunnel Id: 99.99.99.111
      Extcommunity: RT:9999:20000 ENCAP:8
        Label: 20000, Tunnel Id: 99.99.99.112
             Imported paths list: L2-20000
      Extcommunity: RT:9999:20000 ENCAP:8
        Label: 20000, Tunnel Id: 99.99.99.112
```




### Проверка связанности хостов

```
Ep1#ping 10.0.0.13
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.13, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 812/1318/1837 ms

Ep1#ping 10.0.0.21
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.21, timeout is 2 seconds:
!!!!![Презентация по проекту.pptx](https://github.com/user-attachments/files/29159059/default.pptx)

Success rate is 100 percent (5/5), round-trip min/avg/max = 1335/1678/1941 ms

Ep1#ping 20.0.0.12
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 20.0.0.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 410/738/1094 ms

Ep1#ping 20.0.0.22
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 20.0.0.22, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1124/1371/1635 ms

Ep2#ping 10.0.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.11, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 513/688/882 ms

Ep2#ping 20.0.0.22
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 20.0.0.22, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1170/1373/1646 ms

Ep2#ping 10.0.0.13
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.13, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 779/1261/1834 ms

Ep2#ping 10.0.0.21
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.21, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1272/1650/1951 ms

```


### Выводы

```
  В рамках проекта успешно реализована комбинированная модель пересылки BUM-трафика:

  PIM Sparse-Mode с Anycast-RP — используется для VLAN 10. В проекте показана его работа внутри сайтов и синхронизация
источников между сайтами через MSDP. Ingress Replication на базе BGP EVPN — обеспечивает доставку BUM для VLAN 20. Это
более простой и предсказуемый метод: диагностика сводится к проверке BGP EVPN (маршруты типа 3) и связности NVE-пиров.
EVPN Multi-Site через SuperSpine — позволяет объединять независимые EVPN-фабрики (разные AS) с сохранением next-hop маршрутов.
vPC на Leaf11/Leaf12 и Anycast-шлюзы — гарантируют отказоустойчивость серверных сетей.
Проект демонстрирует возможность сосуществования Multicast с Ingress Replication в одной фабрике.
Это полезно для плановой миграции: администратор может переводить VLAN за VLAN с PIM на Ingress Replication и наоборот,
не останавливая сервисы. Такое решение рекомендовано для крупных ЦОД, где требуется баланс между эффективностью использования
полосы пропускания и совместимостью с оборудованием, не поддерживающим PIM во всех сегментах.
```
[Презентация по проекту.pptx](https://github.com/user-attachments/files/29159064/default.pptx)
