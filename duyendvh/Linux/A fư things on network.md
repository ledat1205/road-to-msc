## The Connection Process 

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