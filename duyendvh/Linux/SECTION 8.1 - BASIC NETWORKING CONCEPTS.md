![[Screenshot 2025-11-19 at 18.13.48.png]]
$$\underbrace{8\text{ bits}}_{\text{Octet 1}} \quad \underbrace{8\text{ bits}}_{\text{Octet 2}} \quad \underbrace{8\text{ bits}}_{\text{Octet 3}} \quad \underbrace{8\text{ bits}}_{\text{Octet 4}}$$

Each 8-bit octet can represent $2^8 = 256$ different values, ranging from **0** (binary `00000000`) to **255** (binary `11111111`).

![[Screenshot 2025-11-19 at 18.18.24.png]]
![[Pasted image 20251119182325.png]]
each group of 16 bits comprise of 4 hexadecimal digits (1 hexadecimal digit can be represented by using 4 bit (0-16 in decimal))
![[Screenshot 2025-11-19 at 18.25.37.png]]
![[Screenshot 2025-11-19 at 18.26.06.png]]
![[Screenshot 2025-11-19 at 18.35.48.png]]
- The mask consists of a sequence of **binary $1 \text{s}$** followed by a sequence of **binary $0 \text{s}$**.
    
- The **$1 \text{s}$** cover the **Network ID** portion of the IP address.
    
- The **$0 \text{s}$** cover the **Host ID** portion of the IP address.
    

|**Component**|**Binary Value**|**Meaning**|
|---|---|---|
|**Network ID**|All $1 \text{s}$|This part of the address must be **identical** for all local devices.|
|**Host ID**|All $0 \text{s}$|This part of the address must be **unique** for each device.|

Let's look at a common scenario:

|**Type**|**Dotted-Decimal**|**Binary (32 bits)**|**Role**|
|---|---|---|---|
|**IP Address**|`192.168.1.10`|`11000000.10101000.00000001.00001010`|Identifies the device.|
|**Subnet Mask**|**`255.255.255.0`**|**`11111111.11111111.11111111.00000000`**|**Masks (hides) the host part.**|

In this example, the 6$1 \text{s}$ in the mask cover the first three octets (`192.168.1`).7 When a router applies this mask to the IP address, it calculates that the **Network ID** is `192.168.1.0`.
![[Screenshot 2025-11-19 at 18.27.43.png]]
![[Screenshot 2025-11-19 at 18.37.40.png]]
![[Screenshot 2025-11-19 at 18.39.41.png]]
![[Screenshot 2025-11-19 at 18.41.02.png]]
![[Screenshot 2025-11-19 at 18.45.05.png]]
