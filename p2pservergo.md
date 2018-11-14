
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Server is the main part of p2p. All previous components are assembled.

First look at the structure of the Server


// Server manages all peer connections.
Type Server struct {
// Config fields may not be modified while the server is running.
Config

// Hooks for testing. These are useful because we can inhibit
// the whole protocol stack.
newTransport func(net.Conn) transport
newPeerHook func(*Peer)

Lock sync.Mutex // protects running
Running bool

Ntab discoverTable
Listener net.Listener
ourHandshake *protoHandshake
lastLookup time.Time
DiscV5 *discv5.Network

// These are for Peers, PeerCount (and nothing else).
peerOp chan peerOpFunc
peerOpDone chan struct{}

Quit chan struct{}
Addstatic chan *discover.Node
Removestatic chan *discover.Node
Posthandshake chan *conn
Addpeer chan *conn
Delpeer chan peerDrop
loopWG sync.WaitGroup // loop, listenLoop
peerFeed event.Feed
}

// conn wraps a network connection with information
// during the two handshakes.
Type conn struct {
Fd net.Conn
Transport
Flags connFlag
Cont chan error // The run loop uses cont to signal errors to SetupConn.
Id discover.NodeID // valid after the encryption handshake
Caps []Cap // valid after the protocol handshake
Name string // valid after the protocol handshake
}

Type transport interface {
// The two handshakes.
doEncHandshake(prv *ecdsa.PrivateKey, dialDest *discover.Node) (discover.NodeID, error)
doProtoHandshake(our *protoHandshake) (*protoHandshake, error)
// The MsgReadWriter can only be used after the encryption
// handshake has completed. The code uses conn.id to track this
// by setting it to a non-nil value after the encryption handshake.
MsgReadWriter
// transports must provide Close because we use MsgPipe in some of
// the tests. Closing the actual network connection doesn't do
// anything in those tests because NsgPipe doesn't use it.
Close(err error)
}

There is no way for a newServer. The initialization work is placed in the Start() method.


// Start starts running the server.
// Servers can not be re-used after stopping.
Func (srv *Server) Start() (err error) {
srv.lock.Lock()
Defer srv.lock.Unlock()
If srv.running { // Avoid multiple starts. Srv.lock in order to avoid multi-threaded repeated startup
Return errors.New("server already running")
}
Srv.running = true
log.Info("Starting P2P networking")

//static fields
If srv.PrivateKey == nil {
Return fmt.Errorf("Server.PrivateKey must be set to a non-nil key")
}
If srv.newTransport == nil { //Note here that Transport uses newRLPX to use the network protocol in rlpx.go.
srv.newTransport = newRLPX
}
If srv.Dialer == nil { //Used TCLPDialer
srv.Dialer = TCPDialer{&amp;net.Dialer{Timeout: defaultDialTimeout}}
}
Srv.quit = make(chan struct{})
Srv.addpeer = make(chan *conn)
Srv.delpeer = make(chan peerDrop)
Srv.posthandshake = make(chan *conn)
Srv.addstatic = make(chan *discover.Node)
Srv.removestatic = make(chan *discover.Node)
srv.peerOp = make(chan peerOpFunc)
srv.peerOpDone = make(chan struct{})

// node table
If !srv.NoDiscovery { //Start the discovery network. Enable UDP listening.
Ntab, err := discover.ListenUDP(srv.PrivateKey, srv.ListenAddr, srv.NAT, srv.NodeDatabase, srv.NetRestrict)
If err != nil {
Return err
}
/ / Set the starting node. When no other nodes are found. Then connect these boot nodes. The information of these nodes is written in the configuration file.
If err := ntab.SetFallbackNodes(srv.BootstrapNodes); err != nil {
Return err
}
Srv.ntab = ntab
}

If srv.DiscoveryV5 {//This is the new node discovery protocol. It has not been used yet. There is no analysis here.
Ntab, err := discv5.ListenUDP(srv.PrivateKey, srv.DiscoveryV5Addr, srv.NAT, "", srv.NetRestrict) //srv.NodeDatabase)
If err != nil {
Return err
}
If err := ntab.SetFallbackNodes(srv.BootstrapNodesV5); err != nil {
Return err
}
srv.DiscV5 = ntab
}

dynPeers := (srv.MaxPeers + 1) / 2
If srv.NoDiscovery {
dynPeers = 0
}
//Create dialerstate.
Dialer := newDialState(srv.StaticNodes, srv.BootstrapNodes, srv.ntab, dynPeers, srv.NetRestrict)

// handshake
//Handshake of our own agreement
srv.ourHandshake = &amp;protoHandshake{Version: baseProtocolVersion, Name: srv.Name, ID: discover.PubkeyID(&amp;srv.PrivateKey.PublicKey)}
For _, p := range srv.Protocols {//Add all the protocol's Caps
srv.ourHandshake.Caps = append(srv.ourHandshake.Caps, p.cap())
}
// listen/dial
If srv.ListenAddr != "" {
/ / Start listening to the TCP port
If err := srv.startListening(); err != nil {
Return err
}
}
If srv.NoDial &amp;&amp; srv.ListenAddr == "" {
log.Warn("P2P server will be useless, neither dialing nor listening")
}

srv.loopWG.Add(1)
//Start goroutine to handle the program.
Go srv.run(dialer)
Srv.running = true
Return nil
}


Start monitoring. It can be seen that it is the TCP protocol. The listening port here is the same as the UDP port. The default is 30303

Func (srv *Server) startListening() error {
// Launch the TCP listener.
Listener, err := net.Listen("tcp", srv.ListenAddr)
If err != nil {
Return err
}
Laddr := listener.Addr().(*net.TCPAddr)
srv.ListenAddr = laddr.String()
Srv.listener = listener
srv.loopWG.Add(1)
Go srv.listenLoop()
// Map the TCP listening port if NAT is configured.
If !laddr.IP.IsLoopback() &amp;&amp; srv.NAT != nil {
srv.loopWG.Add(1)
Go func() {
nat.Map(srv.NAT, srv.quit, "tcp", laddr.Port, laddr.Port, "ethereum p2p")
srv.loopWG.Done()
}()
}
Return nil
}

listenLoop(). This is an infinite loop of goroutine. Will listen on the port and receive external requests.

// listenLoop runs in its own goroutine and accepts
// inbound connections.
Func (srv *Server) listenLoop() {
Defer srv.loopWG.Done()
log.Info("RLPx listener up", "self", srv.makeSelf(srv.listener, srv.ntab))

// This ratio acts as a semaphore limiting
// active inbound connections that are lingering pre-handshake.
// If all slots are taken, no further connections are accepted.
Tokens := maxAcceptConns
If srv.MaxPendingPeers &gt; 0 {
Tokens = srv.MaxPendingPeers
}
//Create maxAcceptConns slots. We only handle so many connections at the same time. Don't want more.
Slots := make(chan struct{}, tokens)
// Fill the slot.
For i := 0; i &lt; tokens; i++ {
Slots &lt;- struct{}{}
}

For {
// Wait for a handshake slot before accepting.
&lt;-slots

Var (
Fd net.Conn
Err error
)
For {
Fd, err = srv.listener.Accept()
If tempErr, ok := err.(tempError); ok &amp;&amp; tempErr.Temporary() {
log.Debug("Temporary read error", "err", err)
Continue
} else if err != nil {
log.Debug("Read error", "err", err)
Return
}
Break
}

// Reject connections that do not match NetRestrict.
			// whitelist. If it is not on the white list. Then close the connection.
If srv.NetRestrict != nil {
If tcp, ok := fd.RemoteAddr().(*net.TCPAddr); ok &amp;&amp; !srv.NetRestrict.Contains(tcp.IP) {
log.Debug("Rejected conn (not whitelisted in NetRestrict)", "addr", fd.RemoteAddr())
fd.Close()
Slots &lt;- struct{}{}
Continue
}
}

Fd = newMeteredConn(fd, true)
log.Trace("Accepted connection", "addr", fd.RemoteAddr())

// Spawn the handler. It will give the slot back when the connection
// has been established.
Go func() {
//It seems that as long as the connection is established. The slot will be returned. SetupConn this function we remember to call again in dialTask.Do, this function is mainly a few handshaking to perform the connection.
srv.SetupConn(fd, inboundConn, nil)
Slots &lt;- struct{}{}
}()
}
}

SetupConn, this function performs the handshake protocol and tries to create a peer object for the connection.


// SetupConn runs the handshakes and attempts to add the connection
// as a peer. It returns when the connection has been added as a peer
// or the handshakes have failed.
Func (srv *Server) SetupConn(fd net.Conn, flags connFlag, dialDest *discover.Node) {
// Prevent leftover pending conns from entering the handshake.
srv.lock.Lock()
Running := srv.running
srv.lock.Unlock()
//Created a conn object. The newTransport pointer actually points to the newRLPx method. In fact, fd is packaged with the rlpx protocol.
c := &amp;conn{fd: fd, transport: srv.newTransport(fd), flags: flags, cont: make(chan error)}
If !running {
C.close(errServerStopped)
Return
}
// Run the encryption handshake.
Var err error
//The actual implementation here is doEncHandshake in rlpx.go. Because transport is an anonymous field of conn. The method of anonymous fields will be used directly as a method of conn.
If c.id, err = c.doEncHandshake(srv.PrivateKey, dialDest); err != nil {
log.Trace("Failed RLPx handshake", "addr", c.fd.RemoteAddr(), "conn", c.flags, "err", err)
C.close(err)
Return
}
Clog := log.New("id", c.id, "addr", c.fd.RemoteAddr(), "conn", c.flags)
// For dialed connections, check that the remote public key matches.
// If the ID of the connection handshake does not match the corresponding ID
If dialDest != nil &amp;&amp; c.id != dialDest.ID {
C.close(DiscUnexpectedIdentity)
clog.Trace("Dialed identity mismatch", "want", c, dialDest.ID)
Return
}
// This checkpoint is actually sending the first argument to the queue specified by the second argument. Then receive the return message from c.cout. Is a synchronous method.
// As for this, the subsequent operation just checks if the connection is legal and returns.
If err := srv.checkpoint(c, srv.posthandshake); err != nil {
clog.Trace("Rejected peer before protocol handshake", "err", err)
C.close(err)
Return
}
// Run the protocol handshake
Phs, err := c.doProtoHandshake(srv.ourHandshake)
If err != nil {
clog.Trace("Failed proto handshake", "err", err)
C.close(err)
Return
}
If phs.ID != c.id {
clog.Trace("Wrong devp2p handshake identity", "err", phs.ID)
C.close(DiscUnexpectedIdentity)
Return
}
C.caps, c.name = phs.Caps, phs.Name
// The two handshakes have been completed here. Send c to the addpeer queue. This connection is handled when the queue is processed in the background.
If err := srv.checkpoint(c, srv.addpeer); err != nil {
clog.Trace("Rejected peer", "err", err)
C.close(err)
Return
}
// If the checks completed successfully, runPeer has now been
// launched by run.
}


The process mentioned above is the process of listenLoop, and listenLoop is mainly used to receive external active connecters. There are also cases where the node needs to initiate a connection to connect to the external node. And the process of processing the checkpoint queue information just above. This part of the code is in the goroutine of server.run.



Func (srv *Server) run(dialstate dialer) {
Defer srv.loopWG.Done()
Var (
Peers = make(map[discover.NodeID]*Peer)
Trusted = make(map[discover.NodeID]bool, len(srv.TrustedNodes))
Taskdone = make(chan task, maxActiveDialTasks)
runningTasks []task
queuedTasks []task // tasks that can't run yet
)
// Put trusted nodes into a map to speed up checks.
// Trusted peers are loaded on startup and cannot be
// modified while the server is running.
// The trusted node has such a feature. If there are too many connections, the other nodes will be rejected. But the trusted node will be received.
For _, n := range srv.TrustedNodes {
Trusted[n.ID] = true
}

// removes t from runningTasks
// Define a function to remove a Task from the runningTasks queue
delTask ​​:= func(t task) {
For i := range runningTasks {
If runningTasks[i] == t {
runningTasks = append(runningTasks[:i], runningTasks[i+1:]...)
Break
}
}
}
// starts until max number of active tasks is satisfied
// The number of nodes that started to connect at the same time is 16. Traverse the runningTasks queue and start these tasks.
startTasks := func(ts []task) (rest []task) {
i := 0
For ; len(runningTasks) &lt; maxActiveDialTasks &amp;&amp; i &lt; len(ts); i++ {
t := ts[i]
log.Trace("New dial task", "task", t)
Go func() { t.Do(srv); taskdone &lt;- t }()
runningTasks = append(runningTasks, t)
}
Return ts[i:]
}
scheduleTasks := func() {
// Start from queue first.
// First call startTasks to start part and return the rest to queuedTasks.
queuedTasks = append(queuedTasks[:0], startTasks(queuedTasks)...)
// Query dialer for new tasks and start as many as possible now.
// Call newTasks to generate the task and try to start with startTasks. And put the queue that can't be started temporarily into the queuedTasks queue.
If len(runningTasks) &lt; maxActiveDialTasks {
Nt := dialstate.newTasks(len(runningTasks)+len(queuedTasks), peers, time.Now())
queuedTasks = append(queuedTasks, startTasks(nt)...)
}
}

Running:
For {
// Call dialstate.newTasks to generate a new task. And call startTasks to start a new task.
// If the dialTask ​​has all been started, a sleep timeout task will be generated.
scheduleTasks()

Select {
Case &lt;-srv.quit:
// The server was stopped. Run the cleanup logic.
Break running
Case n := &lt;-srv.addstatic:
// This channel is used by AddPeer to add to the
// ephemeral static peer list. Add it to the dialer,
// it will keep the node connected.
log.Debug("Adding static node", "node", n)
dialstate.addStatic(n)
Case n := &lt;-srv.removestatic:
// This channel is used by RemovePeer to send a
// disconnect request to a peer and begin the
// stop keeping the node connected
log.Debug("Removing static node", "node", n)
dialstate.removeStatic(n)
If p, ok := peers[n.ID]; ok {
p.Disconnect(DiscRequested)
}
Case op := &lt;-srv.peerOp:
// This channel is used by Peers and PeerCount.
Op(peers)
srv.peerOpDone &lt;- struct{}{}
Case t := &lt;-taskdone:
// A task got done. Tell dialstate about it so it
// can update its state and remove it from the active
// tasks list.
log.Trace("Dial task done", "task", t)
dialstate.taskDone(t, time.Now())
delTask(t)
Case c := &lt;-srv.posthandshake:
// A connection has passed the encryption handshake so
// the remote identity is known (but hasn't been verified yet).
// Remember to call the checkpoint method before, and the connection will be sent to this channel.
If trusted[c.id] {
// Ensure that the trusted flag is set before checking against MaxPeers.
C.flags |= trustedConn
}
// TODO: track in-progress inbound node IDs (pre-Peer) to avoid dialing them.
Select {
Case c.cont &lt;- srv.encHandshakeChecks(peers, c):
Case &lt;-srv.quit:
Break running
}
Case c := &lt;-srv.addpeer:
// At this point the connection is past the protocol handshake.
// Its capabilities are known and the remote identity is verified.
// After two handshakes, checkpoint is called to send the connection to the addpeer channel.
// Then create a Peer object via newPeer.
// Start a goroutine to start the peer object. The peer.run method is called.
Err := srv.protoHandshakeChecks(peers, c)
If err == nil {
// The handshakes are done and it passed all checks.
p := newPeer(c, srv.Protocols)
// If message events are enabled, pass the peerFeed
// to the peer
If srv.EnableMsgEvents {
P.events = &amp;srv.peerFeed
}
Name := truncateName(c.name)
log.Debug("Adding p2p peer", "id", c.id, "name", name, "addr", c.fd.RemoteAddr(), "peers", len(peers)+1)
Peers[c.id] = p
Go srv.runPeer(p)
}
// The dialer logic relies on the assumption that
// dial tasks complete after the peer has been added or
// discarded. Unblock the task last.
Select {
Case c.cont &lt;- err:
Case &lt;-srv.quit:
Break running
}
Case pd := &lt;-srv.delpeer:
// A peer disconnected.
d := common.PrettyDuration(mclock.Now() - pd.created)
pd.log.Debug("Removing p2p peer", "duration", d, "peers", len(peers)-1, "req", pd.requested, "err", pd.err)
Delete(peers, pd.ID())
}
}

log.Trace("P2P networking is spinning down")

// Terminate discovery. If there is a running lookup it will terminate soon.
If srv.ntab != nil {
srv.ntab.Close()
}
If srv.DiscV5 != nil {
srv.DiscV5.Close()
}
// Disconnect all peers.
For _, p := range peers {
p.Disconnect(DiscQuitting)
}
// Wait for peers to shut down. Pending connections and tasks are
// not handled here and will terminate soon-ish because srv.quit
// is closed.
For len(peers) &gt; 0 {
p := &lt;-srv.delpeer
p.log.Trace("&lt;-delpeer (spindown)", "remainingTasks", len(runningTasks))
Delete(peers, p.ID())
}
}


runPeer method

// runPeer runs in its own goroutine for each peer.
// it waits until the Peer logic returns and removes
// the peer.
Func (srv *Server) runPeer(p *Peer) {
If srv.newPeerHook != nil {
srv.newPeerHook(p)
}

// broadcast peer add
srv.peerFeed.Send(&amp;PeerEvent{
Type: PeerEventTypeAdd,
Peer: p.ID(),
})

// run the protocol
remoteRequested, err := p.run()

// broadcast peer drop
srv.peerFeed.Send(&amp;PeerEvent{
Type: PeerEventTypeDrop,
Peer: p.ID(),
Error: err.Error(),
})

// Note: run waits for existing peers to be sent on srv.delpeer
// before returning, so this send should not select on srv.quit.
Srv.delpeer &lt;- peerDrop{p, err, remoteRequested}
}


to sum up:

The main work done by the server object combines all the components introduced earlier. Use rlpx.go to handle encrypted links. Use discover to handle node discovery and lookups. Use dial to generate and connect the nodes that need to be connected. Use the peer object to handle each connection.

The server starts a listenLoop to listen for and receive new connections. Start a run goroutine to call dialstate to generate a new dial task and connect. Use channels to communicate and cooperate between goroutines.</pre></body></html>