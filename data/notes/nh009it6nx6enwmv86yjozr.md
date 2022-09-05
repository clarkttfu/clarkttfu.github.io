


The design of DHCP is based on an earlier protocol called the Internet Bootstrap Protocol (BOOTP) [RFC0951][RFC1542].
BOOTP and DHCP are backward-compatible in the sense that BOOTP-only clients can make use of DHCP servers and DHCP clients can make use of BOOTP-only servers. 

BOOTP, and therefore DHCP as well, is carried using UDP/IP.

DHCP comprises two major parts: 
1. address management.
  - dynamic allocation: allocate (from pool) and revoke (on lease expiration)
  - automatic allocation: allocate (from pool) without revoking
  - manual allocation: fix the address (not from pool) for client 
2. delivery of configuration data.

![BOOTP/DHCP Packet](/assets/images/network.dhcp.png)
- **DHCP Option** is carried inside "Server Name" & "Boot File Name" field.
- 
