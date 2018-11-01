Nat is the meaning of network address translation. This part of the source code is relatively independent and single, so I will not analyze it here. Everyone knows the basic functions.

There are two network protocols, **upnp** and **pmp**, under **nat**.

### Upnp application scenario (pmp is a protocol similar to upnp)

If the user accesses the Internet through NAT and needs to use modules such as P2P, BC or eMule, the UPnP function will bring great convenience. UPnP can automatically map the port numbers of BC and eMule to the public network, so that users on the public network can also initiate connections to the private network side of the NAT.

The main function is to provide an interface to map the IP + port of the intranet to the IP + port of the router. This is equivalent to the internal network's IP address of the external network, so that users of the public network can directly access you. Otherwise, you need to access it through UDP holes.

### UDP protocol in p2p

Most of the environments in which users run today are all intranet environments. The ports that are monitored in the intranet environment cannot be directly accessed by other public network programs. Need to go through a hole punching process. Both parties can connect. This is called UDP hole punching.

network can not directly access the program on the intranet. Because the router does not know how to route data to this program on the intranet.

Then we first contact the program of the external network through the program of the intranet, so the router will automatically assign a port to the program on the intranet. And record a mapping 192.168.1.1:3003 -> 111.21.12.12:3003 in the router. This mapping will eventually disappear as time goes by.

After the router establishes such a mapping relationship. Other programs on the Internet can be happy to access the port 111.21.12.12:3003. Because all data sent to this port will eventually be routed to the 192.168.1.1:3003 port. This is the so-called process of hole punching.

![nat](/picture/nat-p2p.png)

**To have access to a node of the LAN network, we also need PAT**

![nat](/picture/PAT.jpg)
