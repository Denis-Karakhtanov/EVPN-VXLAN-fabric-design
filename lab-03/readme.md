# Построение Underlay сети Arista (isis)

## Топология 

![image](https://github.com/user-attachments/assets/b1058244-28d8-4e2f-9e28-5a3083a723e5)

### Создаем процесс isis на устройствах 

Выбор net

|Устройство | isis net |
| ------------| --------------| 
| Leaf-01 | 49.0001.0001.0002.0001.00 |
| Leaf-02 | 49.0001.0001.0002.0002.00 |
| Leaf-03 | 49.0001.0001.0002.0003.00 |
| Spine-01 |  49.0001.0001.0001.0001.00 |
| Spine-02 | 49.0001.0001.0001.0002.00 |

Добавляем маршрутизацию

```
router isis 100
   net 49.0001.0001.000X.000X.00
   is-type level-2
   !
   address-family ipv6 unicast
```

Включаем point-to-point линки + добавляем passive loopback

```
interface EthernetX
  isis enable 100
  isis network point-to-point

interface Loopback0
   isis enable 100
   isis passive
```


### Проверка связности loopback и ecmp

```
 I L2     fd00:c1::201/128 [115/30]
           via fe80::5200:ff:fed5:5dc0, Ethernet7
           via fe80::5200:ff:fecb:38c2, Ethernet8
```

Убеждаемся что соседний leaf доступен по нескольким путям через spines

```
#ping  fd00:c1::201
PING fd00:c1::201(fd00:c1::201) 52 data bytes
60 bytes from fd00:c1::201: icmp_seq=1 ttl=63 time=82.9 ms
60 bytes from fd00:c1::201: icmp_seq=2 ttl=63 time=80.3 ms
60 bytes from fd00:c1::201: icmp_seq=3 ttl=63 time=77.2 ms
60 bytes from fd00:c1::201: icmp_seq=4 ttl=63 time=71.7 ms
60 bytes from fd00:c1::201: icmp_seq=5 ttl=63 time=61.4 ms
```

Проверяем доступность loopback другого лифа. 


