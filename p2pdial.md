
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Dial.go is mainly responsible for establishing part of the link in p2p. For example, find the node that establishes the link. Establish a link with the node. Find the address of the specified node through discover. And other functions.


Dial.go uses a data structure of the dilstate to store the intermediate state, which is the core data structure in the dial function.

// dialstate schedules dials and discovery lookups.
// it get's a chance to compute new tasks on every iteration
// of the main loop in Server.run.
Type dialstate struct {
maxDynDials int //Maximum number of dynamic node links
Ntab discoverTable //discoverTable is used to do node query
Netrestrict *netutil.Netlist

lookupRunning bool
Dialing map[discover.NodeID]connFlag //The node being linked
lookupBuf [] *discover.Node // current discovery lookup results // current discovery query results
randomNodes []*discover.Node // filled from Table // nodes randomly queried from discoverTable
Static map[discover.NodeID]*dialTask ​​// Static node.
Hist *dialHistory

Start time.Time // time when the dialer was first used
Bootnodes []*discover.Node // default dials when there are no peers // This is a built-in node. If no other nodes are found. Then use to link these nodes.
}

The process of creating dailstate.

Func newDialState(static []*discover.Node, bootnodes []*discover.Node, ntab discoverTable, maxdyn int, netrestrict *netutil.Netlist) *dialstate {
s := &amp;dialstate{
maxDynDials: maxdyn,
Ntab: ntab,
Netrestrict: netrestrict,
Static: make(map[discover.NodeID]*dialTask),
Dialing: make(map[discover.NodeID]connFlag),
Bootnodes: make([]*discover.Node, len(bootnodes)),
randomNodes: make([]*discover.Node, maxdyn/2),
Hist: new(dialHistory),
}
Copy(s.bootnodes, bootnodes)
For _, n := range static {
s.addStatic(n)
}
Return s
}

The most important method of dail is the newTasks method. This method is used to generate a task. A task is an interface. There is a Do method.

Type task interface {
Do(*Server)
}

Func (s *dialstate) newTasks(nRunning int, peers map[discover.NodeID]*Peer, now time.Time) []task {
If s.start == (time.Time{}) {
S.start = now
}

Var newtasks []task
//addDial is an internal method that first checks the node via checkDial. Then set the state and finally add the node to the newtasks queue.
addDial := func(flag connFlag, n *discover.Node) bool {
If err := s.checkDial(n, peers); err != nil {
log.Trace("Skipping dial candidate", "id", n.ID, "addr", &amp;net.TCPAddr{IP: n.IP, Port: int(n.TCP)}, "err", err)
Return false
}
S.dialing[n.ID] = flag
Newtasks = append(newtasks, &amp;dialTask{flags: flag, dest: n})
Return true
}

// Compute number of dynamic dials necessary at this point.
needDynDials := s.maxDynDials
/ / First determine the type of connection that has been established. If it is a dynamic type. Then you need to establish a reduction in the number of dynamic links.
For _, p := range peers {
If p.rw.is(dynDialedConn) {
needDynDials--
}
}
// Then judge the link being created. If it is a dynamic type. Then you need to establish a reduction in the number of dynamic links.
For _, flag := range s.dialing {
If flag&amp;dynDialedConn != 0 {
needDynDials--
}
}

// Expire the dial history on every invocation.
S.hist.expire(now)

// Create dials for static nodes if they are not connected.
//View all static types. If you can then create a link.
For id, t := range s.static {
Err := s.checkDial(t.dest, peers)
Switch err {
Case errNotWhitelisted, errSelf:
log.Warn("Removing static dial candidate", "id", t.dest.ID, "addr", &amp;net.TCPAddr{IP: t.dest.IP, Port: int(t.dest.TCP)}, " Err", err)
Delete(s.static, t.dest.ID)
Case nil:
S.dialing[id] = t.flags
Newtasks = append(newtasks, t)
}
}
// If we don't have any peers whatsoever, try to dial a random bootnode. This
// scenario is useful for the testnet (and private networks) where the discovery
// table might be full of mostly bad peers, making it hard to find good ones.
//If there are no links yet. And no link was created within 20 seconds (fallbackInterval). Then use bootnode to create a link.
If len(peers) == 0 &amp;&amp; len(s.bootnodes) &gt; 0 &amp;&amp; needDynDials &gt; 0 &amp;&amp; now.Sub(s.start) &gt; fallbackInterval {
Bootnode := s.bootnodes[0]
S.bootnodes = append(s.bootnodes[:0], s.bootnodes[1:]...)
S.bootnodes = append(s.bootnodes, bootnode)

If addDial(dynDialedConn, bootnode) {
needDynDials--
}
}
// Use random nodes from the table for half of the necessary
// dynamic dials.
// Otherwise create a link using a random node of 1/2.
randomCandidates := needDynDials / 2
If randomCandidates &gt; 0 {
n := s.ntab.ReadRandomNodes(s.randomNodes)
For i := 0; i &lt; randomCandidates &amp;&amp; i &lt; n; i++ {
If addDial(dynDialedConn, s.randomNodes[i]) {
needDynDials--
}
}
}
// Create dynamic dials from random lookup results, removing tried
// items from the result buffer.
i := 0
For ; i &lt; len(s.lookupBuf) &amp;&amp; needDynDials &gt; 0; i++ {
If addDial(dynDialedConn, s.lookupBuf[i]) {
needDynDials--
}
}
s.lookupBuf = s.lookupBuf[:copy(s.lookupBuf, s.lookupBuf[i:])]
// Launch a discovery lookup if more candidates are needed.
// If you don't, you can't create enough dynamic links. Then create a discoveryTask to find other nodes on the network. Put into lookupBuf
If len(s.lookupBuf) &lt; needDynDials &amp;&amp; !s.lookupRunning {
s.lookupRunning = true
Newtasks = append(newtasks, &amp;discoverTask{})
}

// Launch a timer to wait for the next node to expire if all
//same have been tried and no task is currently active.
// This should prevent cases where the dialer logic is not ticked
// because there are no pending events.
// If there are currently no tasks to do, create a sleep task to return.
If nRunning == 0 &amp;&amp; len(newtasks) == 0 &amp;&amp; s.hist.Len() &gt; 0 {
t := &amp;waitExpireTask{s.hist.min().exp.Sub(now)}
Newtasks = append(newtasks, t)
}
Return newtasks
}


The checkDial method is used to check if the task needs to create a link.

Func (s *dialstate) checkDial(n *discover.Node, peers map[discover.NodeID]*Peer) error {
_, dialing := s.dialing[n.ID]
Switch {
Case dialing: // is creating
Return errAlreadyDialing
Case peers[n.ID] != nil: // already linked
Return errAlreadyConnected
Case s.ntab != nil &amp;&amp; n.ID == s.ntab.Self().ID: //The object created is not itself
Return errSelf
Case s.netrestrict != nil &amp;&amp; !s.netrestrict.Contains(n.IP): //Network restrictions. The IP address of the other party is not in the white list.
Return errNotWhitelisted
Case s.hist.contains(n.ID): // This ID was linked.
Return errRecentlyDialed
}
Return nil
}

taskDone method. This method will be called after the task is completed. View the type of task. If it is a link task, then add it to hist. And removed from the queue being linked. If it is a query task. Put the query's record in the lookupBuf.

Func (s *dialstate) taskDone(t task, now time.Time) {
Switch t := t.(type) {
Case *dialTask:
S.hist.add(t.dest.ID, now.Add(dialHistoryExpiration))
Delete(s.dialing, t.dest.ID)
Case *discoverTask:
s.lookupRunning = false
s.lookupBuf = append(s.lookupBuf, t.results...)
}
}



The dialTask.Do method has different Do methods for different tasks. dailTask ​​is primarily responsible for establishing links. If t.dest has no ip address. Then try to query the ip address through resolve. Then call the dial method to create the link. For static nodes. If it fails for the first time, it will try to resolve the static node again. Then try dial again (because the static node's ip is configured. If the static node's ip address changes, then we try to resolve the new address of the static node and then call the link.)

Func (t *dialTask) Do(srv *Server) {
If t.dest.Incomplete() {
If !t.resolve(srv) {
Return
}
}
Success := t.dial(srv, t.dest)
// Try resolving the ID of static nodes if dialing failed.
If !success &amp;&amp; t.flags&amp;staticDialedConn != 0 {
If t.resolve(srv) {
T.dial(srv, t.dest)
}
}
}

The resolve method. This method mainly calls the Resolve method of the discovery network. If it fails, then try again after the timeout.

// resolve attempts to find the current endpoint for the destination
// using discovery.
//
// Resolve operations are throttled with backoff to avoid flooding the
// discovery network with useless queries for nodes that don't exist.
// The backoff delay resets when the node is found.
Func (t *dialTask) resolve(srv *Server) bool {
If srv.ntab == nil {
log.Debug("Can't resolve node", "id", t.dest.ID, "err", "discovery is disabled")
Return false
}
If t.resolveDelay == 0 {
t.resolveDelay = initialResolveDelay
}
If time.Since(t.lastResolved) &lt; t.resolveDelay {
Return false
}
Resolved := srv.ntab.Resolve(t.dest.ID)
t.lastResolved = time.Now()
If resolved == nil {
t.resolveDelay *= 2
If t.resolveDelay &gt; maxResolveDelay {
t.resolveDelay = maxResolveDelay
}
log.Debug("Resolving node failed", "id", t.dest.ID, "newdelay", t.resolveDelay)
Return false
}
// The node was found.
t.resolveDelay = initialResolveDelay
T.dest = resolved
log.Debug("Resolved node", "id", t.dest.ID, "addr", &amp;net.TCPAddr{IP: t.dest.IP, Port: int(t.dest.TCP)})
Return true
}


The dial method, this method performs the actual network connection operation. This is done mainly by the srv.SetupConn method, and then analyzed later when analyzing Server.go.

// dial performs the actual connection attempt.
Func (t *dialTask) dial(srv *Server, dest *discover.Node) bool {
Fd, err := srv.Dialer.Dial(dest)
If err != nil {
log.Trace("Dial error", "task", t, "err", err)
Return false
}
Mfd := newMeteredConn(fd, false)
srv.SetupConn(mfd, t.flags, dest)
Return true
}

The Do method of discoverTask and waitExpireTask,

Func (t *discoverTask) Do(srv *Server) {
// newTasks generates a lookup task whenever dynamic dials are
// necessary. Lookups need to take some time, otherwise the
// event loop spins too fast.
Next := srv.lastLookup.Add(lookupInterval)
If now := time.Now(); now.Before(next) {
time.Sleep(next.Sub(now))
}
srv.lastLookup = time.Now()
Var target discover.NodeID
rand.Read(target[:])
T.results = srv.ntab.Lookup(target)
}


Func (t waitExpireTask) Do(*Server) {
time.Sleep(t.Duration)
}</pre></body></html>