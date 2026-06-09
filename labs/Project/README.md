# Комбинированная модель пересылки BUM-трафика (PIM + Ingress Replication) в архитектуре VXLAN EVPN Multi-Site

## Описание проекта

В рамках данной лабораторной работы реализована **многосайтовая архитектура EVPN VXLAN** с использованием комбинированной модели пересылки неизвестного, широковещательного и многоадресного (BUM) трафика. Решение сочетает два подхода:

* **PIM (Protocol Independent Multicast)** — для оптимизации рассылки BUM-трафика внутри одного сайта (Site 1) в сегменте VLAN 10.
* **Ingress Replication** — для BUM-трафика VLAN 20 в пределах Site 1, а также для всех межсайтовых взаимодействий между Site 1 и Site 2.

Такая гибридная схема позволяет снизить нагрузку на контроллеры и маршрутизаторы, обеспечивая гибкость при построении отказоустойчивых ЦОД.

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
<summary><b>Конфигурация SuperSpine (фрагменты)</b></summary>

```cisco
hostname SS
nv overlay evpn
feature ospf bgp bfd
interface Ethernet1/1-3
  description to Spine*
  no switchport
  mtu 9216
  bfd interval 100 min_rx 100 multiplier 5
  ip address 172.16.11x.0/31
  ip ospf authentication message-digest key-chain OSPF_KC
  ip ospf network point-to-point
  ip router ospf UNDERLAY area 0.0.0.0
router bgp 65254
  neighbor 99.99.99.111 remote-as 65001
  neighbor 99.99.99.111 route-map NH_UNCH out
  neighbor 99.99.99.112 remote-as 65001
  neighbor 99.99.99.113 remote-as 65002
  address-family l2vpn evpn
    retain route-target all
    route-map NH_UNCH out
route-map NH_UNCH permit 10
  set ip next-hop unchanged
</details>
