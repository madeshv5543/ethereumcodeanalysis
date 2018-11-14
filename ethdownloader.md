
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>The downloader is mainly responsible for the initial synchronization of the blockchain. The current synchronization has two modes. One is the traditional fullmode. This mode builds the blockchain by downloading the block header and the block body. The synchronization process is The normal block insertion process is the same, including block header verification, transaction verification, transaction execution, account status change, etc. This is actually a process that consumes CPU and disk. The other mode is the fast sync mode of fast sync, which has a special document to describe. Please refer to the documentation for fast sync. Simply put, the fast sync mode downloads the block header, block body and receipt. The insertion process does not execute the transaction, and then synchronizes all account statuses at a block height (the highest block height - 1024), followed by The 1024 blocks will be built using fullmode. This mode will increase the insertion time of the block without generating a large amount of historical account information. It will save disk relatively, but it will consume more network. Because you need to download the receipt and status.

## downloader Data Structure


Type Downloader struct {
Mode SyncMode // Synchronisation mode defining the strategy used (per sync cycle)
Mux *event.TypeMux // Event multiplexer to announce sync operation events
// The queue object is used to schedule block headers, transactions, and receipt downloads, as well as assembly after download.
Queue *queue // Scheduler for selecting the hashes to download
// The collection of the peer
Peers *peerSet // Set of active peers from which download can proceed
stateDB ethdb.Database
// The head of the Pivot point block in fast sync
fsPivotLock *types.Header // Pivot header on critical section entry (cannot change between retries)
fsPivotFails uint32 // Number of subsequent fast sync failures in the clear section
// Download round trip delay
rttEstimate uint64 // Round trip time to target for download requests
rttConfidence uint64 // Confidence in the estimated RTT (unit: millionths to allow atomic ops) Estimate the confidence of the RTT (unit: one millionth of the atomic operation allowed)

// Statistics statistics,
syncStatsChainOrigin uint64 // Origin block number where syncing started at
syncStatsChainHeight uint64 // Highest block number known when syncing started
syncStatsState stateSyncStats
syncStatsLock sync.RWMutex // Lock protecting the sync stats fields

Lightchain LightChain
Blockchain BlockChain

// Callbacks
dropPeer peerDropFn // Drops a peer for misbehaving

// Status
synchroniseMock func(id string, hash common.Hash) error // Replacement for synchronise during testing
Synchronising int32
Notified int32

// Channels
headerCh chan dataPack // [eth/62] Channel input inbound block headers header input channel, the header downloaded from the network will be sent to this channel
bodyCh chan dataPack // [eth/62] Channel receiving inbound block bodies body input channels, the hearts downloaded from the network will be sent to this channel
receiptCh chan dataPack // [eth/63] Channel receiving inbound receipts inputs input channels, receives downloaded from the network will be sent to this channel
bodyWakeCh chan bool // [eth/62] Channel to signal the block body fetcher of new tasks channel for transferring body fetcher new tasks
receiptWakeCh chan bool // [eth/63] Channel to signal the receipt fetcher of new tasks channel used to transmit the new task of receipt fetcher
headerProcCh chan []*types.Header // [eth/62] Channel to feed the header processor new tasks channel provides new tasks for the header handler

// for stateFetcher
stateSyncStart chan *stateSync // used to start a new state fetcher
trackStateReq chan *stateReq // TODO
stateCh chan dataPack // [eth/63] Channel receiving inbound node state data state input channel, the state downloaded from the network will be sent to this channel
// Cancellation and termination
cancelPeer string // Identifier of the peer currently being used as the master (cancel on drop)
cancelCh chan struct{} // Channel to cancel mid-flight syncs
cancelLock sync.RWMutex // Lock to protect the cancel channel and peer in delivers

quitCh chan struct{} // Quit channel to signal termination
quitLock sync.RWMutex // Lock to prevent double closes

// Testing hooks
syncInitHook func(uint64, uint64) // Method to call upon initiating a new sync run
bodyFetchHook func([]*types.Header) // Method to call upon starting a block body fetch
receiptFetchHook func([]*types.Header) // Method to call upon starting a receipt fetch
chainInsertHook func([]*fetchResult) // Method to call upon inserting a chain of blocks (possibly in multiple invocations)
}


Construction method


// New creates a new downloader to fetch hashes and blocks from remote peers.
Func New(mode SyncMode, stateDb ethdb.Database, mux *event.TypeMux, chain BlockChain, lightchain LightChain, dropPeer peerDropFn) *Downloader {
If lightchain == nil {
Lightchain = chain
}
Dl := &amp;Downloader{
Mode: mode,
stateDB: stateDb,
Mux: mux,
Queue: newQueue(),
Peers: newPeerSet(),
rttEstimate: uint64(rttMaxEstimate),
rttConfidence: uint64(1000000),
Blockchain: chain,
Lightchain: lightchain,
dropPeer: dropPeer,
headerCh: make(chan dataPack, 1),
bodyCh: make(chan dataPack, 1),
receiptCh: make(chan dataPack, 1),
bodyWakeCh: make(chan bool, 1),
receiptWakeCh: make(chan bool, 1),
headerProcCh: make(chan []*types.Header, 1),
quitCh: make(chan struct{}),
stateCh: make(chan dataPack),
stateSyncStart: make(chan *stateSync),
trackStateReq: make(chan *stateReq),
}
Go dl.qosTuner() //Simple Mainly used to calculate rttEstimate and rttConfidence
Go dl.stateFetcher() //Starts the task listener for stateFetcher, but this time there is no task to generate state fetcher.
Return dl
}

## Synchronous download
Synchronise tries to synchronize with a peer. If there are some errors during the synchronization, Peer will be deleted. Then I will be tried again.

// Synchronise tries to sync up our local block chain with a remote peer, both
//add various sanity checks as well as wrapping it with various log entries.
Func (d *Downloader) Synchronise(id string, head common.Hash, td *big.Int, mode SyncMode) error {
Err := d.synchronise(id, head, td, mode)
Switch err {
Case nil:
Case errBusy:

Case errTimeout, errBadPeer, errStallingPeer,
errEmptyHeaderSet, errPeersUnavailable, errTooOld,
errInvalidAncestor, errInvalidChain:
log.Warn("Synchronisation failed, dropping peer", "peer", id, "err", err)
d.dropPeer(id)

Default:
log.Warn("Synchronisation failed, retrying", "err", err)
}
Return err
}


Synchronise

// synchronise will select the peer and use it for synchronising. If an empty string is given
// it will use the best peer possible and synchronize if it's TD is higher than our own. If any of the
// checks fail an error will be returned. This method is synchronous
Func (d *Downloader) synchronise(id string, hash common.Hash, td *big.Int, mode SyncMode) error {
// Mock out the synchronisation if testing
If d.synchroniseMock != nil {
Return d.synchroniseMock(id, hash)
}
// Make sure only one goroutine is ever allowed past this point at once
// This method can only run one at a time, check if it is running.
If !atomic.CompareAndSwapInt32(&amp;d.synchronising, 0, 1) {
Return errBusy
}
Defer atomic.StoreInt32(&amp;d.synchronising, 0)

// Post a user notification of the sync (only once per session)
If atomic.CompareAndSwapInt32(&amp;d.notified, 0, 1) {
log.Info("Block synchronisation started")
}
// Reset the queue, peer set and wake channels to clean any internal leftover state
// Reset the state of the queue and peer.
d.queue.Reset()
d.peers.Reset()
// Empty d.bodyWakeCh, d.receiptWakeCh
For _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
Select {
Case &lt;-ch:
Default:
}
}
// Empty d.headerCh, d.bodyCh, d.receiptCh
For _, ch := range []chan dataPack{d.headerCh, d.bodyCh, d.receiptCh} {
For empty := false; !empty; {
Select {
Case &lt;-ch:
Default:
Empty = true
}
}
}
// Clear headerProcCh
For empty := false; !empty; {
Select {
Case &lt;-d.headerProcCh:
Default:
Empty = true
}
}
// Create cancel channel for aborting mid-flight and mark the master peer
d.cancelLock.Lock()
d.cancelCh = make(chan struct{})
d.cancelPeer = id
d.cancelLock.Unlock()

Defer d.Cancel() // No matter what, we can't leave the cancel channel open

// Set the requested sync mode, unless it's forbidden
D.mode = mode
If d.mode == FastSync &amp;&amp; atomic.LoadUint32(&amp;d.fsPivotFails) &gt;= fsCriticalTrials {
D.mode = FullSync
}
// Retrieve the origin peer and initiate the downloading process
p := d.peers.Peer(id)
If p == nil {
Return errUnknownPeer
}
Return d.syncWithPeer(p, hash, td)
}

syncWithPeer

// syncWithPeer starts a block synchronization based on the hash chain from the
// specified peer and head hash.
Func (d *Downloader) syncWithPeer(p *peerConnection, hash common.Hash, td *big.Int) (err error) {
...
// Look up the sync boundaries: the common ancestor and the target block
/ / Use the hash finger to get the block header, this method will access the network
Latest, err := d.fetchHeight(p)
If err != nil {
Return err
}
Height := latest.Number.Uint64()
// findAncestor tries to get the common ancestor of everyone to find a point to start syncing.
Origin, err := d.findAncestor(p, height)
If err != nil {
Return err
}
d.syncStatsLock.Lock()
If d.syncStatsChainHeight &lt;= origin || d.syncStatsChainOrigin &gt; origin {
d.syncStatsChainOrigin = origin
}
d.syncStatsChainHeight = height
d.syncStatsLock.Unlock()

// Initiate the sync using a concurrent header and content retrieval algorithm
Pivot := uint64(0)
Switch d.mode {
Case LightSync:
Pivot = height
Case FastSync:
// Calculate the new fast/slow sync pivot point
// If the pivot point is not locked.
If d.fsPivotLock == nil {
pivotOffset, err := rand.Int(rand.Reader, big.NewInt(int64(fsPivotInterval)))
If err != nil {
Panic(fmt.Sprintf("Failed to access crypto random source: %v", err)
}
If height &gt; uint64(fsMinFullBlocks)+pivotOffset.Uint64() {
Pivot = height - uint64(fsMinFullBlocks) - pivotOffset.Uint64()
}
} else { // If this point has already been locked. Then use this point
// Pivot point locked in, use this and do not pick a new one!
Pivot = d.fsPivotLock.Number.Uint64()
}
// If the point is below the origin, move origin back to ensure state download
If pivot &lt; origin {
If pivot &gt; 0 {
Origin = pivot - 1
} else {
Origin = 0
}
}
log.Debug("Fast syncing until pivot block", "pivot", pivot)
}
d.queue.Prepare(origin+1, d.mode, pivot, latest)
If d.syncInitHook != nil {
d.syncInitHook(origin, height)
}
// Start several fetchers responsible for headers, toys, receipts, handle headers
Fetchers := []func() error{
Func() error { return d.fetchHeaders(p, origin+1) }, // Headers are always retrieved
Func() error { return d.fetchBodies(origin + 1) }, // Bodies are retrieved during normal and fast sync
Func() error { return d.fetchReceipts(origin + 1) }, // Receipts are retrieved during fast sync
Func() error { return d.processHeaders(origin+1, td) },
}
If d.mode == FastSync { //Add new processing logic depending on the mode
Fetchers = append(fetchers, func() error { return d.processFastSyncContent(latest) })
} else if d.mode == FullSync {
Fetchers = append(fetchers, d.processFullSyncContent)
}
Err = d.spawnSync(fetchers)
If err != nil &amp;&amp; d.mode == FastSync &amp;&amp; d.fsPivotLock != nil {
// If sync failed in the critical section, bump the fail counter.
atomic.AddUint32(&amp;d.fsPivotFails, 1)
}
Return err
}

spawnSync starts a goroutine for each fetcher and then blocks the fetcher error.

// spawnSync runs d.process and all given fetcher functions to completion in
// separate goroutines, returning the first error that appears.
Func (d *Downloader) spawnSync(fetchers []func() error) error {
Var wg sync.WaitGroup
Errc := make(chan error, len(fetchers))
wg.Add(len(fetchers))
For _, fn := range fetchers {
Fn := fn
Go func() { defer wg.Done(); errc &lt;- fn() }()
}
// Wait for the first error, then terminate the others.
Var err error
For i := 0; i &lt; len(fetchers); i++ {
If i == len(fetchers)-1 {
// Close the queue when all fetchers have exited.
// This will cause the block processor to end when
// it has processed the queue.
d.queue.Close()
}
If err = &lt;-errc; err != nil {
Break
}
}
d.queue.Close()
d.Cancel()
wg.Wait()
Return err
}

## headers processing

The fetchHeaders method is used to get the header. Then according to the obtained header to get information such as body and receipt.

// fetchHeaders keeps retrieving headers concurrently from the number
// requested, until no more are returned, potentially throttling on the way. To
// facilitate concurrency but still protect against malicious nodes sending bad
// headers, we construct a header chain skeleton using the "origin" peer we are
// syncing with, and fill in the missing headers using anyone else. Headers from
// other peers are only accepted if they map cleanly to the skeleton. If no one
// can fill in the skeleton - not even the origin peer - it's assumed invalid and
// the origin is dropped.
fetchHeaders continually repeats such operations, sending header requests, waiting for all returns. Until all header requests are completed. To improve concurrency while still preventing malicious nodes from sending the wrong header, we construct a header file chain skeleton using the "origin" peer we are synchronizing, and use others to fill the missing headers. Other peer headers are only accepted when they are cleanly mapped to the skeleton. If no one can fill the skeleton - even the original peer can't fill it - it is considered invalid and the origin peer is also discarded.
Func (d *Downloader) fetchHeaders(p *peerConnection, from uint64) error {
p.log.Debug("Directing header downloads", "origin", from)
Defer p.log.Debug("Header download terminated")

// Create a timeout timer, and the associated header fetcher
Skeleton := true // Skeleton assembly phase or finishing up
Request := time.Now() // time of the last skeleton fetch request
Timeout := time.NewTimer(0) // timer to dump a non-responsive active peer
&lt;-timeout.C // timeout channel should be initially empty
Defer timeout.Stop()

Var ttl time.Duration
getHeaders := func(from uint64) {
Request = time.Now()

Ttl = d.requestTTL()
timeout.Reset(ttl)

If skeleton { //fill the skeleton
p.log.Trace("Fetching skeleton headers", "count", MaxHeaderFetch, "from", from)
Go p.peer.RequestHeadersByNumber(from+uint64(MaxHeaderFetch)-1, MaxSkeletonSize, MaxHeaderFetch-1, false)
} else { // direct request
p.log.Trace("Fetching full headers", "count", MaxHeaderFetch, "from", from)
Go p.peer.RequestHeadersByNumber(from, MaxHeaderFetch, 0, false)
}
}
// Start pulling the header chain skeleton until all is done
getHeaders(from)

For {
Select {
Case &lt;-d.cancelCh:
Return errCancelHeaderFetch

Case packet := &lt;-d.headerCh: //The header returned on the network will be delivered to the headerCh channel.
// Make sure the active peer is giving us the skeleton headers
If packet.PeerId() != p.id {
log.Debug("Received skeleton in in peer", "peer", packet.PeerId())
Break
}
headerReqTimer.UpdateSince(request)
timeout.Stop()

// If the skeleton's finished, pull any remaining head headers directly from the origin
If packet.Items() == 0 &amp;&amp; skeleton {
Skeleton = false
getHeaders(from)
Continue
}
// If no more headers are inbound, notify the content fetchers and return
// If there are no more returns. Then tell the headerProcCh channel
If packet.Items() == 0 {
p.log.Debug("No more headers available")
Select {
Case d.headerProcCh &lt;- nil:
Return nil
Case &lt;-d.cancelCh:
Return errCancelHeaderFetch
}
}
Headers := packet.(*headerPack).headers

// If we received a skeleton batch, resolve internals concurrently
If skeleton { // If it is necessary to fill the skeleton, then fill it in this method
Filled, proced, err := d.fillHeaderSkeleton(from, headers)
If err != nil {
p.log.Debug("Skeleton chain invalid", "err", err)
Return errInvalidChain
}
Headers = filled[proced:]
// proced represents how many have been processed. So only need proced: the headers behind
From += uint64(proced)
}
// Insert all the new headers and fetch the next batch
If len(headers) &gt; 0 {
p.log.Trace("Scheduling new headers", "count", len(headers), "from", from)
//post to headerProcCh and continue to loop.
Select {
Case d.headerProcCh &lt;- headers:
Case &lt;-d.cancelCh:
Return errCancelHeaderFetch
}
From += uint64(len(headers))
}
getHeaders(from)

Case &lt;-timeout.C:
// Header retrieval timed out, consider the peer bad and drop
p.log.Debug("Header request timed out", "elapsed", ttl)
headerTimeoutMeter.Mark(1)
d.dropPeer(p.id)

// Finish the sync gracefully instead of of dumping the aggregate data though
For _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
Select {
Case ch &lt;- false:
Case &lt;-d.cancelCh:
}
}
Select {
Case d.headerProcCh &lt;- nil:
Case &lt;-d.cancelCh:
}
Return errBadPeer
}
}
}

The processHeaders method, which takes the header from the headerProcCh channel. And the obtained header is thrown into the queue for scheduling, so that the body fetcher or the receipt fetcher can receive the fetch task.

// processHeaders takes batches of retrieved headers from an input channel and
// keeps processing and scheduling them into the header chain and downloader's
// queue until the stream ends or a failure occurs.
// processHeaders batch gets headers, processes them, and dispatches them via the downloader's queue object. Until the error occurs or the process ends.
Func (d *Downloader) processHeaders(origin uint64, td *big.Int) error {
// Calculate the pivoting point for switching from fast to slow sync
Pivot := d.queue.FastSyncPivot()

// Keep a count of uncertain headers to roll back
// rollback is used to handle this logic if a point fails. Then the 2048 nodes previously inserted must be rolled back. Because security is not up to standard, you can refer to the fast sync documentation for details.
Rollback := []*types.Header{}
Defer func() { // This function is used to roll back when the error exits. TODO
If len(rollback) &gt; 0 {
// Flatten the headers and roll them back
Hashes := make([]common.Hash, len(rollback))
For i, header := range rollback {
Hashes[i] = header.Hash()
}
lastHeader, lastFastBlock, lastBlock := d.lightchain.CurrentHeader().Number, common.Big0, common.Big0
If d.mode != LightSync {
lastFastBlock = d.blockchain.CurrentFastBlock().Number()
lastBlock = d.blockchain.CurrentBlock().Number()
}
d.lightchain.Rollback(hashes)
curFastBlock, curBlock := common.Big0, common.Big0
If d.mode != LightSync {
curFastBlock = d.blockchain.CurrentFastBlock().Number()
curBlock = d.blockchain.CurrentBlock().Number()
}
log.Warn("Rolled back headers", "count", len(hashes),
"header", fmt.Sprintf("%d-&gt;%d", lastHeader, d.lightchain.CurrentHeader().Number),
"fast", fmt.Sprintf("%d-&gt;%d", lastFastBlock, curFastBlock),
"block", fmt.Sprintf("%d-&gt;%d", lastBlock, curBlock))

// If we're already past the pivot point, this could be an attack, thread carefully
If rollback[len(rollback)-1].Number.Uint64() &gt; pivot {
// If we didn't ever fail, lock in the pivot header (must! not! change!)
If atomic.LoadUint32(&amp;d.fsPivotFails) == 0 {
For _, header := range rollback {
If header.Number.Uint64() == pivot {
log.Warn("Fast-sync pivot locked in", "number", pivot, "hash", header.Hash())
d.fsPivotLock = header
}
}
}
}
}
}()

// Wait for batches of headers to process
gotHeaders := false

For {
Select {
Case &lt;-d.cancelCh:
Return errCancelHeaderProcessing

Case headers := &lt;-d.headerProcCh:
// Terminate header processing if we synced up
If len(headers) == 0 { //Processing completed
// Notify everyone that headers are fully processed
For _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
Select {
Case ch &lt;- false:
Case &lt;-d.cancelCh:
}
}
// If no headers were retrieved at all, the peer violated it's TD promise that it had a
// better chain compared to ours. The only exception is if it's promised blocks were
//owned imported by other means (e.g. fecher):
//
// R &lt;remote peer&gt;, L &lt;local node&gt;: Both at block 10
// R: Mine block 11, and propagate it to L
// L: Queue block 11 for import
// L: Notice that R's head and TD increased compared to ours, start sync
// L: Import of block 11 finishes
// L: Sync begins, and finds common ancestor at 11
// L: Request new headers up from 11 (R's TD was higher, it must have something)
// R: Nothing to give
If d.mode != LightSync { // The other party's TD is bigger than us, but nothing is obtained. Then think that the other party is the wrong partner. Will disconnect from the other party
If !gotHeaders &amp;&amp; td.Cmp(d.blockchain.GetTdByHash(d.blockchain.CurrentBlock().Hash())) &gt; 0 {
Return errStallingPeer
}
}
// If fast or light syncing, ensure promised headers are indeed delivered. This is
// needed to detect scenarios where an attacker feeds a bad pivot and then bails out
// of delivering the post-pivot blocks that would flag the invalid content.
//
// This check cannot be executed "as is" for full imports, since blocks may still be
// queued for processing when the header download completes. However, as long as the
// peer gave us something useful, we're already happy/progressed (above check).
If d.mode == FastSync || d.mode == LightSync {
If td.Cmp(d.lightchain.GetTdByHash(d.lightchain.CurrentHeader().Hash())) &gt; 0 {
Return errStallingPeer
}
}
// Disable any rollback and return
Rollback = nil
Return nil
}
// Otherwise split the chunk of headers into batches and process them
gotHeaders = true

For len(headers) &gt; 0 {
// Terminate if something failed in between processing chunks
Select {
Case &lt;-d.cancelCh:
Return errCancelHeaderProcessing
Default:
}
// Select the next chunk of headers to import
Limit := maxHeadersProcess
If limit &gt; len(headers) {
Limit = len(headers)
}
Chunk := headers[:limit]

// In case of header only syncing, validate the chunk immediately
If d.mode == FastSync || d.mode == LightSync { //If it is fast sync mode, or lightweight sync mode (download block header only)
// Collect the yet unknown headers to mark them as uncertain
Unknown := make([]*types.Header, 0, len(headers))
For _, header := range chunk {
If !d.lightchain.HasHeader(header.Hash(), header.Number.Uint64()) {
Unknown = append(unknown, header)
}
}
// If we're importing pure headers, verify based on their recentness
// Verify every few blocks
Frequency := fsHeaderCheckFrequency
If chunk[len(chunk)-1].Number.Uint64()+uint64(fsHeaderForceVerify) &gt; pivot {
Frequency = 1
}
// Lightchain defaults to chain. Insert the block header. If it fails then you need to roll back.
If n, err := d.lightchain.InsertHeaderChain(chunk, frequency); err != nil {
// If some headers were inserted, add them too to the rollback list
If n &gt; 0 {
Rollback = append(rollback, chunk[:n]...)
}
log.Debug("Invalid header encountered", "number", chunk[n].Number, "hash", chunk[n].Hash(), "err", err)
Return errInvalidChain
}
// All verifications passed, store newly found uncertain headers
Rollback = append(rollback, unknown...)
If len(rollback) &gt; fsHeaderSafetyNet {
Rollback = append(rollback[:0], rollback[len(rollback)-fsHeaderSafetyNet:]...)
}
}
// If we're fast syncing and just pulled in the pivot, make sure it's the one locked in
If d.mode == FastSync &amp;&amp; d.fsPivotLock != nil &amp;&amp; chunk[0].Number.Uint64() &lt;= pivot &amp;&amp; chunk[len(chunk)-1].Number.Uint64() &gt;= pivot { // If PivotLock, check if the hash is the same.
If pivot := chunk[int(pivot-chunk[0].Number.Uint64())]; pivot.Hash() != d.fsPivotLock.Hash() {
log.Warn("Pivot doesn't match locked in one", "remoteNumber", pivot.Number, "remoteHash", pivot.Hash(), "localNumber", d.fsPivotLock.Number, "localHash", d.fsPivotLock .Hash())
Return errInvalidChain
}
}
// unless we're doing light chains, schedule the headers for associated content retrieval
// If we deal with the lightweight chain. The header is scheduled to acquire related data. Body,receipts
If d.mode == FullSync || d.mode == FastSync {
// If we've reached the allowed number of pending headers, stall a bit
// If the current queue capacity is not enough. Then wait.
For d.queue.PendingBlocks() &gt;= maxQueuedHeaders || d.queue.PendingReceipts() &gt;= maxQueuedHeaders {
Select {
Case &lt;-d.cancelCh:
Return errCancelHeaderProcessing
Case &lt;-time.After(time.Second):
}
}
// Otherwise insert the headers for content retrieval
/ / Call Queue to schedule, download body and receipts
Inserts := d.queue.Schedule(chunk, origin)
If len(inserts) != len(chunk) {
log.Debug("Stale headers")
Return errBadPeer
}
}
Headers = headers[limit:]
Origin += uint64(limit)
}
// Signal the content downloaders of the availablility of new tasks
// Send a message to the channel d.bodyWakeCh, d.receiptWakeCh to wake up the processing thread.
For _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
Select {
Case ch &lt;- true:
Default:
}
}
}
}
}


## bodies处理
The fetchBodies function defines some closure functions and then calls the fetchParts function.

// fetchBodies iteratively downloads the scheduled block bodies, taking any
// available peers, reserving a chunk of blocks for each, waiting for delivery
// and also periodic checking for timeouts.
// fetchBodies continues to download blocks, using any available links in the middle, leaving a portion of the block for each link, waiting for the block to be delivered, and periodically checking for timeouts.
Func (d *Downloader) fetchBodies(from uint64) error {
log.Debug("Downloading block bodies", "origin", from)

Var (
Deliver = func(packet dataPack) (int, error) { // The delivery function of the downloaded block
Pack := packet.(*bodyPack)
Return d.queue.DeliverBodies(pack.peerId, pack.transactions, pack.uncles)
}
Expire = func() map[string]int { return d.queue.ExpireBodies(d.requestTTL()) } //Timeout
Fetch = func(p *peerConnection, req *fetchRequest) error { return p.FetchBodies(req) } // fetch function
Capacity = func(p *peerConnection) int { return p.BlockCapacity(d.requestRTT()) } // throughput of the peer
setIdle = func(p *peerConnection, accepted int) { p.SetBodiesIdle(accepted) } // Set peer to idle
)
Err := d.fetchParts(errCancelBodyFetch, d.bodyCh, deliver, d.bodyWakeCh, expire,
d.queue.PendingBlocks, d.queue.InFlightBlocks, d.queue.ShouldThrottleBlocks, d.queue.ReserveBodies,
d.bodyFetchHook, fetch, d.queue.CancelBodies, capacity, d.peers.BodyIdlePeers, setIdle, "bodies")

log.Debug("Block body download terminated", "err", err)
Return err
}


fetchParts


// fetchParts iteratively downloads scheduled block parts, taking any available
// peers, reserving a chunk of fetch requests for each, waiting for delivery and
// also unless checking for timeouts.
// fetchParts iteratively downloads the predetermined block portion, takes any available peers, reserves a large number of fetch requests for each part, waits for delivery, and also periodically checks for timeouts.
// As the scheduling/timeout logic mostly is the same for all downloaded data
// types, this method is used by each for data gathering and is instrumented with
// various callbacks to handle the slight differences between processing them.
// Since the dispatch/timeout logic is mostly the same for all downloaded data types, this method is used for downloading of different block types and uses various callback functions to handle the nuances between them.
// The instrumentation parameters:
// - errCancel: error type to return if the fetch operation is cancelled (mostly makes logging nicer) If the fetch operation is canceled, data will be sent on this channel.
// - deliveryCh: channel from which to retrieve downloaded data packets (merged from all concurrent peers) destinations where the data is delivered after the download is completed
// - deliver: processing callback to deliver data packets into type specific download queues (usually within `queue`) to which queue the data is delivered after processing is complete
// - wakeCh: notification channel for waking the fetcher when new tasks are available (or sync completed) to notify fetcher of the arrival of a new task, or to complete the synchronization
// - expire: task callback method to abort requests that took too long and return the faulty peers (traffic shaping) because of the timeout to terminate the request's callback function.
// - pending: task callback for the number of requests still needing download (detect completion/non-completability) The number of tasks that still need to be downloaded.
// - inFlight: task callback for the number of in-progress requests (wait for all active downloads to finish)
// - throttle: task callback to check if the processing queue is full and activate throttling (bound memory use) A callback function used to check if the processing queue is full.
// - reserve: task callback to reserve new download tasks to a particular peer (also signals partial completions) callback function used to schedule a task for a peer
// - fetchHook: tester callback to notify of new tasks being initiated (allows testing the scheduling logic)
// - fetch: network callback to actually send a particular download request to a physical remote peer // send a callback function for the network request
// - cancel: task callback to abort an in-flight download request and allow rescheduling it (in case of lost peer) callback function to cancel the task being processed
// - capacity: network callback to retrieve the estimated type-specific bandwidth capacity of a peer (traffic shaping) network capacity or bandwidth.
// - idle: network callback to retrieve the currently (type specific) idle peers that can be assigned tasks
// - setIdle: network callback to set a peer back to idle and update its estimated capacity (traffic shaping) Set peer to idle callback function
// - kind: textual label of the type being downloaded to display in log mesages download type for log
Func (d *Downloader) fetchParts(errCancel error, deliveryCh chan dataPack, deliver func(dataPack) (int, error), wakeCh chan bool,
Expire func() map[string]int, pending func() int, inFlight func() bool, throttle func() bool, reserve func(*peerConnection, int) (*fetchRequest, bool, error),
fetchHook func([]*types.Header), fetch func(*peerConnection, *fetchRequest) error, cancel func(*fetchRequest), capacity func(*peerConnection) int,
Idle func() ([]*peerConnection, int), setIdle func(*peerConnection, int), kind string) error {

// Create a ticker to detect expired retrieval tasks
Ticker := time.NewTicker(100 * time.Millisecond)
Defer ticker.Stop()

Update := make(chan struct{}, 1)

// Prepare the queue and fetch block parts until the block header fetcher's done
		finished := false
For {
Select {
			case &lt;-d.cancelCh:
				return errCancel

			case packet := &lt;-deliveryCh:
				// If the peer was previously banned and failed to deliver it's pack
				// in a reasonable time frame, ignore it's message.
				// 如果peer在之前被禁止而且没有在合适的时间deliver它的数据，那么忽略这个数据
				if peer := d.peers.Peer(packet.PeerId()); peer != nil {
					// Deliver the received chunk of data and check chain validity
					accepted, err := deliver(packet)
					if err == errInvalidChain {
						return err
}
					// Unless a peer delivered something completely else than requested (usually
					// caused by a timed out request which came through in the end), set it to
					// idle. If the delivery's stale, the peer should have already been idled.
					if err != errStaleDelivery {
						setIdle(peer, accepted)
}
					// Issue a log to the user to see what's going on
					switch {
					case err == nil &amp;&amp; packet.Items() == 0:
						peer.log.Trace("Requested data not delivered", "type", kind)
					case err == nil:
						peer.log.Trace("Delivered new batch of data", "type", kind, "count", packet.Stats())
					default:
						peer.log.Trace("Failed to deliver retrieved data", "type", kind, "err", err)
}
}
				// Blocks assembled, try to update the progress
Select {
				case update &lt;- struct{}{}:
Default:
}

			case cont := &lt;-wakeCh:
				// The header fetcher sent a continuation flag, check if it's done
				// 当所有的任务完成的时候会写入这个队列。
				if !cont {
					finished = true
}
				// Headers arrive, try to update the progress
Select {
				case update &lt;- struct{}{}:
Default:
}

			case &lt;-ticker.C:
				// Sanity check update the progress
Select {
				case update &lt;- struct{}{}:
Default:
}

			case &lt;-update:
				// Short circuit if we lost all our peers
				if d.peers.Len() == 0 {
					return errNoPeers
}
				// Check for fetch request timeouts and demote the responsible peers
				for pid, fails := range expire() {
					if peer := d.peers.Peer(pid); peer != nil {
						// If a lot of retrieval elements expired, we might have overestimated the remote peer or perhaps
						// ourselves. Only reset to minimal throughput but don't drop just yet. If even the minimal times
						// out that sync wise we need to get rid of the peer.
						//如果很多检索元素过期，我们可能高估了远程对象或者我们自己。 只能重置为最小的吞吐量，但不要丢弃。 如果即使最小的同步任然超时，我们需要删除peer。
						// The reason the minimum threshold is 2 is because the downloader tries to estimate the bandwidth
						// and latency of a peer separately, which requires pushing the measures capacity a bit and seeing
						// how response times reacts, to it always requests one more than the minimum (i.e. min 2).
						// 最小阈值为2的原因是因为下载器试图分别估计对等体的带宽和等待时间，这需要稍微推动测量容量并且看到响应时间如何反应，总是要求比最小值（即，最小值2）。
						if fails &gt; 2 {
							peer.log.Trace("Data delivery timed out", "type", kind)
							setIdle(peer, 0)
						} else {
							peer.log.Debug("Stalling delivery, dropping", "type", kind)
							d.dropPeer(pid)
}
}
}
				// If there's nothing more to fetch, wait or terminate
				// 任务全部完成。 那么退出
				if pending() == 0 { //如果没有等待分配的任务， 那么break。不用执行下面的代码了。
					if !inFlight() &amp;&amp; finished {
						log.Debug("Data fetching completed", "type", kind)
						return nil
}
Break
}
				// Send a download request to all idle peers, until throttled
				progressed, throttled, running := false, false, inFlight()
				idles, total := idle()

				for _, peer := range idles {
					// Short circuit if throttling activated
					if throttle() {
						throttled = true
Break
}
					// Short circuit if there is no more available task.
					if pending() == 0 {
Break
}
					// Reserve a chunk of fetches for a peer. A nil can mean either that
					// no more headers are available, or that the peer is known not to
					// have them.
					// 为某个peer请求分配任务。
					request, progress, err := reserve(peer, capacity(peer))
If err != nil {
						return err
}
					if progress {
						progressed = true
}
					if request == nil {
						continue
}
					if request.From &gt; 0 {
						peer.log.Trace("Requesting new batch of data", "type", kind, "from", request.From)
					} else if len(request.Headers) &gt; 0 {
						peer.log.Trace("Requesting new batch of data", "type", kind, "count", len(request.Headers), "from", request.Headers[0].Number)
					} else {
						peer.log.Trace("Requesting new batch of data", "type", kind, "count", len(request.Hashes))
}
					// Fetch the chunk and make sure any errors return the hashes to the queue
					if fetchHook != nil {
						fetchHook(request.Headers)
}
					if err := fetch(peer, request); err != nil {
						// Although we could try and make an attempt to fix this, this error really
						// means that we've double allocated a fetch task to a peer. If that is the
						// case, the internal state of the downloader and the queue is very wrong so
						// better hard crash and note the error instead of silently accumulating into
						// a much bigger issue.
						panic(fmt.Sprintf("%v: %s fetch assignment failed", peer, kind))
}
					running = true
}
				// Make sure that we have peers available for fetching. If all peers have been tried
				// and all failed throw an error
				if !progressed &amp;&amp; !throttled &amp;&amp; !running &amp;&amp; len(idles) == total &amp;&amp; pending() &gt; 0 {
					return errPeersUnavailable
}
}
}
}


## receipt的处理
receipt的处理和body类似。

	// fetchReceipts iteratively downloads the scheduled block receipts, taking any
	// available peers, reserving a chunk of receipts for each, waiting for delivery
	// and also periodically checking for timeouts.
	func (d *Downloader) fetchReceipts(from uint64) error {
		log.Debug("Downloading transaction receipts", "origin", from)

Var (
			deliver = func(packet dataPack) (int, error) {
				pack := packet.(*receiptPack)
				return d.queue.DeliverReceipts(pack.peerId, pack.receipts)
}
			expire   = func() map[string]int { return d.queue.ExpireReceipts(d.requestTTL()) }
			fetch    = func(p *peerConnection, req *fetchRequest) error { return p.FetchReceipts(req) }
			capacity = func(p *peerConnection) int { return p.ReceiptCapacity(d.requestRTT()) }
			setIdle  = func(p *peerConnection, accepted int) { p.SetReceiptsIdle(accepted) }
)
		err := d.fetchParts(errCancelReceiptFetch, d.receiptCh, deliver, d.receiptWakeCh, expire,
			d.queue.PendingReceipts, d.queue.InFlightReceipts, d.queue.ShouldThrottleReceipts, d.queue.ReserveReceipts,
			d.receiptFetchHook, fetch, d.queue.CancelReceipts, capacity, d.peers.ReceiptIdlePeers, setIdle, "receipts")

		log.Debug("Transaction receipt download terminated", "err", err)
Return err
}


## processFastSyncContent 和 processFullSyncContent



// processFastSyncContent takes fetch results from the queue and writes them to the
// database. It also controls the synchronisation of state nodes of the pivot block.
Func (d *Downloader) processFastSyncContent(latest *types.Header) error {
// Start syncing state of the reported head block.
// This should get us most of the state of the pivot block.
		// 启动状态同步
stateSync := d.syncState(latest.Root)
		defer stateSync.Cancel()
Go func() {
			if err := stateSync.Wait(); err != nil {
				d.queue.Close() // wake up WaitResults
}
}()

		pivot := d.queue.FastSyncPivot()
For {
			results := d.queue.WaitResults() // 等待队列输出处理完成的区块
			if len(results) == 0 {
				return stateSync.Cancel()
}
			if d.chainInsertHook != nil {
				d.chainInsertHook(results)
}
			P, beforeP, afterP := splitAroundPivot(pivot, results)
			// 插入fast sync的数据
			if err := d.commitFastSyncData(beforeP, stateSync); err != nil {
Return err
}
			if P != nil {
				// 如果已经达到了 pivot point 那么等待状态同步完成，
				stateSync.Cancel()
				if err := d.commitPivotBlock(P); err != nil {
Return err
}
}
			// 对于pivot point 之后的所有节点，都需要按照完全的处理。
			if err := d.importBlockResults(afterP); err != nil {
Return err
}
}
}

processFullSyncContent,比较简单。 从队列里面获取区块然后插入。

	// processFullSyncContent takes fetch results from the queue and imports them into the chain.
	func (d *Downloader) processFullSyncContent() error {
For {
			results := d.queue.WaitResults()
			if len(results) == 0 {
Return nil
}
			if d.chainInsertHook != nil {
				d.chainInsertHook(results)
}
			if err := d.importBlockResults(results); err != nil {
Return err
}
}
}
</pre></body></html>