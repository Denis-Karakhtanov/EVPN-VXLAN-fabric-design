# Построение Underlay сети(OSPF)

## Топология 

![image](https://github.com/user-attachments/assets/177128cc-bbc2-4bed-a8bb-c5334b1ecde1)

## Настройка ospf 

Выполним настройку ospf.

# Включаем 
```
ipv6 unicast-routing 
```
для возможности создания инстанса OSPFv3

# Создаем сам процесс OSPF

```
ipv6 router ospf 100
   router-id 1.1.1.1  
   passive-interface default
   no passive-interface Ethernet1
```

При full ipv6 сети без ipv4 необходимо задать вручную в формате ipv4 1.1.1.* для spine/1.1.2.* для leaf
