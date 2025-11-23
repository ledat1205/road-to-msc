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
![[Screenshot 2025-11-24 at 00.35.50.png]]
## Common Network Interface Types and Configurations

### 1. Loopback Interface

- **Typical Name:** `lo`
    
- **Purpose:** Allows a machine to communicate with itself (**`localhost`**), essential for internal diagnostics and running local services. It has no physical hardware.
    
- **Example Output Snippet:**
    

Plaintext

```
lo: <LOOPBACK,UP,LOWER_UP> mtu 65536
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
```

| **Notation** | **Network Bits** | **Host Bits** | **Subnet Mask (Decimal)** | **Max Hosts (Usable)** | **Size**                                       |
| ------------ | ---------------- | ------------- | ------------------------- | ---------------------- | ---------------------------------------------- |
| **$/8$**     | 8                | 24            | **$255.0.0.0$**           | $16,777,214$           | **Huge** (Class A size)                        |
| **$/24$**    | 24               | 8             | **$255.255.255.0$**       | $254$                  | **Small - for private network** (Class C size) |
### 2. Wired Ethernet Interface

- **Typical Name:** `eth0` or `enpXsY`
    
- **Purpose:** Connects the machine to a **Local Area Network (LAN)** via a physical cable, using a **Network Interface Card (NIC)**.
    
- **Example Output Snippet:**
    

Plaintext

```
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    link/ether 00:11:22:33:44:55 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 brd 192.168.1.255 scope global eth0
```

---

### 3. Wireless Interface

- **Typical Name:** `wlan0` or `wlpXsY`
    
- **Purpose:** Allows communication wirelessly via the **$802.11$ (Wi-Fi)** protocol.
    
- **Key Differences:** Uses `link/ieee802.11` instead of `link/ether`.
    
- **Example Output Snippet:**
    

Plaintext

```
wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    link/ieee802.11 66:77:88:99:AA:BB brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.105/24 brd 192.168.1.255 scope global wlan0
```

### 4. Virtual Bridge Interface

- **Typical Name:** `br0` or `virbr0`
    
- **Purpose:** A **software switch** used to connect virtual devices (like VMs) to the physical network (bridging **virtual traffic** to the **physical NIC**).
    
- **Key Differences:** Shows the bridge type and links to other interfaces.
    
- **Example Output Snippet:**
    

Plaintext

```
br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    link/ether CA:FE:C0:LA:00:01 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/24 brd 10.0.0.255 scope global br0
```

### 5. Tunnel/VPN Interface

- **Typical Name:** `tun0`, `tap0`, `wg0`
    
- **Purpose:** Creates a **secure, encrypted tunnel** for Virtual Private Network (VPN) traffic.
    
- **Key Differences:** Often uses the specific link type `link/none` for a purely virtual, encapsulated connection.
    
- **Example Output Snippet (Tunnel):**
    

Plaintext

```
tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1400
    link/none 
    inet 10.8.0.2/24 scope global tun0
```

### 6. Docker/Container Interface

- **Typical Name:** `docker0` (bridge) or `veth...` (virtual ethernet pairs)
    
- **Purpose:** Provides an internal network and IP addresses for **Docker containers** to communicate with each other and the host machine.
    
- **Key Differences:** Uses a dedicated private subnet (e.g., $172.17.0.0/16$) and is often managed entirely by the container platform.
    
- **Example Output Snippet (Docker Bridge):**
    

Plaintext

```
docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    link/ether 02:42:AC:11:00:01 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
```

### 7. Alias/Sub-Interface

- **Typical Name:** `eth0:0` (in older tools) or multiple `inet` entries (in modern `ip` tools)
    
- **Purpose:** Allows a **single physical NIC** to host multiple distinct IP addresses, often for different services or migrating networks.
    
- **Key Differences:** No separate interface entry is created in modern Linux tools; instead, the physical interface has multiple IP addresses listed.
    
- **Example Output Snippet (Modern Alias):**
    

Plaintext

```
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    link/ether 00:11:22:33:44:55 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 scope global eth0
    inet 192.168.10.50/24 scope global eth0  <-- Second IP (The "Alias")
```

![[Screenshot 2025-11-24 at 00.59.24.png]]

##Why Specify `eth0`?

### 1. Unique Association

Every network interface (whether physical like **`eth0`** or virtual like **`tun0`**) represents a distinct connection point or "port" for the operating system.

- An IP address is not just a number floating in space; it is the **address of that specific connection point**.
    
- The system needs to know _which_ piece of hardware or software is responsible for sending and receiving traffic using that IP address.
    

Example:

If you had both a wired connection (eth0) and a Wi-Fi connection (wlan0), and you assigned an IP address without specifying the device:

Bash

```
# This command is ambiguous and would fail!
sudo ip addr add 192.168.1.50/24 
```

The operating system would not know if $192.168.1.50$ should be used by the wired card or the wireless card. By specifying **`dev eth0`**, you link the IP address directly to the physical NIC.

### 2. Routing Decisions

Routers and the Linux kernel rely on this interface-to-address association to make **routing decisions**.

When the system receives a packet destined for a remote network, the routing table looks at the destination and determines which **interface** to send the packet out of. This interface is associated with a **Gateway IP Address**.

- If the system wants to reach a device on the Internet, it knows to send the data out of `eth0` because that is the device configured with the default gateway.
    

### 3. Hardware Differences

Different devices require different Layer 2 (Link Layer) protocols:

- **`eth0`** uses **Ethernet** (Layer 2).
    
- **`wlan0`** uses **$802.11$ (Wi-Fi)** (Layer 2).
    
- **`tun0`** uses **none** (Layer 2 for tunneling).
    

The IP address configuration must be placed on the device that handles the correct lower-layer protocol, making the **device specification mandatory**.
![[Screenshot 2025-11-24 at 01.05.21.png]]
![[Screenshot 2025-11-24 at 01.05.47.png]]
![[Screenshot 2025-11-24 at 01.07.53.png]]
## The Connection Process (Computer-Specific)

Your computer, running an operating system like Linux, macOS, or Windows, uses the **TCP/IP stack** to handle this communication.

---

### 1. 🔍 Naming and Address Resolution (DNS & ARP)

Before your computer can send a data packet, it needs two things: the destination's **IP address** and the next step's **MAC address**.

- **DNS (If Applicable):** If you use a name (e.g., `payroll.corp.com`) instead of an IP address, your computer first sends a **DNS query** to the designated DNS server. The DNS server replies with the service's IP address (e.g., $192.168.5.100$).
    
- **Routing Decision:** Your computer's kernel checks its **routing table**.
    
    - It compares your computer's IP and subnet mask (e.g., $192.168.1.100/24$) with the destination IP ($192.168.5.100$).
        
    - Since these IPs are on **different subnets** (the 1.x network is different from the 5.x network), the computer knows it cannot talk directly to the host.
        
    - **Action:** The computer determines it must send the packet to its designated **Default Gateway (Router)**.
        
- **ARP (Address Resolution Protocol):** The computer now knows the IP address of its router. It needs the router's **MAC address** to build the Layer 2 frame.
    
    - Your computer sends an **ARP request broadcast** (to every device on its local subnet): "Who has the IP address of the router? Tell me their MAC address."
        
    - The router replies with its MAC address.
        

---

### 2. 📦 Data Encapsulation and Framing

Your computer's network stack now prepares the data for transmission (the **Encapsulation** process).

1. **Transport Layer (Layer 4):** The operating system chooses a **Source Port** (a random high number) and uses the service's well-known **Destination Port** (e.g., Port 443 for HTTPS) to create a **TCP Segment**.
    
2. **Network Layer (Layer 3):** The kernel adds the **IP Header**.
    
    - **Source IP:** Your computer's IP (e.g., $192.168.1.100$).
        
    - **Destination IP:** The service's IP (e.g., $192.168.5.100$).
        
    - The result is an **IP Packet**.
        
3. **Data Link Layer (Layer 2):** The kernel adds the **Ethernet Frame Header**.
    
    - **Source MAC:** Your computer's NIC MAC address.
        
    - **Destination MAC:** The **Router's MAC address** (found via ARP).
        
    - The result is an **Ethernet Frame**.
        

---

### 3. 🔌 Physical Transmission

The Frame is sent to the NIC (e.g., `eth0`), where it is converted into electrical signals (bits) and transmitted over the physical medium (cable or air).

---

### 4. 🔀 Network Infrastructure Role (Routers)

The data leaves your computer and is handled by the infrastructure, but this step is crucial for the communication to succeed:

1. **Router Decapsulates:** The **Router** receives the Frame (since its MAC address was the destination). It strips the Layer 2 header and reads the **Layer 3 IP Header**.
    
2. **Router Routes:** The router sees the ultimate destination IP is $192.168.5.100$. It looks up this network in its routing table and forwards the packet toward the next device (likely a core switch or another router) on the $192.168.5.x$ network.


## 🔍 Phase I: Name Resolution and Routing

Before sending the request, your computer must find the server.

### A. Resolve IP Address (DNS)

1. **Browser Action (Application Layer):** When you type `https://www.company.com`, your browser needs the server's IP address. It sends the domain name to the operating system.
    
2. **OS Action:** The OS checks its **cache** and the host file. If the IP is not found, it sends a **DNS Query** to the configured DNS server.
    
3. **Result:** The DNS server resolves `www.company.com` to its public IP address (e.g., $203.0.113.15$).
    

### B. Route to Gateway (ARP and Layer 2 Framing)

1. **Routing Decision (Network Layer):** Your OS compares the public destination IP with your local IP and determines that the server is on a different network. It must send the packet to its **Default Gateway (Router)**.
    
2. **ARP (Data Link Layer):** Your computer uses **ARP** to get the **MAC address** of the router.
    
3. **Encapsulation:** The HTTPS data (which is the payload) is encapsulated into an **IP Packet** (with the server's public IP as the destination) and then into an **Ethernet Frame** (with the **Router's MAC** as the destination). The frame is sent out your NIC (`eth0` or `wlan0`).
    

---

## 2. 🔒 Phase II: The TLS/SSL Handshake (Security Setup)

After the data reaches the server's network, the connection proceeds to establish encryption. This happens at the **Application/Presentation Layers (Layers 7/6)**.

### A. Initial Connection (TCP Handshake)

1. The client first performs a three-way **TCP handshake** (SYN, SYN-ACK, ACK) with the server on **Port 443** (the standard port for HTTPS) to establish a reliable connection channel (Transport Layer).
    

### B. TLS Handshake (Key Exchange)

![Hình ảnh về TLS Handshake Diagram](https://encrypted-tbn3.gstatic.com/licensed-image?q=tbn:ANd9GcTYBpJMseOEJpRk7Ddqt4b7N8o2ESajQwKlHVQQddXDMlerAdeuzfr5ARKR3VpIOVzld2FDuW-qMKRZdlfeG4G4H70XXgmutrRr004xNhbmWszXOlk)

Shutterstock

1. **Client Hello:** The client sends a message containing:
    
    - The highest **TLS protocol version** it supports (e.g., TLS 1.3).
        
    - A list of **Cipher Suites** (algorithms for encryption and authentication) it supports.
        
2. **Server Hello:** The server chooses the best protocol version and Cipher Suite it can support from the client's list.
    
3. **Certificate:** The server sends its **Digital Certificate** (signed by a trusted Certificate Authority). This proves its identity and contains its **Public Key**.
    
4. **Key Exchange:** The client verifies the server's certificate. The client then generates a **Symmetric Session Key** and encrypts it using the server's **Public Key** (from the certificate).
    
5. **Start Encrypting:** The server decrypts the session key using its private key. Both the client and server now have the same **symmetric key**. They declare the handshake is complete, and all subsequent communication will be encrypted using this session key.

