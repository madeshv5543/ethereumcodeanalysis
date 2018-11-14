
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>The translated content is from [ [https://github.com/ethereum/go-ethereum/pull/1889] (https://github.com/ethereum/go-ethereum/pull/1889)

This PR aggregates a lot of small modifications to core, trie, eth and other packages to collectively implement the eth/63 fast synchronization algorithm. In short, geth --fast.

This commit request contains minor modifications to core, trie, eth, and other packages to implement the fast synchronization algorithm for eth/63. In simple terms, geth --fast.

## Algorithm Algorithm

The goal of the fast sync algorithm is to exchange processing power for bandwidth usage. Instead of processing the entire block-chain one link at a time, and replay all transactions that ever happened in history, fast syncing downloads the transaction receipts along the blocks And pulls an entire recent state database. This allows a fast synced node to still retain its status an an archive node containing all historical data for user queries (and thus not influence the network's health in general), but at the same time to reassemble a recent network state at a fraction of the time it would take full block processing.

The goal of the fast synchronization algorithm is to use bandwidth for calculations. Fast synchronization does not process the entire blockchain through a link, but replays all transactions that have occurred in history. Fast synchronization downloads transactional documents along these blocks and then pulls the entire state database. This allows the fast-synchronizing node to maintain its state of the archive node containing all historical data for the user's query (and therefore does not generally affect the health of the network), for the most recent block state change, the full-size zone is used Block processing method.

An outline of the fast sync algorithm would be:

- similar to classical sync, download the block headers and bodies that make up the blockchain
- similar to classical sync, verify the header chain's consistency (POW, total difficulty, etc)
- Instead of processing the blocks, download the transaction receipts as defined by the header
- Store the downloaded blockchain, along with the receipt chain, enabling all historical queries
- When the chain reaches a recent enough state (head - 1024 blocks), pause for state sync:
- Retrieve the entire Merkel Patricia state trie defined by the root hash of the pivot point
- For every account found in the trie, retrieve it's contract code and internal storage state trie
- Upon successful trie download, mark the pivot point (head - 1024 blocks) as the current head
- Import all remaining blocks (1024) by fully processing them as in the classical sync

Summary of the fast synchronization algorithm:

- Similar to the original sync, download the block header and block body that make up the blockchain
- Similar to the original synchronization, verify the consistency of the block header (POW, total difficulty, etc.)
- Download the transaction receipt defined by the block header instead of processing the block.
- Store downloaded blockchains and receipt chains to enable all historical queries
- Pause state synchronization when the chain reaches the most recent state (header - 1024 blocks):
- Get the complete Merkel Patricia Trie status of the block defined by the pivot point
- For each account in Merkel Patricia Trie, get his contract code and intermediate stored Trie
- When the Merkel Patricia Trie is successfully downloaded, the block defined by the pivot point is used as the current block header.
- Import all remaining blocks (1024) by completely processing them like the original sync

## Analysis Analysis
By downloading and verifying the entire header chain, we can guarantee with all the security of the classical sync, that the hashes (receipts, state tries, etc) contained within the headers are valid. Based on those hashes, we can confidently download transaction receipts And the entire state of the current state (1024 blocks), we can ensure that even larger chain reorganizations can be handled without the need of a New sync (as we have all the state going that many blocks back).

By downloading and verifying the entire header chain, we can guarantee all the security of traditional synchronization, and the hashes (receipts, state attempts, etc.) contained in the header are valid. Based on these hashes, we can confidently download the transaction receipt and the entire state tree. In addition, by placing the pivoting point (fast sync switch to block processing) below the current block header (1024 blocks), we can ensure that even larger blockchain reorganizations can be handled without new synchronization (because of the new synchronization (because of We have all the states TODO).

## Notes Caveats
The historical block-processing based synchronization mechanism has two (approximately similarly costing) bottlenecks: transaction processing and PoW verification. The baseline fast sync algorithm successfully circumvents the transaction processing, skipping the need to iterate over every single state the system ever was in. Verifying the proof of work associated with each header is still a notably CPU intensive operation.

The synchronization mechanism based on historical block processing has two (approximate similar cost) bottlenecks: transaction processing and PoW verification. The baseline fast synchronization algorithm successfully bypasses the transaction and skips the need to iterate over every state the system was in. However, verifying the proof of work associated with each header is still CPU intensive.

With a negligible probability of header, we can still guarantee the validity of the chain, only by verifying every K-th header, instead of each and every one. By selecting a single header At random out of every K headers to verify, we guarantee the validity of an N-length chain with the probability of (1/K)^(N/K) (ie we have 1/K chance to spot a forgery in K blocks , a verification that's repeated N/K times).

However, we can notice an interesting phenomenon during the block header verification. Since the error probability is negligible, we can still guarantee the validity of the chain, and only need to verify each Kth header instead of each header. By verifying by randomly selecting a header from each of the K headers, we guarantee that the probability that the N-length chain may be forged is (1 / K)^(N / K) (we have a 1/K chance in the K-block) Found a forgery, and the verification passed N/K times.).

Let's define the negligible Pn as the probability of obtaining a 256 bit SHA3 collision (ie the hash Ethereum is built upon): 1/2^128. To honor the Ethereum security requirements, we need to choose the minimum chain length N (below Which we veriy every header) and maximum K verification batch size such as (1/K)^(N/K) &lt;= Pn holds. Calculating this for various {N, K} pairs is pretty straighforward, a simple and lenient solution being http://play.golang.org/p/B-8sX_6Dq0.

We define the negligible probability Pn as the probability of obtaining a 256-bit SHA3 collision (Ethernet's Hash algorithm): 1/2 ^ 128. In order to comply with the safety requirements of Ethereum, we need to choose the minimum chain length N (before each block is verified), and the maximum K verification batch size is (1 / K)^(N / K)&lt;= Pn. It is very straightforward to calculate the various {N, K} pairs. A simple and loose solution is http://play.golang.org/p/B-8sX_6Dq0.

|N |K |N |K |N |K |N |K |
| ------|-------|-------|-----------|-------|------ -----|-------|---|
|1024 |43 |1792 |91 |2560 |143 |3328 |198|
|1152 |51 |1920 |99 |2688 |152 |3456 |207|
|1280 |58 |2048 |108 |2816 |161 |3584 |217|
|1408 |66 |2176 |116 |2944 |170 |3712 |226|
|1536 |74 |2304 |128 |3072 |179 |3840 |236|
|1664 |82 |2432 |134 |3200 |189 |3968 |246|


The above table should be interpreted in such a way, that if we verify every K-th header, after N headers the probability of a forgery is smaller than the probability of an attacker producing a SHA3 collision. It also means, that if a forgery Is indeed detected, the last N headers should be discarded as not enough enough. Any {N, K} pair may be chosen from the above table, and to keep the numbers reasonably looking, we chose N=2048, K=100. Will be fine tuned later after being able to observe network bandwidth/latency effects and possibly behavior on more CPU limited devices.

The above table should be interpreted as follows: If we verify the block header every K block headers, after N block headers, the probability of forgery is less than the probability of an attacker generating a SHA3 collision. This also means that if forgery is indeed found, then the last N headers should be discarded because it is not secure enough. You can choose any {N, K} pair from the above table. In order to choose a number that looks good, we choose N = 2048, K = 100. Subsequent adjustments may be made based on network bandwidth/latency effects and possible operation on some devices with limited CPU performance.

Using this caveat however would mean, that the pivot point can be considered secure only after N headers have been imported after the pivot itself. To prove the pivot safe faster, we stop the "gapped verificatios" X headers before the pivot point, and verify Every single header onward, including an additioanl X headers post-pivot before accepting the pivot's state. Given the above N and K numbers, we chose X=24 as a safe number.

However, using this feature means that importing only n blocks and then importing the pivot node is considered safe. In order to prove the security of the pivot more quickly, we stop the block verification behavior at a distance X from the pivot node, and verify each block that appears next until the pivot. In view of the above N and K numbers, we choose X = 24 as the safety number.

With this caveat calculated, the fast sync should be modified so that up to the pivoting point - X, only every K=100-th header should be verified (at random), after which all headers up to pivot point + X should be fully Note before starting state database downloading. Note: if a sync fails due to header verification the last N headers must be discarded as they cannot be trusted enough.

By calculating caveat, fast synchronization needs to be modified to pivoting point - X, and one of every 100 blocks is randomly selected for verification. Each subsequent block needs to be fully verified after the state database download is completed, if the block header is verified. The failure caused by the failure of the failure, then the last N block headers need to be discarded, they should not reach the trust standard.


## Disadvantages Weakness
The blockchain protocols in general (ie Bitcoin, Ethereum, and the others) are susceptible to Sybil attacks, where an attacker tries to completely isolate a node from the rest of the network, making it believe a false truth as to the the state of the real Network attack. This permits the attacker to spend certain funds in both the real network and this "fake bubble". However, the attacker can only maintain this state as long as it's feeding new valid blocks it itself is forging; and to successfully shadow the Real network, it needs to do this with a chain height and difficulty close to the real network. In short, to pull off a successful Sybil attack, the attacker needs to match the network's hash rate, so it's a very expensive attack.

Common blockchains (such as Bitcoin, Ethereum, and others) are more susceptible to witch attacks, and attackers attempt to completely isolate the attacker from the primary network, allowing the attacker to receive a false state. This allows an attacker to spend the same amount of money on a real network while on this fake network. However, this requires the attacker to provide real self-forged blocks, and the need to successfully affect the real network requires close to the real network in terms of block height and difficulty. In short, in order to successfully implement a witch attack, the attacker needs to approach the hash rate of the main network, so it is a very expensive attack.

Compared to the classical Sybil attack, fast sync provides such an attacker with an extra ability, that of feeding a node a view of the network that's not only different from the real network, but also that might go around the EVM mechanics. The Ethereum protocol But validive state root hashes by processing all the transactions against the previous state root. But by skipping the transaction processing, we cannot prove that the state root contained within the fast sync pivot point is valid or not, so as long as an attacker can maintain a fake blockchain that's on par with the real network, it could create an invalid view of the network's state.

Compared to traditional witch attacks, fast synchronization provides an additional capability for attackers to provide a network view that is not only different from the real network, but also bypasses the EVM mechanism. The Ethereum protocol verifies the state root hash only by processing all transactions with the previous state root. But by skipping the transaction, we can't prove that the state root contained in the fast synchronization pivot point is valid, so an attacker can maintain an invalid network state view as long as it can maintain the same fake blockchain as the real network.

To avoid opening up nodes to this extra attacker ability, fast sync (beside being sole opt-in) will only ever run during an initial sync (ie when the node's own blockchain is empty). After a node managed to successfully sync with the network This way anybody can quickly catch up with the network, but after the node caught up, the extra attack vector is plugged in. This feature permits users to safely use the fast sync flag (--fast), As having additional safety feature, if a fast sync fails close to or after the random pivot point, fast sync is disabled as a safety precaution and the node reverts to full , block-processing based synchronization.

In order to avoid opening the node to this additional attacker capability, fast synchronization (specially specified) will only run during the initial synchronization (the node's local blockchain is empty). Fast synchronization is always disabled after a node successfully synchronizes with the network. This way anyone can catch up with the network quickly, but after the node catches up, the extra attack vector is inserted. This feature allows users to safely use the fast sync flag (--fast) without worrying about the potential state of the root attack in the future. As an added security feature, if fast synchronization fails near or after a random pivot point, fast synchronization is disabled as a security precaution and the node reverts to full synchronization based on block processing.

## Performance Performance
To benchmark the performance of the new algorithm, four separate tests were run: full syncing from scrath on Frontier and Olympic, using both the classical sync as well as the new sync mechanism. In all scenarios there were two nodes running on a single machine: a seed node featuring a fully synced database, and a leech node with only the genesis block pulling the data. In all test scenarios the seed node had a fast-synced database (smaller, less disk contention) and both nodes were given 1GB database cache (--cache=1024).

To benchmark the performance of the new algorithm, four separate tests were run: full synchronization from the scrath on Frontier and Olympics using classic synchronization and a new synchronization mechanism. In all cases, two nodes are running on one machine: a seed node with a fully synchronized database, and a mink node with only the starting block pulling data. In all test scenarios, the seed node has a fast-synchronizing database (smaller, less disk contention), and both nodes have a 1GB database cache (--cache = 1024).

The machine running the tests was a Zenbook Pro, Core i7 4720HQ, 12GB RAM, 256GB m.2 SSD, Ubuntu 15.04.

The machines running the tests were Zenbook Pro, Core i7 4720HQ, 12GB RAM, 256GB m.2 SSD, Ubuntu 15.04.

| Dataset (blocks, states) | Normal sync (time, db) | Fast sync (time, db) |
| ------------------------- |:---------------------- ---:| ---------------------------:|
|Frontier, 357677 blocks, 42.4K states | 12:21 mins, 1.6 GB | 2:49 mins, 235.2 MB |
|Olympic, 837869 blocks, 10.2M states | 4:07:55 hours, 21 GB | 31:32 mins, 3.8 GB |


The resulting databases contain the entire blockchain (all blocks, all uncles, all transactions), every transaction receipt and generated logs, and the entire state trie of the head 1024 blocks. This allows a fast synced node to act as a full archive node from All intents and purposes.

The resulting database contains the entire blockchain (all blocks, all blocks, all transactions), each transaction receipt and generated logs, and the entire state tree for the first 1024 blocks. This allows a fast synchronization node to act as a full archive node for all intents and purposes.


## Conclusion Closing remarks
The fast sync algorithm requires the functionality defined by eth/63. Because of this, testing in the live network requires for at least a handful of discoverable peers to update their nodes to eth/63. On the same note, verifying that the implementation is Truly correct will also entail waiting for the wider deployment of eth/63.

The fast synchronization algorithm requires the functionality defined by eth / 63. Because of this, tests in the live network require at least a few discoverable peer nodes to update their nodes to eth / 63. The same note, verifying that this implementation is really correct still needs to wait for a broader deployment of eth / 63.</pre></body></html>