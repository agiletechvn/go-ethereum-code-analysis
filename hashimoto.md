Hashimoto :I/O bound proof of work

Abstract: Using a cryptographic hash function not as a proofofwork by itself, but
rather as a generator of pointers to a shared data set, allows for an I/O bound
proof of work. This method of proof of work is difficult to optimize via ASIC
design, and difficult to outsource to nodes without the full data set. The name is
based on the three operations which comprise the algorithm: hash, shift, and
modulo.

The need for proofs which are difficult to outsource and optimize

A common challenge in cryptocurrency development is maintaining decentralization of the
network. The use of proofofwork to achieve decentralized consensus has been most notably
demonstrated by Bitcoin, which uses partial collisions with zero of sha256, similar to hashcash. As
Bitcoin’s popularity has grown, dedicated hardware (currently application specific integrated circuits, or
ASICs) has been produced to rapidly iterate the hash ­based proofofwork function. Newer projects
similar to Bitcoin often use different algorithms for proofofwork, and often with the goal of ASIC
resistance. For algorithms such as Bitcoin’s, the improvement factor of ASICs means that commodity
computer hardware can no longer be effectively used, potentially limiting adoption.

Proofofwork can also be “outsourced”, or performed by a dedicated machine (a “miner”)
without knowledge ofwhat is being verified. This is often the case in Bitcoin’s “mining pools”. It is also
beneficial for a proofofwork algorithm to be difficult to outsource, in order to promote decentralization
and encourage all nodes participating in the proofofwork process to also verify transactions. With these
goals in mind, we present Hashimoto, an I/O bound proofofwork algorithm we believe to be resistant to
both ASIC design and outsourcing.

Initial attempts at "ASIC resistance" involved changing Bitcoin's sha256 algorithm for a different,
more memory intensive algorithm, Percival's "scrypt" password based key derivation function1. Many
implementations set the scrypt arguments to low memory requirements, defeating much ofthe purpose of
the key derivation algorithm. While changing to a new algorithm, coupled with the relative obscurity of the
various scrypt­based cryptocurrencies allowed for a delay, scrypt optimized ASICs are now available.
Similar attempts at variations or multiple heterogeneous hash functions can at best only delay ASIC
implementations.

Leveraging shared data sets to create I/O bound proofs

    "A supercomputer is a device for turning compute-bound problems into I/O-bound  problems."
    -Ken Batcher

Instead, an algorithm will have little room to be sped up by new hardware if it acts in a way that commodity computer systems are already optimized for.

Since I/O bounds are what decades of computing research has gone towards solving, it's unlikely that the relatively small motivation ofmining a few coins would be able to advance the state ofthe art in cache hierarchies. In the case that advances are made, they will be likely to impact the entire industry of computer hardware.

Fortuitously, all nodes participating in current implementations ofcryptocurrency have a large set of mutually agreed upon data; indeed this “blockchain” is the foundation ofthe currency. Using this large data set can both limit the advantage ofspecialized hardware, and require working nodes to have the entire data set.

Hashimoto is based offBitcoin’s proofofwork2. In Bitcoin’s case, as in Hashimoto, a successful
proofsatisfies the following inequality:

    hash_output < target

For bitcoin, the hash_output is determined by

    hash_output = sha256(prev_hash, merkle_root, nonce)

where prev_hash is the previous block’s hash and cannot be changed. The merkle_root is based on the transactions included in the block, and will be different for each individual node. The nonce is rapidly incremented as hash_outputs are calculated and do not satisfy the inequality. Thus the bottleneck of the proofis the sha256 function, and increasing the speed of sha256 or parallelizing it is something ASICs can do very effectively.

Hashimoto uses this hash output as a starting point, which is used to generated inputs for a second hash function. We call the original hash hash_output_A, and the final result of the prooffinal_output.

Hash_output_A can be used to select many transactions from the shared blockchain, which are then used as inputs to the second hash. Instead of organizing transactions into blocks, for this purpose it is simpler to organize all transactions sequentially. For example, the 47th transaction of the 815th block might be termed transaction 141,918. We will use 64 transactions, though higher and lower numbers could work, with different access properties. We define the following functions:

- nonce 64­bits. A new nonce is created for each attempt.
- get_txid(T) return the txid (a hash ofa transaction) of transaction number T from block B.
- block_height the current height ofthe block chain, which increases at each new block

Hashimoto chooses transactions by doing the following:

    hash_output_A = sha256(prev_hash, merkle_root, nonce)
    for i = 0 to 63 do
    	shifted_A = hash_output_A >> i
    	transaction = shifted_A mod total_transactions
    	txid[i] = get_txid(transaction) << i
    end for
    txid_mix = txid[0] ⊕ txid[1] … ⊕ txid[63]
    final_output = txid_mix ⊕ (nonce << 192)

The target is then compared with final_output, and smaller values are accepted as proofs.
