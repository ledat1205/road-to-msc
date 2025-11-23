![[Screenshot 2025-11-24 at 01.26.11.png]]
![[Screenshot 2025-11-24 at 01.26.32.png]]

![[Screenshot 2025-11-24 at 01.27.38.png]]
![[Screenshot 2025-11-24 at 01.28.25.png]]
![[Screenshot 2025-11-24 at 01.29.08.png]]
![[Screenshot 2025-11-24 at 01.30.27.png]]
![[Screenshot 2025-11-24 at 01.31.27.png]]
## Why Use `-f` with Default Size?

When used with the default, small packet size, the primary reasons for including the `-f` flag are:

- **Baseline Connectivity Check:** Since the default packet (approx. 84 bytes) is far smaller than the standard Ethernet MTU of 1500 bytes, fragmentation will definitely **not** be needed. This test verifies the most fundamental connectivity, ensuring that the target is reachable via the basic ICMP protocol without any complications related to packet size or fragmentation.
    
- **Verify DF Bit Handling:** It confirms that all routers and the destination host correctly receive and process a packet that has the DF bit set. This is a subtle diagnostic check that can reveal issues with firewall rules that might improperly filter packets based on header flags.
- 