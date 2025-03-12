Начало составление фабрики

Визуальное отображение состава фабрики 
![image](https://github.com/user-attachments/assets/1b3da798-9843-4c76-89e6-774bccc98f10)


| **Component**       | **IPv6 Subnet**           | **Example**                 |
|---------------------|-------------------------|-----------------------------|
| Loopbacks (VTEPs)  | `fd00:c1::X/128' spine 10X leaf 20X                   | `2001:db8::101/128` (spine-1) <br> `2001:db8::201/128` (leaf-1)|
| Loopbacks (VRF)  | `/128`                   | `2001:db8::101/128` (spine-1) <br> `2001:db8::201/128` (leaf-1)|
| P2P Links          | `/127`                   | `2001:db8:1::/127`          |

