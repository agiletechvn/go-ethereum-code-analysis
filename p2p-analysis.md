P2p source code and several packages below

- Discover contains the[Kademlia protocol](./references/Kademlia.pdf). It is a UDP-based p2p node discovery protocol.
- Discv5: new node discovery protocol. Still a test proposal. This analysis is not covered.
- Part of the code for nat network address translation
- Netutil: some tools
- Simulations simulation of p2p networks. This analysis is not covered.

Source code analysis of the discover package

- [Discovered nodes for persistent storage database.go](p2p-database-analysis.md)
- [The core logic of the Kademlia protocol is tabel.go](p2p-table-analysis.md)
- [UDP protocol processing logic udp.go](p2p-udp-analysis.md)
- [Network address translation nat.go](p2p-nat-analysis.md)

p2p/ package source analysis

- [Encrypted link processing protocol between nodes rlpx.go](p2p-rlpx-analysis.md)
- [The processing logic to select nodes and then connect them.](p2p-dial-analysis.md)
- [Processing of node and node connections and handling of protocols peer.go](p2p-peer-analysis.md)
- [Logical server.go for p2p server](p2p-server-analysis.md)
