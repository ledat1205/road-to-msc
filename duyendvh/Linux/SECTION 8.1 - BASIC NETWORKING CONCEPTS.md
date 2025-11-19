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
![[Screenshot 2025-11-19 at 18.45.46.png]]
![[Screenshot 2025-11-19 at 18.46.15.png]]
![[Screenshot 2025-11-19 at 18.47.00.png]]
![[Screenshot 2025-11-19 at 18.59.47.png]]
![[Screenshot 2025-11-19 at 19.01.30.png]]
![[Screenshot 2025-11-19 at 19.02.46.png]]
![[Screenshot 2025-11-19 at 19.03.10.png]]
![[Screenshot 2025-11-19 at 19.04.02.png]]
![[Screenshot 2025-11-19 at 19.04.22.png]]
![[Screenshot 2025-11-19 at 19.05.08.png]]
![[Screenshot 2025-11-19 at 19.06.12.png]]
![[Screenshot 2025-11-19 at 19.07.17.png]]
![[Screenshot 2025-11-19 at 19.08.07.png]]
![[Screenshot 2025-11-19 at 19.14.49.png]]

| **Component**                                       | **Value**       | **Explanation**                                                                                                                                                                  |
| --------------------------------------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`eth0`**                                          |                 | This is the name of the network interface. **`eth`** typically stands for Ethernet, and `0` indicates it is the first such interface.                                            |
| **`flags=4163<UP, BROADCAST, RUNNING, MULTICAST>`** |                 | These flags indicate the current state and capabilities of the interface:                                                                                                        |
|                                                     | **`UP`**        | The interface is ready to accept and send data.                                                                                                                                  |
|                                                     | **`BROADCAST`** | The interface supports sending broadcast packets (to all devices on the local network).                                                                                          |
|                                                     | **`RUNNING`**   | The network cable is connected, and the interface is active.                                                                                                                     |
|                                                     | **`MULTICAST`** | The interface supports multicast traffic (sending data to a defined group of recipients).                                                                                        |
| **`mtu 1500`**                                      |                 | **Maximum Transmission Unit**. This is the largest packet size (in bytes) that the interface can send without fragmentation. **1500** is the standard MTU for Ethernet networks. |

| **Component**                 | **Value**       | **Explanation**                                                                                                                                                                                                                                                                          |
| ----------------------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`inet 192.168.1.100`**      | `192.168.1.100` | This is the **IPv4 address** assigned to the `eth0` interface. This address is used to identify this specific device on its network.                                                                                                                                                     |
| **`netmask 255.255.255.0`**   | `255.255.255.0` | This is the **subnet mask**. It defines the size of the network. A mask of `255.255.255.0` means the first three octets (`192.168.1`) identify the **network**, and the last octet (`100`) identifies the **host** (the specific device). (In CIDR notation, this is a **/24** network). |
| **`broadcast 192.168.1.255`** | `192.168.1.255` | This is the **broadcast address** for the local network. Any packet sent to this address will be received by every device within the `192.168.1.0/24` subnet.                                                                                                                            |

Having more than one network interface card (NIC) or virtual interface is common practice for several critical reasons:

---
## Reasons for Multiple Network Interfaces

### 1. Network Segmentation (Security and Control)

Devices often need to be connected to different, isolated networks for security or administrative purposes.

- **Routers and Firewalls:** These devices are defined by having multiple interfaces. For example, a home router has one interface facing the **Internet** (WAN) and a separate interface facing the **Local Network** (LAN). A server might have one interface on a secure **Management Network** and another on the public **Application Network**.
    
- **Separating Traffic:** This prevents certain types of traffic (like sensitive database backups) from being mixed with general user traffic.
    

### 2. High Availability and Failover

Multiple interfaces are used to provide redundancy, ensuring continuous operation even if one connection fails:

- **NIC Teaming/Bonding:** Two physical interfaces can be logically grouped ("bonded") together. If one connection or port fails, the operating system immediately switches all traffic to the other interface without interruption. This provides **fault tolerance**.
    

### 3. Increased Throughput (Load Balancing)

Multiple interfaces can be used simultaneously to increase the total available bandwidth:

- **Load Balancing:** By bonding two 1 Gbps interfaces, the server can theoretically handle up to 2 Gbps of network traffic, balancing the load across both physical connections.
    

### 4. Virtualization and Containerization

In modern computing, a single physical interface hosts many virtual interfaces:

- **Virtual Machines (VMs):** The host machine's single physical NIC is shared, but each VM has its own **virtual network interface** (VIF) and its own unique MAC and IP address.
    
- **Containers (e.g., Docker, Kubernetes):** Containers are assigned a virtual interface that connects to a **virtual bridge** on the host, allowing them to communicate. This is how you see the interface names like `eth0` and also many virtual ones like `veth...` or `docker0` on a single machine.
    

### Example

The server output you saw earlier: `eth0: flags=4163<UP, BROADCAST, RUNNING, MULTICAST> mtu 1500 inet 192.168.1.100`

If that server had a second physical card or a virtualized connection, you would see a second entry, such as: `eth1: flags=4163<UP, BROADCAST, RUNNING, MULTICAST> mtu 1500 inet 10.1.1.50`
![[Screenshot 2025-11-19 at 19.17.25.png]]

![[Screenshot 2025-11-19 at 19.17.09.png]]