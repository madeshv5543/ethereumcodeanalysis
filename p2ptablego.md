
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Table.go mainly implements the Kademlia protocol of p2p.

### Kademlia protocol introduction (recommended to read the pdf document in references)
The Kademlia Agreement (hereinafter referred to as Kad) is PetarP. Maymounkov and David Mazieres of New York University.
A study published in 2002 "Kademlia: A peerto -peer information system based on
The XOR metric.
Simply put, Kad is a distributed hash table (DHT) technology, but compared to other DHT implementation techniques, such as
Chord, CAN, Pastry, etc., Kad established a unique basis based on the XOR algorithm.
The new DHT topology greatly improves routing query speed compared to other algorithms.


### table structure and fields

Const (
Alpha = 3 // Kademlia concurrency factor
bucketSize = 16 // Kademlia bucket size
hashBits = len(common.Hash{}) * 8
nBuckets = hashBits + 1 // Number of buckets

maxBondingPingPongs = 16
maxFindnodeFailures = 5

autoRefreshInterval = 1 * time.Hour
seedCount = 30
seedMaxAge = 5 * 24 * time.Hour
)

Type Table struct {
Mutex sync.Mutex // protects buckets, their content, and nursery
Buckets [nBuckets]*bucket // index of known nodes by distance
Nursery []*Node // bootstrap nodes
Db *nodeDB // database of known nodes

refreshReq chan chan struct{}
closeReq chan struct{}
Closed chan struct{}

Bondmu sync.Mutex
Bonding map[NodeID]*bondproc
Bondslots chan struct{} // limits total number of active bonding processes

nodeAddedHook func(*Node) // for testing

Net transport
Self *Node // metadata of the local node
}


### Initialization


Func newTable(t transport, ourID NodeID, ourAddr *net.UDPAddr, nodeDBPath string) (*Table, error) {
// If no node database was given, use an in-memory one
//This is introduced in the previous database.go. Open leveldb. If path is empty. Then open a memory based db
Db, err := newNodeDB(nodeDBPath, Version, ourID)
If err != nil {
Return nil, err
}
Tab := &amp;Table{
Net: t,
Db: db,
Self: NewNode(ourID, ourAddr.IP, uint16(ourAddr.Port), uint16(ourAddr.Port)),
Bonding: make(map[NodeID]*bondproc),
Bondslots: make(chan struct{}, maxBondingPingPongs),
refreshReq: make(chan chan struct{}),
closeReq: make(chan struct{}),
Closed: make(chan struct{}),
}
For i := 0; i &lt; cap(tab.bondslots); i++ {
Tab.bondslots &lt;- struct{}{}
}
For i := range tab.buckets {
Tab.buckets[i] = new(bucket)
}
Go tab.refreshLoop()
Return tab, nil
}

The above initialization starts a goroutine refreshLoop(). This function mainly performs the following work.

1. Perform a refresh every hour (autoRefreshInterval)
2. If a refreshReq request is received. Then do the refresh work.
3. If a close message is received. Then close it.

So the main job of the function is to start the refresh work. doRefresh


// refreshLoop schedules doRefresh runs and coordinates shutdown.
Func (tab *Table) refreshLoop() {
Var (
Timer = time.NewTicker(autoRefreshInterval)
Waiting []chan struct{} // accumulates waiting callers while doRefresh runs
Done chan struct{} // where doRefresh reports completion
)
Loop:
For {
Select {
Case &lt;-timer.C:
If done == nil {
Done = make(chan struct{})
Go tab.doRefresh(done)
}
Case req := &lt;-tab.refreshReq:
Waiting = append(waiting, req)
If done == nil {
Done = make(chan struct{})
Go tab.doRefresh(done)
}
Case &lt;-done:
For _, ch := range waiting {
Close(ch)
}
Waiting = nil
Done = nil
Case &lt;-tab.closeReq:
Break loop
}
}

If tab.net != nil {
Tab.net.close()
}
If done != nil {
&lt;-done
}
For _, ch := range waiting {
Close(ch)
}
Tab.db.close()
Close(tab.closed)
}


doRefresh function

// doRefresh performs a lookup for a random target to keep buckets
// full. seed nodes are inserted if the table is empty (initial
// bootstrap or discarded faulty peers).
// doRefresh randomly finds a target to keep the buckets full. If the table is empty, the seed node will be inserted. (such as the initial start or after deleting the wrong node)
Func (tab *Table) doRefresh(done chan struct{}) {
Defer close(done)

// The Kademlia paper specifies that the bucket refresh should
// perform a lookup in the least recently used bucket. We cannot
// adhere to this because the findnode target is a 512bit value
// (not hash-sized) and it is not easily possible to generate a
// sha3 preimage that falls into a chosen bucket.
// We perform a lookup with a random target instead.
//This time I didn’t understand it.
Var target NodeID
rand.Read(target[:])
Result := tab.lookup(target, false) //lookup is to find the k nodes closest to the target
If len(result) &gt; 0 { //If the result is not 0, the table is not empty, then return directly.
Return
}

// The table is empty. Load nodes from the database and insert
//the. This should yield a few previously seen nodes that are
// (hopefully) still alive.
The //querySeeds function is described in the database.go section, which randomly finds available seed nodes from the database.
//The database is blank at the very beginning. That is, the seeds returned at the beginning are empty.
Seeds := tab.db.querySeeds(seedCount, seedMaxAge)
/ / Call the bondall function. Will try to contact these nodes and insert them into the table.
//tab.nursery is the seed node specified on the command line.
//At the beginning of the startup. The value of tab.nursery is built into the code. This is worthwhile.
//C:\GOPATH\src\github.com\ethereum\go-ethereum\mobile\params.go
//There is a dead value written here. This value is written by the SetFallbackNodes method. This method will be analyzed later.
//This will be a two-way pingpong exchange. Then store the results in the database.
Seeds = tab.bondall(append(seeds, tab.nursery...))

If len(seeds) == 0 { //No seed nodes are found, and may need to wait for the next refresh.
log.Debug("No discv4 seed nodes found")
}
For _, n := range seeds {
Age := log.Lazy{Fn: func() time.Duration { return time.Since(tab.db.lastPong(n.ID)) }}
log.Trace("Found seed node in database", "id", n.ID, "addr", n.addr(), "age", age)
}
Tab.mutex.Lock()
/ / This method adds all the seeded seed to the bucket (provided the bucket is not full)
Tab.stuff(seeds)
tab.mutex.Unlock()

// Finally, do a self lookup to fill up the buckets.
Tab.lookup(tab.self.ID, false) // Has a seed node. Then find yourself to fill the buckets.
}

The bondall method, this method is a multi-threaded call to the bond method.

// bondall bonds with all given nodes concurrently and returns
// those nodes for which bonding has been probably succeeded.
Func (tab *Table) bondall(nodes []*Node) (result []*Node) {
Rc := make(chan *Node, len(nodes))
For i := range nodes {
Go func(n *Node) {
Nn, _ := tab.bond(false, n.ID, n.addr(), uint16(n.TCP))
Rc &lt;- nn
}(nodes[i])
}
For range nodes {
If n := &lt;-rc; n != nil {
Result = append(result, n)
}
}
Return result
}

Bond method. Remember in udp.go. When we receive a ping method, it is also possible to call this method.


// bond ensures the local node has a bond with the given remote node.
// It also attempts to insert the node into the table if precipitation succeeds.
// The caller must not hold tab.mutex.
// The bond ensures that the local node has a binding to the given remote node. (remote ID and remote IP).
// If the binding is successful, it will also try to insert the node into the table. The caller must hold the tab.mutex lock
// A bond is must be established before sending findnode requests.
// Both must have completed a ping/pong exchange for a bond to
// exist. The total number of active bonding processes is limited in
// order to restrain network use.
// A binding must be established before sending a findnode request. Both parties must complete a two-way ping/pong process in order to complete a bond.
// To save network resources. The total number of simultaneous bonding processes is limited.
// bond is meant to operate idempotently in that bonding with a remote
// node which still remembers a previously established bond will work.
// The remote node will simply not send a ping back, causing waitping
// to time out.
// bond is an idempotent operation, and can be done with a remote node that still remembers the previous bond. The remote node will simply not send a ping. Waiting for the waitping timeout.
// If pinged is true, the remote node has just pinged us and one half
// of the process can be skipped.
// If pinged is true. Then the remote node has sent us a ping message. Half of the process can be skipped.
Func (tab *Table) bond(pinged bool, id NodeID, addr *net.UDPAddr, tcpPort uint16) (*Node, error) {
If id == tab.self.ID {
Return nil, errors.New("is self")
}
// Retrieve a previously known node and any recent findnode failures
Node, fails := tab.db.node(id), 0
If node != nil {
Fails = tab.db.findFails(id)
}
// If the node is unknown (non-bonded) or failed (remotely unknown), bond from scratch
Var result error
Age := time.Since(tab.db.lastPong(id))
If node == nil || fails &gt; 0 || age &gt; nodeDBNodeExpiration {
//If the database does not have this node. Or the number of errors is greater than 0 or the node times out.
log.Trace("Starting bonding ping/pong", "id", id, "known", node != nil, "failcount", fails, "age", age)

tab.bondmu.Lock()
w := tab.bonding[id]
If w != nil {
// Wait for an existing bonding process to complete.
tab.bondmu.Unlock()
&lt;-w.done
} else {
// Register a new bonding process.
w = &amp;bondproc{done: make(chan struct{})}
Tab.bonding[id] = w
tab.bondmu.Unlock()
// Do the ping/pong. The result goes into w.
Tab.pingpong(w, pinged, id, addr, tcpPort)
// Unregister the process after it's done.
tab.bondmu.Lock()
Delete(tab.bonding, id)
tab.bondmu.Unlock()
}
// Retrieve the bonding results
Result = w.err
If result == nil {
Node = w.n
}
}
If node != nil {
// Add the node to the table even if the bonding ping/pong
// fails. It will be relaced quickly if it continues to be
// unresponsive.
//This method is more important. If the corresponding bucket has space, the buckets will be inserted directly. If the buckets are full. The ping operation will be used to test the nodes in the bucket to try to make room.
Tab.add(node)
tab.db.updateFindFails(id, 0)
}
Return node, result
}

Pingpong method

Func (tab *Table) pingpong(w *bondproc, pinged bool, id NodeID, addr *net.UDPAddr, tcpPort uint16) {
// Request a bonding slot to limit network usage
&lt;-tab.bondslots
Defer func() { tab.bondslots &lt;- struct{}{} }()

// Ping the remote side and wait for a pong.
// Ping the remote node. And wait for a pong message
If w.err = tab.ping(id, addr); w.err != nil {
Close(w.done)
Return
}
//This is set to true when udp receives a ping message. At this time we have received the ping message from the other party.
// Then we will wait for the ping message differently. Otherwise, you need to wait for the ping message sent by the other party (we initiate the ping message actively).
If !pinged {
// Give the remote node a chance to ping us before we start
// sending findnode requests. If they still remember us,
//waitping will simply time out.
Tab.net.waitping(id)
}
// Bonding succeeded, update the node database.
// Complete the bond process. Insert the node into the database. The database operation is done here. The operation of the bucket is done in tab.add. Buckets are memory operations. The database is a persistent seeds node. Used to speed up the startup process.
W.n = NewNode(id, addr.IP, uint16(addr.Port), tcpPort)
tab.db.updateNode(w.n)
Close(w.done)
}

Tab.add method

// add attempts to add the given node its corresponding bucket. If the
// bucket has space available, adding the node succeeds immediately.
// Otherwise, the node is added if the least recently active node in
// the bucket does not respond to a ping packet.
// add attempts to insert the given node into the corresponding bucket. If the bucket has space, insert it directly. Otherwise, if the most active node in the bucket does not respond to the ping, then we replace it with this node.
// The caller must not hold tab.mutex.
Func (tab *Table) add(new *Node) {
b := tab.buckets[logdist(tab.self.sha, new.sha)]
Tab.mutex.Lock()
Defer tab.mutex.Unlock()
If b.bump(new) { //If the node exists. Then update its value. Then quit.
Return
}
Var oldest *Node
If len(b.entries) == bucketSize {
Oldest = b.entries[bucketSize-1]
If oldest.contested {
// The node is already being replaced, don't attempt
// to replace it.
// If another goroutine is testing this node. Then cancel the replacement and exit directly.
// Because the ping time is longer. So this time is not locked. This state is used to identify this situation.
Return
}
Oldest.contested = true
// Let go of the mutex so other goroutines can access
// the table while we ping the least recently active node.
tab.mutex.Unlock()
Err := tab.ping(oldest.ID, oldest.addr())
Tab.mutex.Lock()
Oldest.contested = false
If err == nil {
// The node responded, don't replace it.
Return
}
}
Added := b.replace(new, oldest)
If added &amp;&amp; tab.nodeAddedHook != nil {
tab.nodeAddedHook(new)
}
}



The stuff method is relatively simple. Find the bucket that the corresponding node should insert. If the bucket is not full, then insert the bucket. Otherwise do nothing. Need to say something about the logdist () method. This method XORs the two values ​​and returns the highest-level subscript. For example logdist(101,010) = 3 logdist(100, 100) = 0 logdist(100,110) = 2

// stuff adds nodes the table to the end of their corresponding bucket
// if the bucket is not full. The caller must hold tab.mutex.
Func (tab *Table) stuff(nodes []*Node) {
Outer:
For _, n := range nodes {
If n.ID == tab.self.ID {
Continue // don't add self
}
Bucket := tab.buckets[logdist(tab.self.sha, n.sha)]
For i := range bucket.entries {
If bucket.entries[i].ID == n.ID {
Continue outer // already in bucket
}
}
If len(bucket.entries) &lt; bucketSize {
Bucket.entries = append(bucket.entries, n)
If tab.nodeAddedHook != nil {
tab.nodeAddedHook(n)
}
}
}
}


Look at the previous Lookup function. This function is used to query the information of a specified node. This function first gets all the 16 nodes closest to this node from the local. Then send a request for findnode to all nodes. Then the bond definition is applied to the returned definition. Then return all the nodes.



Func (tab *Table) lookup(targetID NodeID, refreshIfEmpty bool) []*Node {
Var (
Target = crypto.Keccak256Hash(targetID[:])
Asked = make(map[NodeID]bool)
Seen = make(map[NodeID]bool)
Reply = make(chan []*Node, alpha)
pendingQueries = 0
Result *nodesByDistance
)
// don't query further if we hit ourself.
// unlikely to happen often in practice.
Asked[tab.self.ID] = true
Will not ask ourselves
For {
Tab.mutex.Lock()
// generate initial result set
Result = tab.closest(target, bucketSize)
/ / Find the nearest 16 nodes with the target
tab.mutex.Unlock()
If len(result.entries) &gt; 0 || !refreshIfEmpty {
Break
}
// The result set is empty, all nodes were dropped, refresh.
// We actually wait for the refresh to complete here. The very
// first query will hit this case and run the bootstrapping
// logic.
&lt;-tab.refresh()
refreshIfEmpty = false
}

For {
// ask the alpha closest nodes that we haven't lecture yet
// There will be concurrent queries, each time 3 goroutine concurrency (controlled by the pendingQueries parameter)
// Each iteration will query the three nodes in the result that are closest to the target.
For i := 0; i &lt; len(result.entries) &amp;&amp; pendingQueries &lt; alpha; i++ {
n := result.entries[i]
If !asked[n.ID] { // If there is no query // because this result.entries will be looped many times. So use this variable to control which ones have been processed.
Asked[n.ID] = true
pendingQueries++
Go func() {
// Find potential neighbors to bond with
r, err := tab.net.findnode(n.ID, n.addr(), targetID)
If err != nil {
// Bump the failure counter to detect and evacuate non-bonded entries
Fails := tab.db.findFails(n.ID) + 1
tab.db.updateFindFails(n.ID, fails)
log.Trace("Bumping findnode failure counter", "id", n.ID, "failcount", fails)

If fails &gt;= maxFindnodeFailures {
log.Trace("Too many findnode failures, dropping", "id", n.ID, "failcount", fails)
Tab.delete(n)
}
}
Reply &lt;- tab.bondall(r)
}()
}
}
If pendingQueries == 0 {
// we have asked all closest nodes, stop the search
Break
}
// wait for the next reply
For _, n := range &lt;-reply {
If n != nil &amp;&amp; !seen[n.ID] { //Because different distant nodes may return the same node. All use sheen[] to do the weighting.
Seen[n.ID] = true
/ / This place needs to pay attention to is that the result of the search will be added to the result queue. In other words, this is a process of loop lookup, as long as the result is constantly adding new nodes. This loop will not terminate.
Result.push(n, bucketSize)
}
}
pendingQueries--
}
Return result.entries
}

// closest returns the n nodes in the table that are closest to the
// given id. The caller must hold tab.mutex.
Func (tab *Table) closest(target common.Hash, nresults int) *nodesByDistance {
// This is a very wasteful way to find the closest nodes but
// obviously correct. I believe that tree-based buckets would make
// this easier to implement efficiently.
Close := &amp;nodesByDistance{target: target}
For _, b := range tab.buckets {
For _, n := range b.entries {
Close.push(n, nresults)
}
}
Return close
}

The result.push method, which sorts the distances of all nodes based on the target. The insertion order of the new nodes is determined in a near-to-far manner. (The queue will contain up to 16 elements). This will cause the elements in the queue to be closer to the target. Relatively far away will be kicked out of the queue.

// nodesByDistance is a list of nodes, ordered by
// distance to target.
Type nodesByDistance struct {
Entries []*Node
Target common.Hash
}

// push adds the given node to the list, keeping the total size below maxElems.
Func (h *nodesByDistance) push(n *Node, maxElems int) {
Ix := sort.Search(len(h.entries), func(i int) bool {
Return distcmp(h.target, h.entries[i].sha, n.sha) &gt; 0
})
If len(h.entries) &lt; maxElems {
H.entries = append(h.entries, n)
}
If ix == len(h.entries) {
// farther away than all nodes we already have.
// if there was room for it, the node is now the last element.
} else {
// slide existing entries down to make room
// this will overwrite the entry we just appended.
Copy(h.entries[ix+1:], h.entries[ix:])
H.entries[ix] = n
}
}


### table.go Some methods of exporting
Resolve method and Lookup method

// Resolve searches for a specific node with the given ID.
// It returns nil if the node could not be found.
//Resolve method is used to get a node with the specified ID. If the node is local. Then return to the local node. Otherwise execute
//Lookup queries once on the network. If the query is to a node. Then return. Otherwise return nil
Func (tab *Table) Resolve(targetID NodeID) *Node {
// If the node is present in the local table, no
// network interaction is required.
Hash := crypto.Keccak256Hash(targetID[:])
Tab.mutex.Lock()
Cl := tab.closest(hash, 1)
tab.mutex.Unlock()
If len(cl.entries) &gt; 0 &amp;&amp; cl.entries[0].ID == targetID {
Return cl.entries[0]
}
// Otherwise, do a network lookup.
Result := tab.Lookup(targetID)
For _, n := range result {
If n.ID == targetID {
Return n
}
}
Return nil
}

// Lookup performs a network search for nodes close
// to the given target. It approaches the target by querying
// nodes that are closer to it on each iteration.
// The given target does not need to be an actual node
// identifier.
Func (tab *Table) Lookup(targetID NodeID) []*Node {
Return tab.lookup(targetID, true)
}

SetFallbackNodes method, this method sets the initial contact node. The table is empty and there are no known nodes in the database. These nodes can help connect to the network.

// SetFallbackNodes sets the initial points of contact. These nodes
// are used to connect to the network if the table is empty and there
// are no known nodes in the database.
Func (tab *Table) SetFallbackNodes(nodes []*Node) error {
For _, n := range nodes {
If err := n.validateComplete(); err != nil {
Return fmt.Errorf("bad bootstrap/fallback node %q (%v)", n, err)
}
}
Tab.mutex.Lock()
Tab.nursery = make([]*Node, 0, len(nodes))
For _, n := range nodes {
Cpy := *n
// Recompute cpy.sha because the node might not have been
//created by NewNode or ParseNode.
Cpy.sha = crypto.Keccak256Hash(n.ID[:])
Tab.nursery = append(tab.nursery, &amp;cpy)
}
tab.mutex.Unlock()
Tab.refresh()
Return nil
}


### to sum up

In this way, the Kademlia protocol of the p2p network is over. Basically, it is implemented according to the paper. Udp for network communication. The database stores the linked nodes. Table implements the core of Kademlia. Find the node based on the XOR distance. Process of discovery and update of nodes.</pre></body></html>