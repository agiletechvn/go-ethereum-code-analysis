**Proof-of-stake（POS)** is an algorithm for the distributed consensus of blockchain networks of cryptocurrency. In Pos-based cryptocurrency, the creator of the next block is selected by combining random selection, wealth value, or age. Conversely, Pow-based cryptocurrencies (such as Bitcoin) determine the creator of the block by cracking the hash puzzle.

## Multiple block selection mechanisms

Proof-of-stake must have a way to define the next valid block in the blockchain. If you only rely on account balances, it will lead to a centralization result, because if the single richest member will have a permanent advantage. Instead, there are several different options that are designed.

### Random block selection

Nxt and BlackCoin use a random way to predict the next block producer. By using a formula, this formula selects the minimum value of the user's share hash value. Argmin hash(stake). Because the shares are public, all nodes can calculate the same value.

### Based on currency age selection

Peercoin's proof-of-stake system combines the concept of random selection and currency age. The age of the coin is the number of coins multiplied by the holding time of the currency. Coins held for more than 30 days will have the opportunity to become the forge of the next block. Users with older coins will have a greater chance to sign the next block. However, once used to sign a block, his currency age will be cleared to zero. You must wait another 30 days before you can sign the next block. Similarly, the maximum age of the coin will only increase to the maximum of 90 days will not increase, in order to avoid the very old age of the user has an absolute role in the blockchain. This process makes the network secure and gradually creates new currencies without consuming very large computing resources. Peercoin's developers affirm that it is more difficult to attack on such a network than Pow, because without a centralized mine, it is more difficult to get 51% of the currency than to get 51% of the calculation.

## Advantage

Proof of Work relies on energy consumption. According to the bitcoin mine operator, the energy consumption of a bitcoin did not reach 240 kWh in 2014 (equivalent to burning 16 gallons of gasoline, in terms of carbon production). And the consumption of energy is paid in non-cryptocurrency. Proof of Stake is thousands of times more efficient than Pow.

The incentives for block producers are also different. In Pow mode, the producer of the block may not have any cryptocurrency. Miners' intention is to maximize their own benefits. It is unclear whether such inconsistencies will reduce the security of the currency or increase the security risks of the system. In the Pos system, these people who are safe in the guardian system always use people with more money.

## Criticism

Some authors believe that pos is not an ideal option for distributed coherence protocols. One of the problems is the usual "nothing at stake" problem, which says that for the producers of the blockchain, voting at the two forked points at the same time does not cause any problems, and this may result in Consistency is difficult to resolve. Because working on multiple chains at the same time consumes very little resources (the root Pow is different), anyone can abuse this feature, so that anyone can spend double on different chains.

There are also many ways to try to solve this problem:

- Ethereum recommends using the Slasher protocol to allow users to punish people who cheat. If someone tries to create a block on multiple different block branches, it is considered a cheater. This proposal assumes that if you want to create a fork you must double sign, if you create a fork without a stake, then you will be punished. However, Slasher has never been adopted. Ethereum developers believe that Pow is a problem worthy of facing. The plan is to replace it with a different Pos protocol CASPER.
- Peercoin uses a centralized broadcast checkpoint approach (signed with the developer's private key). The depth of the blockchain reconstruction cannot exceed the latest checkpoint. The trade-off is that the developer is a centralized authority and controls the blockchain.
- The Nxt protocol only allows the latest 720 block reconstructions. However, this only adjusts the problem. A client can follow a fork with 721 blocks, regardless of whether the fork is the highest blockchain, thus preventing consistency.
- A hybrid agreement between Proof of burn and proof of stake. Proof of burn exists as a checkpoint node, has the highest reward, does not contain transactions, is the safest...
- A mixture of pow and pos. Pos as an extension of dependency pow, based on the Proof of Activity proposal, this proposal hopes to solve the nothing-at-stake problem, using pow miners mining, and pos as the second authentication mechanism. Some blockchain solutions like PIVX using pow and pos mixture.
