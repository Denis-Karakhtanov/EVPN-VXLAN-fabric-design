# Построение Underlay сети(OSPF)

## Топология 

![image](https://github.com/user-attachments/assets/177128cc-bbc2-4bed-a8bb-c5334b1ecde1)

## Настройка ospfv3 ipv6 Arista с аутентификацией и bfd

Выполним настройку ospf.

### Включаем 
```
ipv6 unicast-routing 
```
для возможности создания инстанса OSPFv3

### Создаем сам процесс OSPF

```
ipv6 router ospf 100
   router-id 1.1.1.1  
   passive-interface default
   no passive-interface Ethernet1
```

При full ipv6 сети без ipv4 необходимо задать вручную в формате ipv4 1.1.1.* для spine/1.1.2.* для leaf

### Добавляем настройки интерфейсов

```
interface Ethernet1
   ipv6 ospf network point-to-point
   ipv6 ospf 100 area 0.0.0.0
```

Для всех интерфейсов используется одна зона c point-to-point соединениями 

Включаем BFD на интерфейсе для быстрого определения проблем доступности при включенном линке

```
ipv6 ospf bfd
```

Включаем аутентификацию для консистентности для избежания ошибок неправильных подключений/конфигурации

```
ipv6 ospf authentication ipsec spi 3456 md5 passphrase "leaf1-spine1"
```

На каждое соединение используем уникальную фразу обозначающую устройства.

### Выборочная проверка доступности и соседства

```
Neighbor 1.1.1.1 VRF default priority is 0, state is Full
  In area 0.0.0.0 interface Ethernet7
  DR is None BDR is None
  Options is E R V6
  Dead timer is due in 33 seconds
  Graceful-restart-helper mode is Inactive
  Graceful-restart attempts: 0

 O3       fd00:c1::101/128 [110/20]
           via fe80::5200:ff:fed5:5dc0, Ethernet7

PING fd00:c1::101(fd00:c1::101) 52 data bytes
60 bytes from fd00:c1::101: icmp_seq=1 ttl=64 time=115 ms
60 bytes from fd00:c1::101: icmp_seq=2 ttl=64 time=114 ms
60 bytes from fd00:c1::101: icmp_seq=3 ttl=64 time=104 ms
60 bytes from fd00:c1::101: icmp_seq=4 ttl=64 time=97.3 ms
60 bytes from fd00:c1::101: icmp_seq=5 ttl=64 time=92.0 ms
```
