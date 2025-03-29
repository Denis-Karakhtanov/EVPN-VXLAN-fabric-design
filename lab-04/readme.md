# Построение Underlay сети(iBGP ipv6)

## Топология 

![image](https://github.com/user-attachments/assets/b1058244-28d8-4e2f-9e28-5a3083a723e5)

### Создаем процесс BGP на устройствах 

#### Spines

```
router bgp 4200000000
   router-id 1.1.1.1
   no bgp default ipv4-unicast
   bgp cluster-id 1.1.1.1
   maximum-paths 4 ecmp 4
   neighbor fd00:c1:1:: remote-as 4200000000
   neighbor fd00:c1:1:: next-hop-self
   neighbor fd00:c1:1:: route-reflector-client
   neighbor fd00:c1:2:: remote-as 4200000000
   neighbor fd00:c1:2:: next-hop-self
   neighbor fd00:c1:2:: route-reflector-client
   neighbor fd00:c1:3:: remote-as 4200000000
   !
   address-family ipv6
      neighbor fd00:c1:1:: activate
      neighbor fd00:c1:2:: activate
      neighbor fd00:c1:3:: activate
      network fd00:c1::101/128
```

В конфигурации spine используем функцию route reflection c подменой next-hop для установления связности.

#### Leafs

```
router bgp 4200000000
   router-id 1.1.2.1
   no bgp default ipv4-unicast
   maximum-paths 4 ecmp 4
   neighbor fd00:c1:1::1 remote-as 4200000000
   neighbor fd00:c1:1::3 remote-as 4200000000
   !
   address-family ipv6
      neighbor fd00:c1:1::1 activate
      neighbor fd00:c1:1::3 activate
      network fd00:c1::201/128
```

В конфигурации leaf используем обычный пиринг с включенным ECMP


### Проверка связности loopback и ecmp

```
 B I      fd00:c1::202/128 [200/0]
           via fd00:c1:1::1, Ethernet7
           via fd00:c1:1::3, Ethernet8
```

Убеждаемся что соседний leaf доступен по нескольким путям через spines

```
#ping fd00:c1::202 source fd00:c1::201
PING fd00:c1::202(fd00:c1::202) from fd00:c1::201 : 52 data bytes
60 bytes from fd00:c1::202: icmp_seq=1 ttl=63 time=31.0 ms
60 bytes from fd00:c1::202: icmp_seq=2 ttl=63 time=23.0 ms
60 bytes from fd00:c1::202: icmp_seq=3 ttl=63 time=27.8 ms
60 bytes from fd00:c1::202: icmp_seq=4 ttl=63 time=23.3 ms
60 bytes from fd00:c1::202: icmp_seq=5 ttl=63 time=21.0 ms
```

Проверяем доступность loopback другого лифа. 

