# Построение выхода в интернет из фабрики и связи двух VRF через внешниее устройство (iBGP ipv6)

## Топология 

![image](https://github.com/user-attachments/assets/db373b73-f9a6-4f01-b6da-3e1ee219d08d)


| **VRF**       | **IPv6 Subnet**           | 
|---------------------|-------------------------|
| int  | `2001:679:1024:1:/64` <br> `2001:679:1024:2:/64`                  | 
| int2  | `2001:679:1024:A01:/64`                  | 


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

### Создание vxlan интерсейс, vlan-aware mac-vrf и l3 vrf

```
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vlan 30 vni 10030
   vxlan vrf int vni 5000
   vxlan vrf int2 vni 5002
```

При ipv6 адрессации необходимо явно указать метод инкапсуляции vxlan encapsulation ipv6

Проверяем корректно конфигурации

```
show vxlan config-sanity
```

Добавляем mac-vrf и vrf

```
 vlan-aware-bundle int
      rd 1.1.2.2:1
      route-target both 1:1
      redistribute learned
      vlan 10,20,30

ip routing
ip routing vrf int
ip routing vrf int2
!
ipv6 unicast-routing
ipv6 unicast-routing vrf int
ipv6 unicast-routing vrf int2

   vrf int
      rd 1.1.2.2:5000
      route-target import evpn 1:5000
      route-target export evpn 1:5000
   !
   vrf int2
      rd 1.1.2.2:5002
      route-target import evpn 1:5002
      route-target export evpn 1:5002

```

### Создание ipv6 anycast gw в необходимых VRF

```
ip virtual-router mac-address 00:00:11:11:22:22

interface Vlan10
   ipv6 enable
   ip address virtual 10.0.0.1/24
   ipv6 address virtual 2001:679:1024:1::1/64
!
interface Vlan20
   vrf int
   ipv6 enable
   ip address virtual 20.0.0.1/24
   ipv6 address virtual 2001:679:1024:2::1/64
!
interface Vlan30
   vrf int2
   ipv6 enable
   ip address virtual 30.0.0.1/24
   ipv6 address virtual 2001:679:1024:a01::1/64

```

### Выполняем редистрибьюцию маршрутов в сторону fw-1 внутрь VRF int и int2 через leaf-03

```
  vrf int
      rd 1.1.2.3:5000
      route-target import evpn 1:5000
      route-target export evpn 1:5000
      redistribute static
   !
   vrf int2
      rd 1.1.2.3:5002
      route-target import evpn 1:5002
      route-target export evpn 1:5002
      redistribute static

ipv6 route vrf int ::/0 fd00:c1:3::5
ipv6 route vrf int2 ::/0 fd00:c1:3::7
```

Добавляем маршруты в сети vrf int и int 2 на fw-01 для взаимодействия между ними

```
ipv6 route 2001:679:1024::/56 fd00:c1:3::4
ipv6 route 2001:679:1024:a00::/56 fd00:c1:3::6
```

Так же настраиваем Loopback 0 на fw-1 с интернет адресом, для эмуляции подключенияю в интернет

```
interface Loopback0
   ipv6 address 2001:4860:4860::8888/128
```


### Проверка доступности интернета и взаимодействия между VRF

Проверка машрутов type 5 в VRF

```
#show ipv6 route vrf int

VRF: int
Displaying 5 of 10 IPv6 routing table entries
Codes: C - connected, S - static, K - kernel, O3 - OSPFv3,
       B - Other BGP Routes, A B - BGP Aggregate, R - RIP,
       I L1 - IS-IS level 1, I L2 - IS-IS level 2, DH - DHCP,
       NG - Nexthop Group Static Route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       RC - Route Cache Route

 C        2001:679:1024:1::/64 [0/0]
           via Vlan10, directly connected
 B I      2001:679:1024:2::12/128 [200/0]
           via VTEP fd00:c1::202 VNI 5000 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 C        2001:679:1024:2::/64 [0/0]
           via Vlan20, directly connected
 B I      fd00:c1:3::4/127 [200/0]
           via VTEP fd00:c1::203 VNI 5000 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
 B I      ::/0 [200/0]
           via VTEP fd00:c1::203 VNI 5000 router-mac 50:00:00:72:8b:31 local-interface Vxlan1
```

Проверка выхода в интернет

```
vpc-1> ping 2001:4860:4860::8888

2001:4860:4860::8888 icmp6_seq=1 ttl=24 time=929.984 ms
2001:4860:4860::8888 icmp6_seq=2 ttl=62 time=45.940 ms
2001:4860:4860::8888 icmp6_seq=3 ttl=62 time=45.744 ms
2001:4860:4860::8888 icmp6_seq=4 ttl=62 time=46.474 ms
2001:4860:4860::8888 icmp6_seq=5 ttl=62 time=42.416 ms

```

Проверка связности между VRF


```
vpc-3> ping 2001:679:1024:1::11

2001:679:1024:1::11 icmp6_seq=1 ttl=55 time=140.034 ms
2001:679:1024:1::11 icmp6_seq=2 ttl=55 time=99.716 ms
2001:679:1024:1::11 icmp6_seq=3 ttl=55 time=89.474 ms
2001:679:1024:1::11 icmp6_seq=4 ttl=55 time=97.523 ms
2001:679:1024:1::11 icmp6_seq=5 ttl=55 time=89.231 ms

vpc-3> ping 2001:679:1024:2::12

2001:679:1024:2::12 icmp6_seq=1 ttl=55 time=91.284 ms
2001:679:1024:2::12 icmp6_seq=2 ttl=55 time=92.923 ms
2001:679:1024:2::12 icmp6_seq=3 ttl=55 time=82.541 ms
2001:679:1024:2::12 icmp6_seq=4 ttl=55 time=93.531 ms
2001:679:1024:2::12 icmp6_seq=5 ttl=55 time=87.778 ms
```







