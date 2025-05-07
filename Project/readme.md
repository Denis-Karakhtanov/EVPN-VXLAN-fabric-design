# Архитектура распределённых EVPN/VXLAN-фабрик с междатацентровой связью по L3VPN/MPLS

## Цели проекта и обоснование выбора технологии EVPN/VXLAN

- Повышение отказоустойчивости и надежности сетевой инфраструктуры
- Оптимизация и стабилизация задержек RTT как внутри дата-центра (DC), так и между дата-центрами (DCI)
- Увеличение пропускной способности взаимодействия между хостами
- Обеспечение гибкого и эффективного масштабирования сетевой инфраструктуры



### Рассмотрение архитектуры и технологий типового ЦОД в проекте

#### Структурная схема 


![image](https://github.com/user-attachments/assets/f6937da7-4e64-4f71-bd65-208edf1b7765)



#### Ключевые используемые технологии

- Underlay: Intermediate System to Intermediate System protocol
- Control plane - bgp evpn, data plane - vxlan
- EVPN-VXLAN Edge-Routed Bridging Fabric
- EVPN-VXLAN Central-Routed Bridging Fabric
- EVPN Multihoming
- VRF Route Leaking
- Equal-cost multi-path routing
- EVPN MAC-VRF Routing Instance
- EVPN Asymmetric IRB


#### Примеры настройки типового DC на оборудовании Juniper

##### Underlay: Intermediate System to Intermediate System protocol

Необходимо обеспечить доступность loopback между оборудование фабрики.

Все линки используют point-to-point и level 2 wide-metrics-only, level 1 отключаем.

```
set policy-options policy-statement isis-export term loopback from interface lo0.0
set policy-options policy-statement isis-export term loopback then accept
set policy-options policy-statement isis-export then reject
set protocols isis interface et1 point-to-point
set protocols isis interface et2 point-to-point
set protocols isis interface lo0.0 passive
set protocols isis level 1 disable
set protocols isis level 2 wide-metrics-only
set protocols isis export isis-export
```

Одинаковая конфигурация на leaf-02 и leaf-03, необходимо указать route-target import, которая будет применяться и на экспорт.

ESI метка создается вручную в произвольном формате начиная с 0000:


### Проверка доступности хостов и анонсов

```
leaf-01#show bgp evpn route-type ethernet-segment
BGP routing table information for VRF default
Router identifier 1.1.2.1, local AS number 4200100100
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 1.1.2.2:1 ethernet-segment 0000:0000:0000:0000:0010 fd00:c1::202
                                 fd00:c1::202          -       100     0       i Or-ID: 1.1.2.2 C-LST: 1.1.1.1
 *  ec    RD: 1.1.2.2:1 ethernet-segment 0000:0000:0000:0000:0010 fd00:c1::202
                                 fd00:c1::202          -       100     0       i Or-ID: 1.1.2.2 C-LST: 1.1.1.2
```


```
vpc-1> ping 2001:679:1024:1::12

2001:679:1024:1::12 icmp6_seq=1 ttl=64 time=249.987 ms
2001:679:1024:1::12 icmp6_seq=2 ttl=64 time=50.019 ms
2001:679:1024:1::12 icmp6_seq=3 ttl=64 time=48.323 ms
2001:679:1024:1::12 icmp6_seq=4 ttl=64 time=51.428 ms
2001:679:1024:1::12 icmp6_seq=5 ttl=64 time=45.099 ms

```


### Проверка отказоустойчивости

```
interface Ethernet2
   shutdown
```

Отключаем интерфейс на leaf-02 в сторону access-01

Проверяем доступность

```
access-01#show lacp aggregates
Port Channel Port-Channel1:
 Aggregate ID: [(8000,50-00-00-af-d3-f6,0001,0000,0000),(8000,de-ad-de-ad-00-10,0001,0000,0000)]
  Bundled Ports: Ethernet8
```

```
ping 2001:679:1024:1::12

2001:679:1024:1::12 icmp6_seq=1 ttl=64 time=48.727 ms
2001:679:1024:1::12 icmp6_seq=2 ttl=64 time=46.188 ms
2001:679:1024:1::12 icmp6_seq=3 ttl=64 time=49.962 ms
2001:679:1024:1::12 icmp6_seq=4 ttl=64 time=46.177 ms
2001:679:1024:1::12 icmp6_seq=5 ttl=64 time=45.867 ms

```






