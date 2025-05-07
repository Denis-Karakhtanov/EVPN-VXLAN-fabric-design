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
set protocols bgp group evpn neighbor 172.X.X.X description leafX
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

##### EVPN MAC-VRF Routing Instance

Все взаимодействие EVPN будет использовано с использованием отдельной mac-vrf vlan aware routing instance.


Leaf-X

```
set routing-instances 1 instance-type mac-vrf
set routing-instances 1 protocols evpn encapsulation vxlan
set routing-instances 1 protocols evpn default-gateway do-not-advertise
set routing-instances 1 protocols evpn extended-vni-list all
set routing-instances 1 vtep-source-interface lo0.0
set routing-instances 1 service-type vlan-aware
set routing-instances 1 interface xe-0
set routing-instances 1 interface xe-1
set routing-instances 1 route-distinguisher 172.X.X.X:1
set routing-instances 1 vrf-target target:64510:1
set routing-instances 1 vlans vlan100 description "test"
set routing-instances 1 vlans vlan100 vlan-id 100
set routing-instances 1 vlans vlan100 vxlan vni 10100
```

##### EVPN-VXLAN Edge-Routed Bridging Fabric

Основной метод взаимодействия внутри DC это assymetric IRB with edge routing

```
set interfaces irb unit 100 virtual-gateway-accept-data
set interfaces irb unit 100 family inet address 10.100.1.1/24 preferred
set interfaces irb unit 100 family inet address 10.100.1.1/24 virtual-gateway-address 10.100.1.3
set interfaces irb unit 100 virtual-gateway-v4-mac 00:00:5e:00:00:60
```

Конфигурация для всех leaf.

##### EVPN-VXLAN Central-Routed Bridging Fabric

При необходимости использования ACL на определенных сетях, будет использоваться CRB с шлюзом на border-leaf, которые имеют больше возможностей фильтрации.

Конфигурация для всех leaf, добавление vlan в mac-vrf без шлюза.

```
set routing-instances 1 vlans vlan200 description "test-crb"
set routing-instances 1 vlans vlan200 vlan-id 200
set routing-instances 1 vlans vlan200 vxlan vni 10200
```

Конфигурация для border-leaf, создание mac-vrf и шлюза сети

```
set routing-instances 1 instance-type mac-vrf
set routing-instances 1 protocols evpn encapsulation vxlan
set routing-instances 1 protocols evpn default-gateway no-gateway-community
set routing-instances 1 protocols evpn extended-vni-list all
set routing-instances 1 vtep-source-interface lo0.0
set routing-instances 1 bridge-domains vlan200 vlan-id 200
set routing-instances 1 bridge-domains vlan200 routing-interface irb.200
set routing-instances 1 bridge-domains vlan200 vxlan vni 10200

set interfaces irb unit 200 proxy-macip-advertisement
set interfaces irb unit 200 virtual-gateway-accept-data
set interfaces irb unit 200 family inet address 10.200.1.1/24 preferred
set interfaces irb unit 200 family inet address 10.200.1.1/24 virtual-gateway-address 10.200.1.3
```

##### EVPN Multihoming

Для поддержки legacy access and management коммутаторов используются технологии EVPN Multihoming

![image](https://github.com/user-attachments/assets/a98e3544-c779-4547-a093-1e9509e43114)


Конфигурация для leaf

```
set interfaces ae1 description "legacy"
set interfaces ae1 esi 00:11:11:11:11:11:11:11:20:00
set interfaces ae1 esi all-active
set interfaces ae1 aggregated-ether-options lacp active
set interfaces ae1 aggregated-ether-options lacp periodic fast
set interfaces ae1 aggregated-ether-options lacp system-id 00:40:00:00:00:11
set interfaces ae1 unit 0 family ethernet-switching interface-mode trunk
set interfaces ae1 unit 0 family ethernet-switching vlan members vlan100

```

##### Equal-cost multi-path routing

Для обеспечения равномерного распределения трафика в underlay и overlay везде необходимо использовать опцию multipath

##### Подключение внешних соединений DC и фильтрация трафика

Подключение внешних сегментов осуществляется через border leaf и стандартный BGP unicast routing

![image](https://github.com/user-attachments/assets/9157877d-0b72-4c14-930f-34e46ec46373)



### Рассмотрение архитектуры взаимодействия ЦОД в проекте

Общая структурная схема 

![image](https://github.com/user-attachments/assets/6525758e-2923-41bd-a55b-7899b5e014ca)


Структура транспортной сети MPLS с использованием отдельных RR для обмена маршрутов l3vpn 

![image](https://github.com/user-attachments/assets/25248e13-0ed3-41c2-8ae0-3146e52c5e71)

Border leaf каждого DC имеет связь с локальным RR, а так же с RR других ЦОД для обеспечения отказоустойчивости обмена маршрутами. 

Общий underlay транспортной сети использует протокол маршрутизации isis.

Взаимодействия между evpn/vxlan и l3vpn/mpls осуществляется с помощью VRF route leaking на border leaf.

Пример конфигурации

```
set routing-instances evpn vrf-import evpn-vrf-import
set routing-instances evpn vrf-export evpn-vrf-export
set routing-instances evpn vrf-table-label

set routing-instances l3vpn vrf-import l3vpn-vrf-import
set routing-instances l3vpn vrf-export l3vpn-vrf-export
set routing-instances l3vpn vrf-table-label

set policy-options policy-statement l3vpn-vrf-import term inside from community rt:l3vpn
set policy-options policy-statement l3vpn-vrf-import term inside then accept
set policy-options policy-statement l3vpn-vrf-import term inside-evpn from community rt:evpn
set policy-options policy-statement l3vpn-vrf-import term inside-evpn then accept


set policy-options policy-statement evpn-vrf-import term inside-evpn from community rt:evpn
set policy-options policy-statement evpn-vrf-import term inside-evpn then accept
set policy-options policy-statement evpn-vrf-import term inside from community rt:l3vpn
set policy-options policy-statement evpn-vrf-import term inside then accept

```

Необходимо учитывать, что внутри одного DC обмен маршрутами между border-leaf по evpn/l3pn должен быть заблокирован для предотвращения петель маршрутов

```
set policy-options policy-statement inside-vrf-import term loop from community border-leaf-X
set policy-options policy-statement inside-vrf-import term loop then reject

set policy-options policy-statement inside-evpn-vrf-import term loop from community border-leaf-X
set policy-options policy-statement inside-evpn-vrf-import term loop then reject
```







