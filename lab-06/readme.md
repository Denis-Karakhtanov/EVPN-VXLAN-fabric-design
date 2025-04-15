# Построение Overlay l3 сети Arista (iBGP ipv6)

## Топология 

![image](https://github.com/user-attachments/assets/3c407a2c-38d0-45d6-8672-ca934767d181)



### Создаем процесс BGP на устройствах для обмена маршрутами EVPN 

#### Spines

```
router bgp 4200100100
   router-id 1.1.1.1
   bgp cluster-id 1.1.1.1
   maximum-paths 10
   neighbor fd00:c1::201 remote-as 4200100100
   neighbor fd00:c1::201 update-source Loopback0
   neighbor fd00:c1::201 route-reflector-client
   neighbor fd00:c1::201 send-community
   neighbor fd00:c1::202 remote-as 4200100100
   neighbor fd00:c1::202 update-source Loopback0
   neighbor fd00:c1::202 route-reflector-client
   neighbor fd00:c1::202 send-community
   neighbor fd00:c1::203 remote-as 4200100100
   neighbor fd00:c1::203 update-source Loopback0
   neighbor fd00:c1::203 route-reflector-client
   neighbor fd00:c1::203 send-community
   !
   address-family evpn
      neighbor fd00:c1::201 activate
      neighbor fd00:c1::202 activate
      neighbor fd00:c1::203 activate
!
```

В конфигурации необходимо включить опцию для корректной работы EVPN - service routing protocols model multi-agen

#### Leafs

```
router bgp 4200100100
   router-id 1.1.2.1
   maximum-paths 10
   neighbor fd00:c1::101 remote-as 4200100100
   neighbor fd00:c1::101 update-source Loopback0
   neighbor fd00:c1::101 send-community extended
   neighbor fd00:c1::102 remote-as 4200100100
   neighbor fd00:c1::102 update-source Loopback0
   neighbor fd00:c1::102 send-community
   !
   address-family evpn
      neighbor fd00:c1::101 activate
      neighbor fd00:c1::102 activate
!
```

В конфигурации необходимо включить опцию для корректной работы EVPN - service routing protocols model multi-agen

### Создание vxlan интерсейс и vlan-aware mac-vrf

```
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
```

При ipv6 адрессации необходимо явно указать метод инкапсуляции vxlan encapsulation ipv6

Проверяем корректно конфигурации

```
show vxlan config-sanity
```

Добавляем mac-vrf и vlan

```
   vlan-aware-bundle int
      rd 1.1.2.1:1
      route-target both 1:1
      redistribute learned
      vlan 10,20
```

### Создание ipv6 anycast gw и необходимых VRF

```
vrf instance int

ipv6 unicast-routing vrf int

ip virtual-router mac-address 00:00:11:11:22:22

interface Vlan10
   vrf int
   ipv6 enable
   ipv6 address virtual 2001:679:1024:1::1/64
!
interface Vlan20
   vrf int
   ipv6 enable
   ipv6 address virtual 2001:679:1024:2::1/64
```


### Проверка доступности хостов между leaf-01 и leaf-02

```
vpc-1> show ipv6

NAME              : vpc-1[1]
LINK-LOCAL SCOPE  : fe80::250:79ff:fe66:6804/64
GLOBAL SCOPE      : 2001:679:1024:1::11/64
DNS               :
ROUTER LINK-LAYER : 00:00:11:11:22:22
MAC               : 00:50:79:66:68:04
LPORT             : 20000
RHOST:PORT        : 127.0.0.1:30000
MTU:              : 1500

vpc-1> ping 2001:679:1024:2::12

2001:679:1024:2::12 icmp6_seq=1 ttl=62 time=46.503 ms
2001:679:1024:2::12 icmp6_seq=2 ttl=62 time=49.116 ms
2001:679:1024:2::12 icmp6_seq=3 ttl=62 time=47.323 ms
2001:679:1024:2::12 icmp6_seq=4 ttl=62 time=50.419 ms
2001:679:1024:2::12 icmp6_seq=5 ttl=62 time=53.264 ms

```

Проверка TTL

![image](https://github.com/user-attachments/assets/33334103-2ff2-43e1-bddc-b3bba79e1c6d)


### Создание symmetric IRB

Дополнительно рассмотрим включение symmetric IRB

Добавляем в конфигурацию L3 VNI

```
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf int vni 5000 
```
```
router bgp 4200100100
   vrf int
      rd 1.1.2.1:5000
      route-target import evpn 1:5000
      route-target export evpn 1:5000
```






