The translation is from ( [https://github.com/ethereum/go-ethereum/pull/1889](https://github.com/ethereum/go-ethereum/pull/1889) )

This PR aggregates a lot of small modifications to core, trie, eth and other packages to collectively implement the eth/63 fast synchronization algorithm. In short, geth --fast.

## Algorithm

The goal of the the fast sync algorithm is to exchange processing power for bandwidth usage. Instead of processing the entire block-chain one link at a time, and replay all transactions that ever happened in history, fast syncing downloads the transaction receipts along the blocks, and pulls an entire recent state database. This allows a fast synced node to still retain its status an an archive node containing all historical data for user queries (and thus not influence the network's health in general), but at the same time to reassemble a recent network state at a fraction of the time it would take full block processing.

An outline of the fast sync algorithm would be:

- Similarly to classical sync, download the block headers and bodies that make up the blockchain
- Similarly to classical sync, verify the header chain's consistency (POW, total difficulty, etc)
- Instead of processing the blocks, download the transaction receipts as defined by the header
- Store the downloaded blockchain, along with the receipt chain, enabling all historical queries
- When the chain reaches a recent enough state (head - 1024 blocks), pause for state sync:
	- Retrieve the entire Merkel Patricia state trie defined by the root hash of the pivot point
	- For every account found in the trie, retrieve it's contract code and internal storage state trie
- Upon successful trie download, mark the pivot point (head - 1024 blocks) as the current head
- Import all remaining blocks (1024) by fully processing them as in the classical sync

## Analysis
By downloading and verifying the entire header chain, we can guarantee with all the security of the classical sync, that the hashes (receipts, state tries, etc) contained within the headers are valid. Based on those hashes, we can confidently download transaction receipts and the entire state trie afterwards. Additionally, by placing the pivoting point (where fast sync switches to block processing) a bit below the current head (1024 blocks), we can ensure that even larger chain reorganizations can be handled without the need of a new sync (as we have all the state going that many blocks back).

## Caveats
The historical block-processing based synchronization mechanism has two (approximately similarly costing) bottlenecks: transaction processing and PoW verification. The baseline fast sync algorithm successfully circumvents the transaction processing, skipping the need to iterate over every single state the system ever was in. However, verifying the proof of work associated with each header is still a notably CPU intensive operation.

However, we can notice an interesting phenomenon during header verification. With a negligible probability of error, we can still guarantee the validity of the chain, only by verifying every K-th header, instead of each and every one. By selecting a single header at random out of every K headers to verify, we guarantee the validity of an N-length chain with the probability of (1/K)^(N/K) (i.e. we have 1/K chance to spot a forgery in K blocks, a verification that's repeated N/K times).

Let's define the negligible probability Pn as the probability of obtaining a 256 bit SHA3 collision (i.e. the hash Ethereum is built upon): 1/2^128. To honor the Ethereum security requirements, we need to choose the minimum chain length N (below which we veriy every header) and maximum K verification batch size such as (1/K)^(N/K) <= Pn holds. Calculating this for various {N, K} pairs is pretty straighforward, a simple and lenient solution being http://play.golang.org/p/B-8sX_6Dq0.


| N    | K   | N    | K   | N    | K   | N    | K   |
| ---- | --- | ---- | --- | ---- | --- | ---- | --- |
| 1024 | 43  | 1792 | 91  | 2560 | 143 | 3328 | 198 |
| 1152 | 51  | 1920 | 99  | 2688 | 152 | 3456 | 207 |
| 1280 | 58  | 2048 | 108 | 2816 | 161 | 3584 | 217 |
| 1408 | 66  | 2176 | 116 | 2944 | 170 | 3712 | 226 |
| 1536 | 74  | 2304 | 128 | 3072 | 179 | 3840 | 236 |
| 1664 | 82  | 2432 | 134 | 3200 | 189 | 3968 | 246 |


The above table should be interpreted in such a way, that if we verify every K-th header, after N headers the probability of a forgery is smaller than the probability of an attacker producing a SHA3 collision. It also means, that if a forgery is indeed detected, the last N headers should be discarded as not safe enough. Any {N, K} pair may be chosen from the above table, and to keep the numbers reasonably looking, we chose N=2048, K=100. This will be fine tuned later after being able to observe network bandwidth/latency effects and possibly behavior on more CPU limited devices.

Using this caveat however would mean, that the pivot point can be considered secure only after N headers have been imported after the pivot itself. To prove the pivot safe faster, we stop the "gapped verificatios" X headers before the pivot point, and verify every single header onward, including an additioanl X headers post-pivot before accepting the pivot's state. Given the above N and K numbers, we chose X=24 as a safe number.

With this caveat calculated, the fast sync should be modified so that up to the pivoting point - X, only every K=100-th header should be verified (at random), after which all headers up to pivot point + X should be fully verified before starting state database downloading. Note: if a sync fails due to header verification the last N headers must be discarded as they cannot be trusted enough.


## Weakness
Blockchain protocols in general (i.e. Bitcoin, Ethereum, and the others) are susceptible to Sybil attacks, where an attacker tries to completely isolate a node from the rest of the network, making it believe a false truth as to what the state of the real network is. This permits the attacker to spend certain funds in both the real network and this "fake bubble". However, the attacker can only maintain this state as long as it's feeding new valid blocks it itself is forging; and to successfully shadow the real network, it needs to do this with a chain height and difficulty close to the real network. In short, to pull off a successful Sybil attack, the attacker needs to match the network's hash rate, so it's a very expensive attack.

Compared to the classical Sybil attack, fast sync provides such an attacker with an extra ability, that of feeding a node a view of the network that's not only different from the real network, but also that might go around the EVM mechanics. The Ethereum protocol only validates state root hashes by processing all the transactions against the previous state root. But by skipping the transaction processing, we cannot prove that the state root contained within the fast sync pivot point is valid or not, so as long as an attacker can maintain a fake blockchain that's on par with the real network, it could create an invalid view of the network's state.

To avoid opening up nodes to this extra attacker ability, fast sync (beside being solely opt-in) will only ever run during an initial sync (i.e. when the node's own blockchain is empty). After a node managed to successfully sync with the network, fast sync is forever disabled. This way anybody can quickly catch up with the network, but after the node caught up, the extra attack vector is plugged in. This feature permits users to safely use the fast sync flag (--fast), without having to worry about potential state root attacks happening to them in the future. As an additional safety feature, if a fast sync fails close to or after the random pivot point, fast sync is disabled as a safety precaution and the node reverts to full, block-processing based synchronization.

## Performance
To benchmark the performance of the new algorithm, four separate tests were run: full syncing from scrath on Frontier and Olympic, using both the classical sync as well as the new sync mechanism. In all scenarios there were two nodes running on a single machine: a seed node featuring a fully synced database, and a leech node with only the genesis block pulling the data. In all test scenarios the seed node had a fast-synced database (smaller, less disk contention) and both nodes were given 1GB database cache (--cache=1024).


The machine running the tests was a Zenbook Pro, Core i7 4720HQ, 12GB RAM, 256GB m.2 SSD, Ubuntu 15.04.


| Dataset (blocks, states)              | Normal sync (time, db) | Fast sync (time, db) |
| ------------------------------------- | :--------------------: | -------------------: |
| Frontier, 357677 blocks, 42.4K states | 12:21 mins, 1.6 GB     | 2:49 mins, 235.2 MB  |
| Olympic, 837869 blocks, 10.2M states  | 4:07:55 hours, 21 GB   | 31:32 mins, 3.8 GB   |


The resulting databases contain the entire blockchain (all blocks, all uncles, all transactions), every transaction receipt and generated logs, and the entire state trie of the head 1024 blocks. This allows a fast synced node to act as a full archive node from all intents and purposes.



## Closing remarks
The fast sync algorithm requires the functionality defined by eth/63. Because of this, testing in the live network requires for at least a handful of discoverable peers to update their nodes to eth/63. On the same note, verifying that the implementation is truly correct will also entail waiting for the wider deployment of eth/63.