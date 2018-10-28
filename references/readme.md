The dotted line contains the subtrees, with the prefixes from top to bottom being 0,01,000,0010.

> According to the above understanding: For any node, you can decompose this binary tree into a series of consecutive subtrees that do not contain your own. The highest-level subtree consists of the other half of the tree that does not contain its own tree; the next subtree consists of the remaining half that does not contain itself; and so on, until the entire tree is split.

The dotted line contains the subtrees, with the prefixes from top to bottom being 1,01,000,0010.

Each such list is called a K bucket, and the internal information storage location of each K bucket is arranged according to the time sequence seen last time. The most recent (least-recently) look is placed on the head, and finally (most-recently) See the tail. Each bucket has no more than k data items.

> Recent (least-recently) and last-most (recently) translations will create ambiguity

The least-recently visited node is placed in the queue header, and the most-recently visited node is placed at the end of the queue. As a result of simulating the analysis of gnutella user behavior, the most recently visited active node is also the node most likely to need access in the future. This strategy uses Epoch.
