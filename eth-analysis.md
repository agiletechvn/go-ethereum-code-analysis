The source code of eth includes the following packages.

- The downloader is mainly used to synchronize with the network, including the traditional synchronization method and fast synchronization method.
- Fetcher is mainly used for block-based notification synchronization. When we receive the NewBlockHashesMsg message, we only receive a lot of block hash values. The hash value needs to be used to synchronize the block.
- Filter provides RPC-based filtering, including real-time data synchronization (PendingTx), and historical log filtering (Log filter)
- Gasprice offers price advice for gas, based on the gasprice of the last few blocks, to get the current price of gasprice

Partial source analysis of eth protocol

- [Ethereum's network protocol](eth-network-analysis.md)

Source analysis of the fetcher part

- [fetch partial source analysis](eth-fetcher-analysis.md)

Downloader partial source code analysis

- [Node fast synchronization algorithm](fast-sync-algorithm.md)
- [The schedule and result assembly used to provide the download task. queue.go](eth-downloader-queue-analysis.md)
- [Used to represent the peer, provide QoS and other functions peer.go](eth-downloader-peer-analysis.md)
- [The fast synchronization algorithm is used to provide the state-root synchronization of the Pivot point statesync.go](eth-downloader-statesync.md)
- [Analysis of the general process of synchronization](eth-downloader-analysis.md)

Filter part of the source code analysis

- [Provide Bloom filter query and RPC filtering](eth-bloombits-and-filter-analysis.md)
