
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>The p2p package implements a generic p2p network protocol. It includes the functions of p2p such as node lookup, node state maintenance, and node connection establishment. The p2p package implements the generic p2p protocol. A specific protocol (such as the eth protocol. whisper protocol. swarm protocol) is encapsulated into a specific interface to inject p2p packets. Therefore, p2p does not contain the implementation of a specific protocol. Only completed what the p2p network should do.


## discover / discv5 node discovery
The package currently in use is discover. Discv5 is a recently developed feature, or it is experimental, basically some optimization of the discovery package. Here we only analyze the code of the discovery for the time being. A basic introduction to the functions that are completed.


### database.go
As the name suggests, this file mainly implements the persistence of nodes, because the node discovery and maintenance of p2p network nodes are relatively time-consuming. In order to repeatedly start, the previous work can be inherited to avoid rediscovery every time. So persistent work is a must.

Before we analyzed the code of ethdb and the code of trie, the persistence of trie used leveldb. Leveldb is also used here. However, the leveldb instance of p2p is not the same as the leveldb instance of the main blockchain.

newNodeDB, open a memory-based database based on the parameter path, or a file-based database.

// newNodeDB creates a new node database for storing and retrieving infos about
//known peers in the network. If no path is given, an in-memory, temporary
// database is constructed.
Func newNodeDB(path string, version int, self NodeID) (*nodeDB, error) {
If path == "" {
Return newMemoryNodeDB(self)
}
Return newPersistentNodeDB(path, version, self)
}
// newMemoryNodeDB creates a new in-memory node database without a persistent
// backend.
Func newMemoryNodeDB(self NodeID) (*nodeDB, error) {
Db, err := leveldb.Open(storage.NewMemStorage(), nil)
If err != nil {
Return nil, err
}
Return &amp;nodeDB{
Lvl: db,
Self: self,
Quit: make(chan struct{}),
}, nil
}

// newPersistentNodeDB creates/opens a leveldb backed persistent node database,
// also flushing its contents in case of a version mismatch.
Func newPersistentNodeDB(path string, version int, self NodeID) (*nodeDB, error) {
Opts := &amp;opt.Options{OpenFilesCacheCapacity: 5}
Db, err := leveldb.OpenFile(path, opts)
If _, iscorrupted := err.(*errors.ErrCorrupted); iscorrupted {
Db, err = leveldb.RecoverFile(path, nil)
}
If err != nil {
Return nil, err
}
// The nodes contained in the cache correspond to a certain protocol version.
// Flush all nodes if the version doesn't match.
currentVer := make([]byte, binary.MaxVarintLen64)
currentVer = currentVer[:binary.PutVarint(currentVer, int64(version))]
Blob, err := db.Get(nodeDBVersionKey, nil)
Switch err {
Case leveldb.ErrNotFound:
// Version not found (i.e. empty cache), insert it
If err := db.Put(nodeDBVersionKey, currentVer, nil); err != nil {
db.Close()
Return nil, err
}
Case nil:
// Version present, flush if different
/ / version is different, first delete all database files, re-create one.
If !bytes.Equal(blob, currentVer) {
db.Close()
If err = os.RemoveAll(path); err != nil {
Return nil, err
}
Return newPersistentNodeDB(path, version, self)
}
}
Return &amp;nodeDB{
Lvl: db,
Self: self,
Quit: make(chan struct{}),
}, nil
}


Node storage, query and delete

// node retrieves a node with a given id from the database.
Func (db *nodeDB) node(id NodeID) *Node {
Blob, err := db.lvl.Get(makeKey(id, nodeDBDiscoverRoot), nil)
If err != nil {
Return nil
}
Node := new(Node)
If err := rlp.DecodeBytes(blob, node); err != nil {
log.Error("Failed to decode node RLP", "err", err)
Return nil
}
Node.sha = crypto.Keccak256Hash(node.ID[:])
Return node
}

// updateNode inserts - potentially overwriting - a node into the peer database.
Func (db *nodeDB) updateNode(node ​​*Node) error {
Blob, err := rlp.EncodeToBytes(node)
If err != nil {
Return err
}
Return db.lvl.Put(makeKey(node.ID, nodeDBDiscoverRoot), blob, nil)
}

// deleteNode deletes all information/keys associated with a node.
Func (db *nodeDB) deleteNode(id NodeID) error {
Deleter := db.lvl.NewIterator(util.BytesPrefix(makeKey(id, "")), nil)
For deleter.Next() {
If err := db.lvl.Delete(deleter.Key(), nil); err != nil {
Return err
}
}
Return nil
}

Node structure

Type Node struct {
IP net.IP // len 4 for IPv4 or 16 for IPv6
UDP, TCP uint16 // port numbers
ID NodeID // the node's public key
// This is a cached copy of sha3(ID) which is used for node
// distance calculations. This is part of Node in order to make it
// possible to write tests that need a node at a certain distance.
// In those tests, the content of sha will not correspond
// with ID.
Sha common.Hash
// whether this node is currently being pinged in order to replace
// it in a bucket
Contested bool
}

Node timeout processing


// ensureExpirer is a small helper method ensuring that the data expiration
// mechanism is running. If the expiration goroutine is already running, this
// method simply returns.
// The ensureExpirer method is used to ensure that the expirer method is running. If the expirer is already running, then this method returns directly.
// The purpose of this method setting is to start the data timeout drop after the network is successfully started (in case some potentially useful seed nodes are discarded).
// The goal is to start the data evacuation only after the network successfully
// bootstrapped itself (to prevent dumping potentially useful seed nodes).
// it would require significant overhead to exactly trace the first successful
// convergence, it's simpler to "ensure" the correct state when an appropriate
// condition occurs (i.e. a successful bonding), and discard further events.
Func (db *nodeDB) ensureExpirer() {
db.runner.Do(func() { go db.expirer() })
}

// expirer should be started in a go routine, and is responsible for looping ad
// infinitum and dropping stale data from the database.
Func (db *nodeDB) expirer() {
Tick ​​:= time.Tick(nodeDBCleanupCycle)
For {
Select {
Case &lt;-tick:
If err := db.expireNodes(); err != nil {
log.Error("Failed to expire nodedb items", "err", err)
}

Case &lt;-db.quit:
Return
}
}
}

// expireNodes iterates over the database and deletes all nodes that have not
// Been seen (i.e. received a pong from) for some allotted time.
// This method iterates through all the nodes. If a node receives a message that exceeds the specified value, it deletes the node.
Func (db *nodeDB) expireNodes() error {
Threshold := time.Now().Add(-nodeDBNodeExpiration)

// Find discovered nodes that are older than the allowance
It := db.lvl.NewIterator(nil, nil)
Defer it.Release()

For it.Next() {
// Skip the item if not a discovery node
Id, field := splitKey(it.Key())
If field != nodeDBDiscoverRoot {
Continue
}
// Skip the node if not expired yet (and not self)
If !bytes.Equal(id[:], db.self[:]) {
If seen := db.lastPong(id); seen.After(threshold) {
Continue
}
}
// Otherwise delete all associated information
db.deleteNode(id)
}
Return nil
}


Some state update functions

// lastPing retrieves the time of the last ping packet send to a remote node,
// requesting binding.
Func (db *nodeDB) lastPing(id NodeID) time.Time {
Return time.Unix(db.fetchInt64(makeKey(id, nodeDBDiscoverPing)), 0)
}

// updateLastPing updates the last time we tried contacting a remote node.
Func (db *nodeDB) updateLastPing(id NodeID, instance time.Time) error {
Return db.storeInt64(makeKey(id, nodeDBDiscoverPing), instance.Unix())
}

// lastPong retrieves the time of the last successful contact from remote node.
Func (db *nodeDB) lastPong(id NodeID) time.Time {
Return time.Unix(db.fetchInt64(makeKey(id, nodeDBDiscoverPong)), 0)
}

// updateLastPong updates the last time a remote node successfully contacted.
Func (db *nodeDB) updateLastPong(id NodeID, instance time.Time) error {
Return db.storeInt64(makeKey(id, nodeDBDiscoverPong), instance.Unix())
}

// findFails retrieves the number of findnode failures since bonding.
Func (db *nodeDB) findFails(id NodeID) int {
Return int(db.fetchInt64(makeKey(id, nodeDBDiscoverFindFails)))
}

// updateFindFails updates the number of findnode failures since bonding.
Func (db *nodeDB) updateFindFails(id NodeID, fails int) error {
Return db.storeInt64(makeKey(id, nodeDBDiscoverFindFails), int64(fails))
}


Randomly pick the appropriate seed node from the database


// querySeeds retrieves random nodes to be used as potential seed nodes
// for bootstrapping.
Func (db *nodeDB) querySeeds(n int, maxAge time.Duration) []*Node {
Var (
Now = time.Now()
Nodes = make([]*Node, 0, n)
It = db.lvl.NewIterator(nil, nil)
Id NodeID
)
Defer it.Release()

Seek:
For seeks := 0; len(nodes) &lt; n &amp;&amp; seeks &lt; n*5; seeks++ {
// Seek to a random entry. The first byte is incremented by a
// random amount each time in order to increase the likelihood
// of hitting all existing nodes in very small databases.
Ctr := id[0]
rand.Read(id[:])
Id[0] = ctr + id[0]%16
it.Seek(makeKey(id, nodeDBDiscoverRoot))

n := nextNode(it)
If n == nil {
Id[0] = 0
Continue seek // iterator exhausted
}
If n.ID == db.self {
Continue seek
}
If now.Sub(db.lastPong(n.ID)) &gt; maxAge {
Continue seek
}
For i := range nodes {
If nodes[i].ID == n.ID {
Continue seek // duplicate
}
}
Nodes = append(nodes, n)
}
Return nodes
}

// reads the next node record from the iterator, skipping over other
// database entries.
Func nextNode(it iterator.Iterator) *Node {
For end := false; !end; end = !it.Next() {
Id, field := splitKey(it.Key())
If field != nodeDBDiscoverRoot {
Continue
}
Var n Node
If err := rlp.DecodeBytes(it.Value(), &amp;n); err != nil {
log.Warn("Failed to decode node RLP", "id", id, "err", err)
Continue
}
Return &amp;n
}
Return nil
}



</pre></body></html>