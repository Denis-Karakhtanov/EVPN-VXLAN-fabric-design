# Построение Overlay l2 сети Arista (iBGP ipv6)

## Топология 

![image](https://github.com/user-attachments/assets/3dcaa6ba-33bc-4b90-aa96-46df000c646c)


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
   vlan 10
      rd 1.1.2.1:10010
      route-target both 10:10010
      redistribute learned
   !
   address-family evpn
      neighbor fd00:c1::101 activate
      neighbor fd00:c1::102 activate
!
```

В конфигурации необходимо включить опцию для корректной работы EVPN - service routing protocols model multi-agen

### Проверка vxlan интерсейс и vlan-base mac-vrf

```
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan encapsulation ipv6
   vxlan vlan 10 vni 10010
```

При ipv6 адрессации необходимо явно указать метод инкапсуляции vxlan encapsulation ipv6

Проверяем корректно конфигурации

```
show vxlan config-sanity
```

Добавляем mac-vrf и vlan

```
 vlan 10
      rd 1.1.2.1:10010
      route-target both 10:10010
      redistribute learned
```

### Проверка доступности хостов между leaf-01 и leaf-02

```
vpc-1> ping  2001:679:1024:1::12

2001:679:1024:1::12 icmp6_seq=1 ttl=64 time=126.798 ms
2001:679:1024:1::12 icmp6_seq=2 ttl=64 time=41.973 ms
2001:679:1024:1::12 icmp6_seq=3 ttl=64 time=51.625 ms
2001:679:1024:1::12 icmp6_seq=4 ttl=64 time=173.735 ms
2001:679:1024:1::12 icmp6_seq=5 ttl=64 time=35.594 ms
```

```
vpc-2> ping  2001:679:1024:1::11

2001:679:1024:1::11 icmp6_seq=1 ttl=64 time=56.761 ms
2001:679:1024:1::11 icmp6_seq=2 ttl=64 time=48.772 ms
2001:679:1024:1::11 icmp6_seq=3 ttl=64 time=38.793 ms
2001:679:1024:1::11 icmp6_seq=4 ttl=64 time=55.854 ms
2001:679:1024:1::11 icmp6_seq=5 ttl=64 time=40.833 ms
```






