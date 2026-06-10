# Комбинированная модель пересылки BUM-трафика (PIM + Ingress Replication) в архитектуре VXLAN EVPN Multi-Site

## Описание проекта

В рамках данной лабораторной работы реализована **многосайтовая архитектура EVPN VXLAN** с использованием комбинированной модели пересылки неизвестного, широковещательного и многоадресного (BUM) трафика. Решение сочетает два подхода:

* **PIM (Protocol Independent Multicast)** — для оптимизации рассылки BUM-трафика внутри одного сайта (Site 1) в сегменте VLAN 10.
* **Ingress Replication** — для BUM-трафика VLAN 20 в пределах Site 1, а также для всех межсайтовых взаимодействий между Site 1 и Site 2.

Такая гибридная схема позволяет снизить нагрузку на контроллеры и маршрутизаторы, обеспечивая гибкость при построении отказоустойчивых ЦОД.

## Описание проекта

В рамках данной лабораторной работы реализована **многосайтовая архитектура VXLAN EVPN Multi-Site**, демонстрирующая сосуществование двух моделей пересылки BUM-трафика:

* **PIM (Protocol Independent Multicast)** — используется внутри Site 1 и Site 2 для BUM-трафика VLAN 10.
* **Ingress Replication** — используется внутри Site 1 и Site 2 для BUM-трафика VLAN 20.

### Как работает пересылка BUM-трафика в EVPN Multi-Site?

Ключевая особенность технологии EVPN Multi-Site:

* **Внутри сайта (intra-site):** Метод репликации определяется настройками на листьях. В данном проекте для VLAN 10 используется PIM (на обоих сайтах), для VLAN 20 — Ingress Replication (на обоих сайтах).
* **Между сайтами (inter-site):** BUM-трафик передаётся **только через Ingress Replication** — это обязательное требование технологии Multi-Site. Согласно официальной документации Cisco, **до версии NX-OS 10.2(2)F между DCI-пирами поддерживается только Ingress Replication**.

На пограничных шлюзах (BGW) команда `multisite ingress-replication` включает именно этот механизм межсайтовой репликации. Она определяет метод репликации BUM-трафика для расширения Layer 2 VNI между сайтами.

> **Источник:** Официальная документация Cisco NX-OS VXLAN Configuration Guide, раздел "Configure VNI dual mode":  
> *"Defines the Multi-Site BUM replication method for extending the Layer 2 VNI"*  
> `switch(config-if-nve-vni)# multisite ingress-replication`  
> — [Cisco NX-OS VXLAN Configuration Guide, Release 10.4(x), стр. 15-16](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/104x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-104x/m_configuring_multisite_93x.pdf#page=15)

### Зачем нужна такая комбинация?

Данная конфигурация показывает два сценария:

1. **Для VLAN 10** — демонстрация того, как PIM, используемый внутри каждого сайта, стыкуется с обязательным Ingress Replication на границе сайта. Это типичный сценарий миграции или сосуществования legacy-мультикаста с современной EVPN Multi-Site архитектурой.

2. **Для VLAN 20** — сценарий, где Ingress Replication используется везде (и внутри сайтов, и между ними).

**Важное замечание про PIM:** PIM сложнее в траблшутинге — требует проверки RPF (обратного пути), наличия (*,G) и (S,G) записей в mroute. Основные проблемы с RP включают: недоступность RP по unicast-маршруту, рассогласование адресов RP на разных устройствах, неправильный RPF-путь к RP (с петлями или asymmetric routing), перегрузку Register-сообщениями при старте источников, а также специфические проблемы Anycast-RP (например, отсутствие синхронизации MSDP или Register, уходящие в никуда при переключении). Ingress Replication проще: при проблемах достаточно проверить BGP EVPN (маршруты типа 3, route-target) и связность NVE-пиров.



## Схема сети

<img src="https://github.com/user-attachments/assets/c6936e33-662d-44dd-a4f1-0d13465b4bf5" width="700" style="max-width: 100%; height: auto;">

*Рисунок 1 — Топология VXLAN EVPN Multi-Site с двумя сайтами и SuperSpine*

### Архитектура и топология

* **SuperSpine** — единая точка подключения для всех сайтов, выполняет маршрутизацию между AS 65001 (Site 1) и AS 65002 (Site 2) через eBGP.
* **Сайт 1 (AS 65001)**: три Spina (Spine1/2/3) и три листа (Leaf11, Leaf12, Leaf13). Spine1 и Spine2 объединены в **Anycast-RP (172.0.0.1)** и являются пограничными шлюзами сайта (BGW) с VIP-адресом 100.0.0.1. Leaf11 и Leaf12 связаны в **vPC**.
* **Сайт 2 (AS 65002)**: один Spine3 (также BGW с VIP 100.0.0.2) и один лист Leaf21.
* **Underlay**: OSPF (processes `UNDERLAY` для сайтов и `DCI` для SuperSpine) с BFD. Адресация P2P (/31), Loopback'и — для идентификации устройств и BGP-пиринга.
* **Overlay**: VXLAN с EVPN-сигнализацией. Маршруты между сайтами распространяются через SuperSpine с сохранением следующего перехода (route-map `NH_UNCH`).

### Цели работы

1. Реализовать **мультисайтовую EVPN-фабрику** с двумя независимыми AS.
2. Настроить **Anycast-RP и PIM SM** для доставки BUM-трафика внутри VLAN 10 (мультикаст-группа 225.0.0.10).
3. Настроить **Ingress Replication** для VLAN 20 (на уровне BGP EVPN).
4. Обеспечить **взаимодействие между сайтами** через SuperSpine с транзитом как L2-, так и L3-маршрутов EVPN.
5. Проверить резервирование и отказоустойчивость за счет vPC и Anycast-шлюзов.

## Адресация и параметры

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

### Spine3 (BGW для Site2, Anycast-RP 172.0.0.2)
<details> 
<summary><b>Конфигурация Spine3</b></summary>
  
```
 hostname Spine3

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
<summary><b>Конфигурация Leaf13</b></summary>
  
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

