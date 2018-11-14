
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Eth source code and several packages below

- downloader is mainly used to synchronize with the network, including the traditional synchronization method and fast synchronization method.
- fetcher is mainly used for block-based notification synchronization. When we receive the NewBlockHashesMsg message, we only receive a lot of block hash values. The hash value needs to be used to synchronize the block.
- filter Provides RPC-based filtering, including real-time data synchronization (PendingTx), and historical log filtering (Log filter)
- gasprice offers price advice for gas, based on the gasprice of the last few blocks, to get the current price of gasprice


Partial source analysis of eth protocol

- [Ethernet's network protocol approximate process] (eth Ethereum protocol analysis.md)

Source analysis of the fetcher part

- [fetch partial source code analysis] (eth-fetcher source code analysis. md)

Downloader partial source code analysis

- [Node Fast Synchronization Algorithm] (Ethereum fast%20sync algorithm.md)
- [Used to provide the scheduling and result assembly of the download task queue.go] (eth-downloader-queue.go source code analysis.md)
- [Used to represent the peer, provide QoS and other functions peer.go] (eth-downloader-peer source code analysis.md)
- [Fast synchronization algorithm used to provide state-root synchronization of Pivot point statesync.go](eth-downloader-statesync.md)
- [Analysis of the general process of synchronization] (eth-downloader source code analysis.md)

Filter part of the source code analysis

- [Provide Bloom filter query and RPC filtering] (eth-bloombits and filter source code analysis.md)
</pre></body></html>