
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Hashimoto :I/O bound proof of work


Abstract: Using a cryptographic hash function not as a proofofwork by itself, but
Rather as a generator of pointers to a shared data set, allows for an I/O bound
Proof of work. This method of proof of work is difficult to optimize via ASIC
Design, and difficult to outsource to nodes without the full data set. The name is
Based on the three operations which comprises the algorithm: hash, shift, and
Modulo.

Abstract: Using the cryptographic hash function itself is not a proof of work.
Instead, as a pointer generator to a shared data set, allowing I/O binding
Proof of employment. This proof of work method is difficult to optimize through ASIC design and is difficult to outsource to nodes without a complete data set. This name is based on three operations that make up the algorithm: hash, shift, and
mold.


The need for proofs which are difficult to outsource and optimize

Workload proves difficult to outsource and optimize demand

A common challenge in cryptocurrency development is maintaining decentralization ofthe
Network. The use ofproofofwork to achieve decentralized consensus has been most notably
By bitcoin, which uses partial collisions with zero ofsha256, similar to hashcash. As
Bitcoin’s popularity has grown, dedicated hardware (currently application specific integrated circuits, or
ASICs) has been produced to rapidly iterate the hash­based proofofwork function. Newer projects
Similar to Bitcoin often use different algorithms for proofofwork, and often with the goal ofASIC
For algorithms such as Bitcoin’s, the improvement factor ofASICs means that commodity
Computer hardware can no longer be useful used, potentially limiting adoption.

One of the challenges in the development of cryptocurrencies is how to maintain a decentralized network structure. Just as Bitcoin uses the workload proof of the sha256 hash puzzle to achieve decentralization consistency. With the popularity of bitcoin, dedicated hardware (currently ASICs, or ASICs) has been used to quickly perform hash-based workload proof functions. New projects like Bitcoin often use different workload proofing algorithms and often have targets that resist ASICs. For algorithms such as Bitcoin, the performance improvement of ASICs means that ordinary commercial computer hardware is no longer effectively used and may be restricted.

Proofofwork can also be "outsourced", or performed by a dedicated machine (a "miner")
Without knowledge of what is being verified. This is often the case in Bitcoin’s “mining pools”. It is also
Beneficial for a proofofwork algorithm to be difficult to outsource, in order to promote decentralization
And encourage all nodes participating in the proofofwork process to also verify transactions.
Goals in mind, we present Hashimoto, an I/O bound proofofwork algorithm we believe to be resistant to
Both ASIC design and outsourcing.

Proof of work can also be outsourced, or a dedicated machine (mining machine) can be used to perform proof of work, and these machines are not clear about the content of the verification. This is usually the case with Bitcoin's “mine pool”. If the workload proof algorithm is difficult to outsource, to promote decentralization
All nodes participating in the attestation process are also encouraged to verify the transaction. To achieve this goal, we designed hashimoti, a workload proofing algorithm based on I/O bandwidth, which we believe can resist ASICs and is difficult to outsource.

Initial attempts at "ASIC resistance" involved changing Bitcoin's sha256 algorithm for a different,
More memory intensive algorithm, Percival's "scrypt" password based key derivation function1. Many
Implementations set the scrypt arguments to low memory requirements, defeating much of the purpose of
The key derivation algorithm. while changing to a new algorithm, coupled with the relative obscurity of the
Various scrypt­based cryptocurrencies allowed for a delay, scrypt optimized ASICs are now available.
Similar attempts at variations or multiple heterogeneous hash functions can at best only delay ASIC
Implementations.

The initial attempts of "ASIC resistance" included changing the bitcoin's sha256 algorithm with a different, more memory-intensive algorithm, Percival's "scrypt" password based key derivation function. Many implementations set script parameters to low memory requirements, which greatly undermines the purpose of the key derivation algorithm. While switching to the new algorithm, the relative enthusiasm of various scrypt-based cryptocurrencies may lead to delays, and scrypt-optimized ASICs are now available. Similar change attempts or multiple heterogeneous hash functions can only delay the ASIC implementation at most.

Leveraging shared data sets to create I/O bound proofs

Create I/O limit proofs with shared data sets

"A supercomputer is a device for turning compute-bound problems into I/O-bound problems."
-Ken Batcher


"Supercomputers are a device that converts computationally constrained problems into I/O constraints."
Ken Batcher

Instead, an algorithm will have little room to be sped up by new hardware if act in a way that commodity computer systems are already optimized for.

Conversely, if an algorithm is run in a way that the commodity computer system has been optimized, then the algorithm will have little room to be accelerated by the new hardware.

Since I/O bounds are what decades of computing research has gone towards solving, it's unlikely that the relatively small motivation ofmining a few coins would be able to advance the state of the art in cache hierarchies. In the case that advances are made, they will be Possible to impact the entire industry of computer hardware.

Since the I/O bounds are a problem that has been solved by computational research for decades, the relatively small motives for mining some cryptocurrencies will not increase the art level of the cache hierarchy. In the case of progress, it may affect the entire computer hardware industry.

Fortuitously, all nodes participating in current implementations of cryptocurrency have a large set of mutually agreed upon data; Indeed this "blockchain" is the foundation of the currency. Using this large data set can both limit the advantage of specialized power, and require working nodes to have the Entire data set.

Fortunately, all nodes participating in the current cryptocurrency implementation have a large amount of mutually agreed data; in fact, the "blockchain" is the basis of money. Using this big data set not only limits the benefits of dedicated hardware, but also allows the worker node to own the entire data set.

Hashimoto is based offBitcoin’s proofofwork2. In Bitcoin’s case, as in Hashimoto, a successful
Proofsatisfies the following inequality:

Hashimoto is based on Bitcoin's proof of workload. In the case of Bitcoin, like Hashimoto, a successful proof satisfies the following inequality:

Hash_output &lt; target

For bitcoin, the hash_output is determined by

In Bitcoin, hash_output is determined by the following.

Hash_output = sha256(prev_hash, merkle_root, nonce)

Where the prev_hash is the previous block's hash and cannot be changed. The merkle_root is based on the transactions included in the block, and will be different for each individual node. The nonce is rapidly incremented as hash_outputs are calculated and do not satisfy the inequality. The bottleneck of the proofis the sha256 function, and increasing the speed of sha256 or parallelizing it is something ASICs can do very effectively.

Prev_hash is the hash value of the previous block and cannot be changed. Merkle_root is generated based on transactions in the block and will be different for each individual node. We make the above inequality true by modifying the value of nonce. The bottleneck that proves the whole workload is the sha256 method, and the calculation speed of sha256 can be greatly increased by the ASIC, or it can be run in parallel.

Hashimoto uses this hash output as a starting point, which is used to generate inputs for a second hash function. We call the original hash hash_output_A, and the final result of the prooffinal_output.

Hashimoto uses this hash_output as a starting point to generate input for the second hash function. We call the original hash hash_output_A, and the final result is prooffinal_output.

Hash_output_A can be used to select many transactions from the shared blockchain, which are then used as inputs to the second hash. Instead of organizing transactions into blocks, for this purpose it is simpler to organize all transactions sequentially. For example, the 47th transaction of The 815th block might be termed transaction 141,918. We will use 64 transactions, though higher and below numbers could work, with different access properties. We define the following functions:

Hash_output_a can be used to select multiple transactions from a shared blockchain and then use it as input for the second hash. Rather than organizing transactions into chunks, it is easier to organize all transactions in order for this purpose. For example, the 47th transaction of the 815th block may be referred to as transaction 141,918. We will use 64 transactions, although higher and lower numbers can work with different access properties. We define the following features:

- nonce 64­bits. A new nonce is created for each attempt.
- get_txid(T) return the txid (a hash ofa transaction) of transaction number T from block B.
- block_height the current height of the block chain, which increases at each new block

- nonce 64­bits. A new nonce value is generated for each attempt.
- get_txid(T) Get the transaction id from block B by transaction number
- block_height current block height

Hashimoto chooses transactions by doing the following:

Hashimoto uses the following algorithm to select transactions:

hash_output_A = sha256(prev_hash, merkle_root, nonce)
For i = 0 to 63 do
shifted_A = hash_output_A &gt;&gt; i
Transaction = shifted_A mod total_transactions
Txid[i] = get_txid(transaction) &lt;&lt; i
End for
Txid_mix = txid[0] ⊕ txid[1] ... ⊕ txid[63]
Final_output = txid_mix ⊕ (nonce &lt;&lt; 192)

The target is then compared with final_output, and smaller values ​​are accepted as proofs.

If final_output is smaller than target, it will be accepted.


</pre></body></html>