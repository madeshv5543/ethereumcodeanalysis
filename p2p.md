
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>P2p source code and several packages below

- discover contains [Kademlia Protocol] (references/Kademlia protocol principle introduction.pdf). It is a UDP-based p2p node discovery protocol.
- discv5 new node discovery protocol. Still a test attribute. This analysis is not covered.
- Part of the code for nat network address translation
- netutil some tools
- Simulations simulation of p2p networks. This analysis is not covered.

Source code analysis of the discover part

- [Discovered the persistent storage of the node database.go] (p2p-database.go source code analysis.md)
- [Kadellia protocol core logic tabel.go] (p2p-table.go source code analysis.md)
- [UDP protocol processing logic udp.go] (p2p-udp.go source code analysis.md)
- [Network address translation nat.go] (p2p-nat source code analysis.md)

P2p/ partial source analysis

- [Encrypted Link Processing Protocol between Nodes rlpx.go] (encrypted link between p2p-rlpx nodes.md)
- [Processing logic for picking nodes and then connecting them dail.go] (p2p-dial.go source code analysis.md)
- [Node and node connection processing and protocol processing peer.go] (p2p-peer.go source code analysis.md)
- [p2p server logic server.go] (p2p-server.go source code analysis.md)
</pre></body></html>