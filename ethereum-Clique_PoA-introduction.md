https://github.com/ethereum/EIPs/issues/225

In Clique mode, users can't get Ethereum because they can't mine, so if you need Ethereum, you need to get it through special channels.

You can get ether from this website.

https://faucet.rinkeby.io/

You need to have a google+ account, facebook or twitter account to get it. For detailed information, please refer to the above website.

Clique is a Power of authority implementation of Ethereum and is now primarily used on the Rinkeby test network.

## Background

The first official test network of Ethereum was Morden. From July 2015 to November 2016, due to the accumulated garbage between Geth and Parity and some testnet consensus issues, it was decided to stop restarting testnet.

Ropsten was born, cleaned up all the rubbish, starting with a clean slate. This operation continued until the end of February 2017, when malicious actors decided to abuse Pow and gradually increased GasLimit from the normal 4.7 million to 9 billion, at which time a huge transaction was sent to damage the entire network. Even before this, the attacker tried several very long blockchain reorganizations, resulting in network splits between different clients, even different versions.

The root cause of these attacks is that the security of the PoW network is as secure as the computing power behind it. Restarting a new test network from scratch does not solve any problems because the attacker can install the same attack over and over again. The Parity team decided to take an urgent solution, roll back a large number of blocks, and develop a soft cross that does not allow GasLimit to exceed a certain threshold.

Although this solution may work in the short term:

This is not elegant: Ethereum should have dynamic restrictions. This is not portable: other customers need to implement new fork logic on their own. It is not compatible with synchronous mode: fast sync and light client are both bad luck. This only extends the attack. Time: Garbage can still steadily advance in the endless situation Parity's solution is not perfect, but still feasible. I want to come up with a longer-term alternative solution that involves more, but should be simple enough to be launched in a reasonable amount of time.

## Standardized PoA

As mentioned above, in a network with no value, Pow cannot work safely. Ethereum has a long-term PoS goal based on Casper, but this is a tedious study, so we can't rely on it to solve today's problems. However, a solution is easy to implement and is sufficiently efficient to properly repair the test network, the proof-of-authority scheme.

Note that Parity does have a PoA implementation, although it looks more complicated than needed, there is not much protocol documentation, but it's hard to see it can be played with other customers. I welcome them to give me more feedback on this proposal based on their experience.

The main design goal of the PoA protocol described here is to implement and embed any existing Ethereum client should be very simple, while allowing the use of existing synchronization technologies (fast, easy, and distorted) without the need for client developers. Add custom logic to critical software.

## PoA101

For those who don't realize how PoA works, this is a very simple agreement, rather than the miners competing to solve a difficult problem, and the signatory can decide whether to create a new block at any time.

The challenge revolves around how to control the mining frequency, how to distribute the load (and opportunity) between different signers, and how to dynamically adjust the signer list. The next section defines a recommended protocol for handling all of these scenarios.

## Rinkeby proof-of-authority

In general, there are two ways to synchronize blockchains:

- The traditional approach is to start all blocks and tighten the transactions one by one. This approach has been tried and has proven to be very computationally intensive in a complex network such as Ethereum. The other is to download only the blockchains and verify their validity, after which you can download an arbitrary recent state from the network and check the nearest header.
- The PoA solution is based on the idea that blocks can only be completed by trusted signers. Therefore, each block (or header) seen by the client can match the list of trusted signers. The challenge here is how to maintain a list of authorized signers that can be changed in time? The obvious answer (stored in the Ethereum contract) is also the wrong answer: it is inaccessible during fast synchronization.

**The protocol that maintains the list of authorized signers must be fully contained in the block header.**

The next obvious idea is to change the structure of the block header so that you can abandon the concept of PoW and introduce new fields to cater to the voting mechanism. This is also the wrong answer: Changing such a core data structure in multiple implementations will be a nightmare for development, maintenance, and security.

**The protocol that maintains the list of authorized signers must be fully adapted to the current data model.**

So, according to the above, we can't use the EVM to vote, but have to resort to the block header. And we can't change the block header field and have to resort to the currently available fields. There are not many choices.

### Use some other fields of the block header to implement voting and signature

The most obvious field currently used only as interesting metadata is the 32-byte ExtraData portion of the block header. Miners usually put their clients and versions there, but some people fill them with additional "information." The protocol will extend this field to add 65 bytes to store the miner's KEC signature. This will allow anyone who gets a block to verify it based on the list of authorized signers. At the same time it also invalidates the field of the miner address in the block header.

Note that changing the length of the block header is a non-intrusive operation because all code (such as RLP encoding, hashing) is agnostic, so the client does not need custom logic.

The above is enough to verify a chain, but how do we update a dynamic list of signers. The answer is that we can re-use the newly obsolete miner field beneficiary and the PoA obsolete nonce field to create a voting protocol:

- In a regular block, both fields will be set to zero.
- If the signer wishes to make changes to the list of authorized signers, it will:
  - Set the miner field **beneficiary** to the signer who wishes to vote
  - Set the **nonce** to 0 or 0xff ... f to vote for adding or kicking out

Clients of any synchronization chain can "count" votes during block processing and maintain a dynamic list of authorized signers by ordinary voting. The initial set of signers is provided by the parameters of the genesis block (to avoid the complexity of deploying the "initial voter list" contract in the initial state).

To avoid having an infinite window to count votes and to allow regular elimination of stale proposals, we can reuse ethash's concept epoch, and each epoch conversion will refresh all pending votes. In addition, these epoch conversions can also be used as stateless checkpoints that contain a list of currently authorized signers within the header's extra data. This allows the client to synchronize based only on the checkpoint hash without having to replay all the votes made on the chain. It also allows the complete definition of the blockchain with the genesis block that contains the initial signer.

### Attack vector: malicious signer

It may happen that a malicious user is added to the list of signers or the signer's key/machine is compromised. In this case, the agreement needs to be able to withstand restructuring and spam. The proposed solution is that given a list of N authorized signers, any signer may only fill 1 block per K. This ensures that damage is limited and the remaining miners can cast malicious users.

## Attack vector: review signer

Another interesting attack vector is if a signer (or a group of signers) tries to check out the blocks that removed them from the authorization list. To solve this problem, we limit the minimum frequency allowed by the signer to N / 2. This ensures that the malicious signer needs to control at least 51% of the signed account, in which case the game will not be able to proceed anyway.

## Attack vector: spammer signer

The final small attack vector is that malicious signers inject new voting suggestions into each block. Since the node needs to count all votes to create an actual list of authorized signers, they need to track all votes by time. There is no limit to the voting window, which may grow slowly, but it is unlimited. The solution is to place a W block's moving window, after which the vote is considered stale. A sensible window may be 1-2 times. We call this an epoch.

## Attack vector: Concurrent blocks

If the number of authorized signers are N, and we allow each signer to mint 1 block out of K, then at any point in time N-K+1 miners are allowed to mint. To avoid these racing for blocks, every signer would add a small random "offset" to the time it releases a new block. This ensures that small forks are rare, but occasionally still happen (as on the main net). If a signer is caught abusing it's authority and causing chaos, it can be voted out.

## Notes

### Does this indicate that we recommend using a testnet that is being reviewed?

The proposal suggests that, given the malicious nature of certain actors and the weaknesses of the PoW program in the “monopoly funds” network, it is best to establish a network with a certain garbage filtering function that developers can rely on to test Its program.
Why regulate PoA?
Different customers will be better in different situations. Go may be great in a server-side environment, but CPP may be better suited to run on RPI Zero.
Is manual voting a lot of trouble?
This is an implementation detail, but the signer can use the contract-based voting strategy to take advantage of the full power of the EVM and push only the results to the head of the average node for verification.

## Clarification and feedback

- This recommendation does not preclude the client from running a PoW-based test network, whether it is Ropsten or a new test network based on it. Ideally, customers offer a way to connect PoW and PoA-based test networks (#225 (comment)).

- Although the protocol parameters can be configured in the destruction of the client implementer, the Rinkeby network should be as close as possible to the primary network. This includes dynamic GasLimit, variable block time of around 15 seconds, GasPrice, etc. (#225 (comment)).

- The program requires at least K signers to stay online, as this is the minimum number of people needed to ensure “minimize” diversity. This means that if K is exceeded, the network stops. This should be resolved by ensuring that the signer is a high-running machine, and that the failed machine is voted out in time (#225 (comment)) before too many failures occur.

- The proposal does not address the "legal" spam issue, just as an attacker effectively uses testnet to create garbage, but without PoW mining, an attacker may not be able to gain unlimited ether attacks. One possibility is to provide a way to get ether based on a GitHub (or any other way) account in a limited way (eg 10 times a day) (#225 (comment)).

- It has been suggested to create a checkpoint block for each epoch that contains a list of authorized signers at the time. This will allow later light customers to say "synchronize from here" without having to start from the origin. This can be added to the extradata field (#225(comment)) as a prefix before signing.

## Clique PoA Protocol (Clique proof-of-authority consensus protocol )

We define the following constants:

- EPOCH_LENGTH：Checkpoints and reset the number of blocks for pending voting.
  - Recommended 30000 to be similar to the ethhash epoch of the main network
- BLOCK_PERIOD：The smallest difference between the timestamps of two consecutive blocks.
  - Recommendation 15, to be similar to the ethhash epoch of the main network
- EXTRA_VANITY：A fixed number of ExtraData prefix bytes are reserved for the signer vanity.
  - The recommended 32 bytes are the same length as the current ExtraData.
- EXTRA_SEAL：A fixed number of extra data suffix bytes reserved for the signer's stamp.
  - The 65 bytes of the signature are saved, based on the standard secp256k1 curve.
- NONCE_AUTH：Magic random number 0xffffffffffffffff vote to add a new signer.
- NONCE_DROP：The magic random number 0x0000000000000000 votes to remove the signer.
- UNCLE_HASH：Always Keccak256 (RLP([])) as Uncles has no meaning outside of PoW.
- DIFF_NOTURN：If you don't have your signature yet, the difficulty of the block you signed is the difficulty.
  - Recommendation 1, because it only needs to be an arbitrary baseline constant.
- DIFF_INTURN：If your current turn is your signature, then the difficulty of your signature.
  - Recommendation 2, which is more difficult than a signer who does not have a turn.

We also define the following constants for each block:

- BLOCK_NUMBER：The height of the block in the chain. The height of the genesis block is 0.
- SIGNER_COUNT：The number of authorized signers that are valid on a particular instance in the blockchain.
- SIGNER_INDEX：The index in the sorted list of the current authorized signer.
- SIGNER_LIMIT：Every so many blocks, the signer can only sign one. - There must be floor(SIGNER_COUNT / 2)+1 so many signers agree to reach a resolution.

We re-adjust the purpose of the block header field as follows:

- beneficiary：It is recommended to modify the address of the authorized signer list. - Zero should be filled normally, only when voting.

  - Nonetheless, arbitrary values ​​(even meaningless values, such as throwing non-signers) are allowed to avoid adding extra complexity around the voting mechanism.
  - The block must be padded with zeros at the checkpoint (ie epoch conversion).

- nonce：Signer's suggestion about the account defined in the Beneficiary field.
  - NONCE_DROP proposes to deauthorize the beneficiary as an existing signer.
  - NONCE_AUTH proposes to authorize beneficiaries as new signers.
  - The block must be filled with zeros at the checkpoint.
  - No other value other than the above (now).
- extraData： The combined field of vanity, checkpointing and signer signatures.
  - The first EXTRA_VANITY byte (fixed length) can contain any signer vanity data.
  - The last EXTRA_SEAL byte (fixed length) is the signer signature of the sealed title.
  - The checkpoint block must contain a list of signers (N \* 20 bytes), otherwise omitted.
  - The list of signers in the Additional Data section of the checkpoint block must be sorted in ascending order.
- mixHash：Reserved for forks. Additional data similar to Dao
  - Zero must be filled during normal operation.
- ommersHash：Must be UNCLE_HASH, because Uncles has no meaning outside of PoW.
- timestamp：Must be at least the timestamp of the parent block + BLOCK_PERIOD.
- difficulty：Contains the independent score of the block to derive the quality of the chain.
  - If BLOCK_NUMBER%SIGNER_COUNT! = SIGNER_INDEX, must be DIFF_NOTURN
  - Must be DIFF_INTURN if BLOCK_NUMBER%SIGNER_COUNT == SIGNER_INDEX

### Authorizing a block

In order to authorize a block for the network, the signer needs to sign everything that contains the signature itself. This means that the hash contains every field in the block header (including nonce and mixDigest), as well as extraData in addition to the 65-byte signature suffix. These fields are hashed in the order they are defined in the Yellow Book.

The hash is signed using the standard secp256k1 curve, and the resulting 65-byte signature (R, S, V, where V is 0 or 1) is embedded in the extraData as a trailing 65-byte suffix.

To ensure that a malicious signer (signature key loss) cannot be compromised on the network, each signer can sign up to one SIGNER_LIMIT contiguous block. The order is not fixed, but the signer of (DIFF_INTURN) is more difficult to check out than (DIFF_NOTURN)

#### Authorization strategy

As long as the signer meets the above specifications, they can authorize and assign the blocks they think are appropriate. The following suggested strategies will reduce network traffic and forks, so this is a suggested feature:

- If the signer is allowed to sign a block (on the authorization list and has not recently signed).
  - Calculate the best signature time for the next block (parent + BLOCK_PERIOD).
  - If it is the turn, wait for the exact time to arrive, sign and play immediately.
  - If there is no turn, delay rand (SIGNER_COUNT \* 500ms) for a long time signature. This small strategy will ensure that the current turn of the signer (who is heavier) has a slight advantage over signing and propagating and outbound signers. In addition, the scheme allows for a certain scale as the number of signers increases.

### Voting signer

Each epoch conversion (including the genesis block) acts as a stateless checkpoint, and capable clients should be able to synchronize without any previous state. This means that the new epoch header must not contain votes, all uncommitted votes will be discarded and counted from the beginning.

For all non-epoch conversion blocks:

- The signer can vote for the block using his own signed block to make a change to the authorization list.
- Only one latest vote is kept for each proposal.
- As the chain progresses, the vote will also take effect (allowing the proposal to be submitted at the same time).
- The proposal SIGNER_LIMIT, which reached the majority opinion, takes effect immediately.
- Invalid proposals are not penalized for the simplicity of the client implementation.

**The proposal in force means giving up all pending votes on the proposal (whether in favor or against) and starting with a clear list.**

### Cascade voting

Complex cases may occur during signer cancellation. If the previously authorized signer is revoked, the number of signers required to approve the proposal may be reduced by one. This may lead to a consensus on one or more pending proposals, which may further affect the new proposal.

When multiple conflicting proposals are passed at the same time (for example, adding a new signer vs deleting an existing proposer), it is not obvious that the situation is evaluated, and the evaluation order may completely change the result of the final authorization list. Since the signer may reverse their own vote in each of their own blocks, it is not so obvious which proposal will be "first".

In order to avoid the defects caused by cascading implementation, the solution is to explicitly prohibit the cascading effect. In other words: only the current title/voting beneficiary can be added to or removed from the authorization list. If this leads to consensus on other recommendations, then when their respective beneficiaries “trigger” again, these recommendations will be implemented (because most people's consensus is still at this point).

### Voting strategy

Since the blockchain can have very small reorgs, the naïve voting mechanism of "cast-and-forget" may not be optimal, as blocks containing singleton votes may not end in the final chain.

A simple but working strategy is to allow users to configure "proposals" on the signer (eg "add 0x ...", "drop 0x ..."). Sign the code and then you can choose a random recommendation for each block it signs and injects. This ensures that multiple concurrent proposals and reorgs are ultimately noticed on the chain.

This list may expire after a certain number of blocks/epoch, but it is important to realize that "seeing" a proposal does not mean that it will not be reassembled, so it should not be abandoned immediately when the proposal passes.
