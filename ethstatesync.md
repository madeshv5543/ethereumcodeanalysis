
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Statesync is used to get the trie tree of all the states of the block specified by the pivot point, that is, the information of all accounts, including ordinary accounts and contract accounts.

## data structure
stateSync schedules the download of a request for a particular state trie defined by a given state root.

// stateSync schedules requests for downloading a particular state trie defined
// by a given state root.
Type stateSync struct {
d *Downloader // Downloader instance to access and manage current peerset

Sched *trie.TrieSync // State trie sync scheduler defining the tasks
Keccak hash.Hash // Keccak256 hasher to verify deliveries with
Tasks map[common.Hash]*stateTask // Set of tasks currently queued for retrieval

numUncommitted int
bytesUncommitted int

Deliver chan *stateReq // Delivery channel multiplexing peer responses
Cancel chan struct{} // Channel to signal a termination request
cancelOnce sync.Once // Ensures cancel only ever gets called once
Done chan struct{} // Channel to signal termination completion
Err error // Any error hit during sync (set before completion)
}

Constructor

Func newStateSync(d *Downloader, root common.Hash) *stateSync {
Return &amp;stateSync{
d: d,
Sched: state.NewStateSync(root, d.stateDB),
Keccak: sha3.NewKeccak256(),
Tasks: make(map[common.Hash]*stateTask),
Deliver: make(chan *stateReq),
Cancel: make(chan struct{}),
Done: make(chan struct{}),
}
}

NewStateSync

// NewStateSync create a new state trie download scheduler.
Func NewStateSync(root common.Hash, database trie.DatabaseReader) *trie.TrieSync {
Var syncer *trie.TrieSync
Callback := func(leaf []byte, parent common.Hash) error {
Var obj Account
If err := rlp.Decode(bytes.NewReader(leaf), &amp;obj); err != nil {
Return err
}
syncer.AddSubTrie(obj.Root, 64, parent, nil)
syncer.AddRawEntry(common.BytesToHash(obj.CodeHash), 64, parent)
Return nil
}
Syncer = trie.NewTrieSync(root, database, callback)
Return syncer
}

syncState, this function is called by the downloader.

// syncState starts downloading state with the given root hash.
Func (d *Downloader) syncState(root common.Hash) *stateSync {
s := newStateSync(d, root)
Select {
Case d.stateSyncStart &lt;- s:
Case &lt;-d.quitCh:
S.err = errCancelStateFetch
Close(s.done)
}
Return s
}

## start up
A new goroutine is started in the downloader to run the stateFetcher function. This function first tries to get information from the stateSyncStart channel. The syncState function will send data to the stateSyncStart channel.

// stateFetcher manages the active state sync and accepts requests
// on its behalf.
Func (d *Downloader) stateFetcher() {
For {
Select {
Case s := &lt;-d.stateSyncStart:
For next := s; next != nil; { // This for loop represents that the downloader can change the object that needs to be synchronized at any time by sending a signal.
Next = d.runStateSync(next)
}
Case &lt;-d.stateCh:
// Ignore state responses while no sync is running.
Case &lt;-d.quitCh:
Return
}
}
}

Let's see below where we will call the syncState() function. The processFastSyncContent function will start when the peer is first discovered.

// processFastSyncContent takes fetch results from the queue and writes them to the
// database. It also controls the synchronisation of state nodes of the pivot block.
Func (d *Downloader) processFastSyncContent(latest *types.Header) error {
// Start syncing state of the reported head block.
// This should get us most of the state of the pivot block.
stateSync := d.syncState(latest.Root)



runStateSync, this method gets the downloaded state from stateCh, and then delivers it to the deliver channel for others to process.

// runStateSync runs a state synchronisation until it completes or another root
// hash is requested to be switched over to.
Func (d *Downloader) runStateSync(s *stateSync) *stateSync {
Var (
Active = make(map[string]*stateReq) // Currently in-flight requests
Finished []*stateReq // Completed or failed requests
Timeout = make(chan *stateReq) // Timed out active requests
)
Defer func() {
// Cancel active request timers on exit. Also set peers to idle so they're
// available for the next sync.
For _, req := range active {
req.timer.Stop()
req.peer.SetNodeDataIdle(len(req.items))
}
}()
// Run the state sync.
// running state synchronization
Go s.run()
Defer s.Cancel()

// Listen for peer departure events to cancel assigned tasks
peerDrop := make(chan *peerConnection, 1024)
peerSub := s.d.peers.SubscribePeerDrops(peerDrop)
Defer peerSub.Unsubscribe()

For {
// Enable sending of the first buffered element if there is one.
Var (
deliverReq *stateReq
deliverReqCh chan *stateReq
)
If len(finished) &gt; 0 {
deliverReq = finished[0]
deliverReqCh = s.deliver
}

Select {
// The stateSync lifecycle:
// Another stateSync application is running. We quit.
Case next := &lt;-d.stateSyncStart:
Return next

Case &lt;-s.done:
Return nil

// Send the next finished request to the current sync:
// Send the downloaded data to sync
Case deliverReqCh &lt;- deliverReq:
Finished = append(finished[:0], finished[1:]...)

// Handle incoming state packs:
// Process incoming packets. The data that the downloader receives the state will be sent to this channel.
Case pack := &lt;-d.stateCh:
// Discard any data not requested (or previsouly timed out)
Req := active[pack.PeerId()]
If req == nil {
log.Debug("Unrequested node data", "peer", pack.PeerId(), "len", pack.Items())
Continue
}
// Finalize the request and queue up for processing
req.timer.Stop()
Req.response = pack.(*statePack).states

Finished = append(finished, req)
Delete(active, pack.PeerId())

// Handle dropped peer connections:
Case p := &lt;-peerDrop:
// Skip if no request is currently pending
Req := active[p.id]
If req == nil {
Continue
}
// Finalize the request and queue up for processing
req.timer.Stop()
Req.dropped = true

Finished = append(finished, req)
Delete(active, p.id)

// Handle timed-out requests:
Case req := &lt;-timeout:
// If the peer is already requesting something else, ignore the stale timeout.
// This can happen when the timeout and the delivery happens simultaneous,
// causing both pathways to trigger.
If active[req.peer.id] != req {
Continue
}
// Move the timed out data back into the download queue
Finished = append(finished, req)
Delete(active, req.peer.id)

// Track outgoing state requests:
Case req := &lt;-d.trackStateReq:
// If an active request already exists for this peer, we have a problem. In
// theory the trie node schedule must never assign two requests to the same
// peer. In practive however, a peer might receive a request, disconnect and
// immediately reconnect before the previous times out. In this case the first
// request is never honored, alas we must not silently overwrite it, as that
// causes valid requests to go missing and sync to get stuck.
If old := active[req.peer.id]; old != nil {
log.Warn("Busy peer assigned new state fetch", "peer", old.peer.id)

// Make sure the previous one doesn't get siletly lost
old.timer.Stop()
Old.dropped = true

Finished = append(finished, old)
}
// Start a timer to notify the sync loop if the peer stalled.
Req.timer = time.AfterFunc(req.timeout, func() {
Select {
Case timeout &lt;- req:
Case &lt;-s.done:
// Prevent leaking of timer goroutines in the unlikely case where a
// timer is fired just before exiting runStateSync.
}
})
Active[req.peer.id] = req
}
}
}


Run and loop methods, get tasks, assign tasks, and get results.

Func (s *stateSync) run() {
S.err = s.loop()
Close(s.done)
}

// loop is the main event loop of a state trie sync. It it responsible for the
// assignment of new tasks to peers (including sending it to them) as well as
// for the processing of inbound data. Note, that the loop does not directly
// receive data from peers, rather those are buffered up in the downloader and
// push here async. The reason is to decouple processing from data receipt
// and timeouts.
Func (s *stateSync) loop() error {
// Listen for new peer events to assign tasks to them
newPeer := make(chan *peerConnection, 1024)
peerSub := s.d.peers.SubscribeNewPeers(newPeer)
Defer peerSub.Unsubscribe()

// Keep assigning new tasks until the sync completes or aborts
// wait until sync is completed or is terminated
For s.sched.Pending() &gt; 0 {
// Refresh the data from the cache to the persistent store. This is the size specified by the command line --cache.
If err := s.commit(false); err != nil {
Return err
}
// Assign a task,
s.assignTasks()
// Tasks assigned, wait for something to happen
Select {
Case &lt;-newPeer:
// new peer arrived, try to assign it download tasks

Case &lt;-s.cancel:
Return errCancelStateFetch

Case req := &lt;-s.deliver:
// Received the return message sent by the runStateSync method. Note that the return message contains the successful request and also contains the unsuccessful request.
// Response, disconnect or timeout triggered, drop the peer if stalling
log.Trace("Received node data response", "peer", req.peer.id, "count", len(req.response), "dropped", req.dropped, "timeout", !req.dropped &amp;&amp; req .timedOut())
If len(req.items) &lt;= 2 &amp;&amp; !req.dropped &amp;&amp; req.timedOut() {
// 2 items are the minimum requested, if even that times out, we've no use of
// this peer at the moment.
log.Warn("Stalling state sync, dropping peer", "peer", req.peer.id)
s.d.dropPeer(req.peer.id)
}
// Process all the received blobs and check for stale delivery
Stale, err := s.process(req)
If err != nil {
log.Warn("Node data write error", "err", err)
Return err
}
// The the delivery contains requested data, mark the node idle (otherwise it's a timed out delivery)
If !stale {
req.peer.SetNodeDataIdle(len(req.response))
}
}
}
Return s.commit(true)
}</pre></body></html>