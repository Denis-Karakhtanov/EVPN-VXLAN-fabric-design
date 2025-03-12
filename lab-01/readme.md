Начало составление фабрики

Визуальное отображение состава фабрики 
![image](https://github.com/user-attachments/assets/1b3da798-9843-4c76-89e6-774bccc98f10)


| **Component**       | **IPv6 Subnet**           | **Example**                 |
|---------------------|-------------------------|-----------------------------|
| Loopbacks (VTEPs&BGP)  | `fd00:c1::X/128` (spine 10X leaf 20X)                   | `fd00:c1::101/128` (spine-1) <br> `fd00:c1::201/128` (leaf-1)|
| Loopbacks (VRF-a1)  | `fd00:c1:a1::X/128`                   | `fd00:c1:a1::1/128` (leaf-1)|
| P2P Links          | `fd00:c1:X::1-2/127` (X leaf number, high - spine)                  | `fd00:c1:1::/127`- leaf-01 - `fd00:c1:1::1/127` - spine-01 <br>  `fd00:c1:1::2/127`- leaf-01 - `fd00:c1:1::3/127` - spine-02 <br> `fd00:c1:2::/127`- leaf-02 - `fd00:c1:2::1/127` - spine-01 <br>  `fd00:c1:2::2/127`- leaf-02 - `fd00:c1:2::3/127` - spine-02    |


C1 - DC number
