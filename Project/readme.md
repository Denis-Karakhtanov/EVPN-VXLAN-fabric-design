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
- BGP IPv4 unicast


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

Одинаковая конфигурация на leaf и spine


##### Control plane - bgp evpn, data plane - vxlan

Для создания overlay сети и обмена маршрутной информацией используется iBGP с функией RR на spine с функцией BFD

Spine-X

```
set protocols bgp group evpn type internal
set protocols bgp group evpn description "evpn/vxlan"
set protocols bgp group evpn local-address 172.X.X.X
set protocols bgp group evpn family evpn signaling
set protocols bgp group evpn family route-target
set protocols bgp group evpn cluster 172.X.X.X
set protocols bgp group evpn multipath
set protocols bgp group evpn bfd-liveness-detection minimum-interval 1000
set protocols bgp group evpn bfd-liveness-detection multiplier 3
set protocols bgp group evpn bfd-liveness-detection session-mode automatic
set protocols bgp group evpn neighbor 172.X.X.X description leaf1
set protocols bgp group evpn neighbor 172.X.X.X description leaf2
set protocols bgp group evpn neighbor 172.X.X.X description leaf3
```

Leaf-X

```
set protocols bgp group evpn type internal
set protocols bgp group evpn description "evpn/vxlan"
set protocols bgp group evpn local-address 172.X.X.X
set protocols bgp group evpn family evpn signaling
set protocols bgp group evpn family route-target
set protocols bgp group evpn multipath
set protocols bgp group evpn bfd-liveness-detection minimum-interval 1000
set protocols bgp group evpn bfd-liveness-detection multiplier 3
set protocols bgp group evpn bfd-liveness-detection session-mode automatic
set protocols bgp group evpn neighbor 172.X.X.X description spine1
set protocols bgp group evpn neighbor 172.X.X.X description spineX
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






