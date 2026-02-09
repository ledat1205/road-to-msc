
Network Appliance Plane

3 Layers:
**Management Plane**:
- Means for how user or automation software configures and monitor the appliance 
- Tools: CLI, Terraform, SNMP
**Control Plane**: 
- Run protocols to determine how data plane should process traffic 
- Functions: BGP, OSPF, DHCP
**Data Plane**:
- Process packets
- Need to be fast
- Functions: NAT, forwarding, filtering

Linux covers the whole stack

