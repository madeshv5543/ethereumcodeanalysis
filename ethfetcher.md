
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Fetcher contains synchronization based on block notifications. When we received the NewBlockHashesMsg message, we only received a lot of Block hash values. The hash value needs to be used to synchronize the block and then update the local blockchain. Fetcher provides such a feature.


data structure

// announce is the hash notification of the availability of a new block in the
// network.
// announce is a hash notification indicating that a suitable new block appears on the network.
Type announce struct {
Hash common.Hash // Hash of the block being announced // The hash value of the new block
Number uint64 // Number of the block being announced (0 = unknown | old protocol) The height value of the block,
Header *types.Header // Header of the block partially reassembled (new protocol)
Time time.Time // Timestamp of the announcement

Origin string // Identifier of the peer originating the notification

fetchHeader headerRequesterFn // Fetcher function to retrieve the header of an announced block. Get the function pointer of the block header, which contains the information of the peer. That is to find who wants this block head
fetchBodies bodyRequesterFn // Fetcher function to retrieve the body of an announced block
}

// headerFilterTask represents a batch of headers needing fetcher filtering.
Type headerFilterTask struct {
Peer string // The source peer of block headers
Headers []*types.Header // Collection of headers to filter
Time time.Time // Arrival time of the headers
}

// headerFilterTask represents a batch of block bodies (transactions and uncles)
//contact fetcher filtering.
Type bodyFilterTask struct {
Peer string // The source peer of block bodies
Transactions [][]*types.Transaction // Collection of transactions per block bodies
Uncles [][]*types.Header // Collection of uncles per block bodies
Time time.Time // Arrival time of the blocks' contents
}

// inject represents a schedules import operation.
// When the node receives the message NewBlockMsg, it will insert a block.
Type inject struct {
Origin string
Block *types.Block
}

// Fetcher is responsible for accumulating block announcements from various peers
// and scheduling them for retrieval.
Type Fetcher struct {
// Various event channels
Notify chan *announce //announce channel,
Inject chan *inject //inject channel

blockFilter chan chan []*types.Block // channel of the channel?
headerFilter chan chan *headerFilterTask
bodyFilter chan chan *bodyFilterTask

Done chan common.Hash
Quit chan struct{}

// Announce states
Announces map[string]int // Per peer announce counts to prevent memory exhaustion key is the name of the peer, value is the count of announce, in order to avoid memory usage is too large.
Announce map[common.Hash][]*announce // Announced blocks, scheduled for fetching waiting for dispatching fetching announce
Fetching map[common.Hash]*announce // Announced blocks, currently fetching is fetching announce
Fetched map[common.Hash][]*announce // Blocks with headers fetched, scheduled for body retrieval // has acquired the block header, waiting to get the block body
Representation map[common.Hash]*announce // Blocks with headers, currently body-completing //The header and body have already obtained the completed announce

// Block cache
Queue *prque.Prque // Queue containing the import operations (block number sorted) // Queue containing import operations (arranged by block number)
Queues map[string]int // Per peer block counts to prevent memory exhaustion key is peer, value is the number of blocks. Avoid too much memory consumption.
Queued map[common.Hash]*inject // Set of already queued blocks (to dedup imports) Blocks that have been placed in the queue. In order to go heavy.

// Callbacks rely on some callback functions.
getBlock blockRetrievalFn // Retrieves a block from the local chain
verifyHeader headerVerifierFn // Checks if a block's headers have a valid proof of work
broadcastBlock blockBroadcasterFn // Broadcasts a block to connected peers
chainHeight chainHeightFn // Retrieves the current chain's height
insertChain chainInsertFn // Injects a batch of blocks into the chain
dropPeer peerDropFn // Drops a peer for misbehaving

// Testing hooks are for testing purposes only.
announceChangeHook func(common.Hash, bool) // Method to call upon adding or deleting a hash from the announce list
queueChangeHook func(common.Hash, bool) // Method to call upon adding or deleting a block from the import queue
fetchingHook func([]common.Hash) // Method to call upon starting a block (eth/61) or header (eth/62) fetch
stepHook func([]common.Hash) // Method to call upon starting a block body fetch (eth/62)
ImportHook func(*types.Block) // Method to call upon successful block import (both eth/61 and eth/62)
}

Start fetcher and start a goroutine directly. This function is a bit long. Follow-up analysis.

// Start boots up the announcement based synchroniser, accepting and processing
// hash notifications and block fetches until termination requested.
Func (f *Fetcher) Start() {
Go f.loop()
}


The loop function is too long. I will post an ellipsis version first. Fetcher records the state of announce via four maps (announced, fetching, fetched, completing) (waiting for fetch, fetching, fetch waiting for fetch body, fetch is done). The loop actually uses a timer and various messages to perform state transitions on the announcements in various maps.


// Loop is the main fetcher loop, checking and processing various notification
// events.
Func (f *Fetcher) loop() {
// Iterate the block fetching until a quit is requested
fetchTimer := time.NewTimer(0) //Fetch timer.
completeTimer := time.NewTimer(0) // The timer for compelte.

For {
// Clean up any expired block fetches
// If fetching takes more than 5 seconds, then give up this fetching
For hash, announce := range f.fetching {
If time.Since(announce.time) &gt; fetchTimeout {
f.forgetHash(hash)
}
}
// Import any queued blocks that could potentially fit
// This fetcher.queue caches the blocks that have completed fetch and waits to insert them into the local blockchain in order.
//fetcher.queue is a priority queue. The priority level is the negative number of their block number, so that the number of blocks is at the top.
Height := f.chainHeight()
For !f.queue.Empty() { //
Op := f.queue.PopItem().(*inject)
If f.queueChangeHook != nil {
f.queueChangeHook(op.block.Hash(), false)
}
// If too high up the chain or phase, continue later
Number := op.block.NumberU64()
If number &gt; height+1 { //The current block height is too high to import
f.queue.Push(op, -float32(op.block.NumberU64()))
If f.queueChangeHook != nil {
f.queueChangeHook(op.block.Hash(), true)
}
Break
}
// Otherwise if fresh and still unknown, try and import
Hash := op.block.Hash()
If number+maxUncleDist &lt; height || f.getBlock(hash) != nil {
// The height of the block is too low. Below the current height-maxUncleDist
// or the block has been imported
f.forgetBlock(hash)
Continue
}
// insert block
F.insert(op.origin, op.block)
}
// Wait for an outside event to occur
Select {
Case &lt;-f.quit:
// Fetcher terminating, abort all operations
Return

Case notification := &lt;-f.notify: //When NewBlockHashesMsg is received, the hash value of the block that is not yet in the local blockchain will be sent to the notify channel by calling the fetcher's Notify method.
...

Case op := &lt;-f.inject: // When the NewBlockMsg is received, the fetcher Enqueue method is called. This method will send the currently received block to the inject channel.
...
F.enqueue(op.origin, op.block)

Case hash := &lt;-f.done: //When the import of a block is completed, the hash value of the block will be sent to the done channel.
...

Case &lt;-fetchTimer.C: // fetchTimer timer, periodically fetch the block headers that need fetch
...

Case &lt;-completeTimer.C: // The completeTimer timer periodically fetches the block that needs fetch
...

Case filter := &lt;-f.headerFilter: //When a message for BlockHeadersMsg is received (some block headers are received), these messages are posted to the headerFilter queue. This will leave the data belonging to the fetcher request, and the rest will be returned for use by other systems.
...

Case filter := &lt;-f.bodyFilter: //When the BlockBodiesMsg message is received, these messages are posted to the bodyFilter queue. This will leave the data belonging to the fetcher request, and the rest will be returned for use by other systems.
...
}
}
}

### Block header filtering process
#### FilterHeaders Request
The FilterHeaders method is called when a BlockHeadersMsg is received. This method first delivers a channel filter to the headerFilter. Then post a headerFilterTask task to the filter. Then block waiting for the filter queue to return a message.


// FilterHeaders extracts all the headers that were explicitly requested by the fetcher,
// returning those that should be handled differently.
Func (f *Fetcher) FilterHeaders(peer string, headers []*types.Header, time time.Time) []*types.Header {
log.Trace("Filtering headers", "peer", peer, "headers", len(headers))

// Send the filter channel to the fetcher
Filter := make(chan *headerFilterTask)

Select {
Case f.headerFilter &lt;- filter:
Case &lt;-f.quit:
Return nil
}
// Request the filtering of the header list
Select {
Case filter &lt;- &amp;headerFilterTask{peer: peer, headers: headers, time: time}:
Case &lt;-f.quit:
Return nil
}
// Retrieve the headers remaining after filtering
Select {
Case task := &lt;-filter:
Return task.headers
Case &lt;-f.quit:
Return nil
}
}


#### headerFilter processing
This is handled in the gorout of loop().

Case filter := &lt;-f.headerFilter:
// Headers arrived from a remote peer. Extract those that were explicitly
// requested by the fetcher, and return everything else so it's delivered
// to other parts of the system.
Var task *headerFilterTask
Select {
Case task = &lt;-filter:
Case &lt;-f.quit:
Return
}
headerFilterInMeter.Mark(int64(len(task.headers)))

// Split the batch of headers into unknown ones (to return to the caller),
// known incomplete ones (requiring body retrievals) and completed blocks.
Unknown, incomplete, complete := []*types.Header{}, []*announce{}, []*types.Block{}
For _, header := range task.headers {
Hash := header.Hash()

// Filter fetcher-requested headers from other synchronisation algorithms
// See if this is the information returned by our request, depending on the situation.
If announce := f.fetching[hash]; announce != nil &amp;&amp; announce.origin == task.peer &amp;&amp; f.fetched[hash] == nil &amp;&amp; f.completing[hash] == nil &amp;&amp; f.queued[hash ] == nil {
// If the delivered header does not match the promised number, drop the announcer
// If the returned header has a different block height than we requested, then delete the peer that returned the header. And forget about this hash (to regain the block information)
If header.Number.Uint64() != announce.number {
log.Trace("Invalid block number fetched", "peer", announce.origin, "hash", header.Hash(), "announced", announce.number, "provided", header.Number)
f.dropPeer(announce.origin)
f.forgetHash(hash)
Continue
}
// Only keep if not imported by other means
If f.getBlock(hash) == nil {
Announce.header = header
Announce.time = task.time

// If the block is empty (header only), short circuit into the final import queue
// View from the block header if this block does not contain any transactions or Uncle blocks. Then we don't have to get the body of the block. Then insert the completion list directly.
If header.TxHash == types.DeriveSha(types.Transactions{}) &amp;&amp; header.UncleHash == types.CalcUncleHash([]*types.Header{}) {
log.Trace("Block empty, skipping body retrieval", "peer", announce.origin, "number", header.Number, "hash", header.Hash())

Block := types.NewBlockWithHeader(header)
block.ReceivedAt = task.time

Complete = append(complete, block)
F.completing[hash] = announce
Continue
}
// Otherwise add to the list of blocks needing completion
// Otherwise, insert into the unfinished list and wait for the fetch blockbody
Incomplete = append(incomplete, announce)
} else {
log.Trace("Block comes imported, discarding header", "peer", announce.origin, "number", header.Number, "hash", header.Hash())
f.forgetHash(hash)
}
} else {
// Fetcher doesn't know about it, add to the return list
// Fetcher doesn't know about this header. Increase to the return list and wait for return.
Unknown = append(unknown, header)
}
}
headerFilterOutMeter.Mark(int64(len(unknown)))
Select {
// Return the returned result.
Case filter &lt;- &amp;headerFilterTask{headers: unknown, time: task.time}:
Case &lt;-f.quit:
Return
}
// Schedule the retrieved headers for body completion
For _, announce := range incomplete {
Hash := announce.header.Hash()
If _, ok := f.completing[hash]; ok { //if it is already done in other places
Continue
}
// Put it waiting for the body to wait for the map to be processed.
F.fetched[hash] = append(f.fetched[hash], announce)
If len(f.fetched) == 1 { //If the fetched map has only one element that has just been added. Then reset the timer.
f.rescheduleComplete(completeTimer)
}
}
// Schedule the header-only blocks for import
// These only the header block is placed in the queue waiting for import
For _, block := range complete {
If announce := f.completing[block.Hash()]; announce != nil {
F.enqueue(announce.origin, block)
}
}


#### BodyFilter Processing
Similar to the above processing.

Case filter := &lt;-f.bodyFilter:
// Block bodies arrived, extract any explicitly requested blocks, return the rest
Var task *bodyFilterTask
Select {
Case task = &lt;-filter:
Case &lt;-f.quit:
Return
}
bodyFilterInMeter.Mark(int64(len(task.transactions)))

Blocks := []*types.Block{}
For i := 0; i &lt; len(task.transactions) &amp;&amp; i &lt; len(task.uncles); i++ {
// Match up a body to any possible completion request
Matched := false

For hash, announce := range f.completing {
If f.queued[hash] == nil {
txnHash := types.DeriveSha(types.Transactions(task.transactions[i]))
uncleHash := types.CalcUncleHash(task.uncles[i])

If txnHash == announce.header.TxHash &amp;&amp; uncleHash == announce.header.UncleHash &amp;&amp; announce.origin == task.peer {
// Mark the body matched, reassemble if still unknown
Matched = true

If f.getBlock(hash) == nil {
Block := types.NewBlockWithHeader(announce.header).WithBody(task.transactions[i], task.uncles[i])
block.ReceivedAt = task.time

Blocks = append(blocks, block)
} else {
f.forgetHash(hash)
}
}
}
}
If matched {
Task.transactions = append(task.transactions[:i], task.transactions[i+1:]...)
Task.uncles = append(task.uncles[:i], task.uncles[i+1:]...)
I--
Continue
}
}

bodyFilterOutMeter.Mark(int64(len(task.transactions)))
Select {
Case filter &lt;- task:
Case &lt;-f.quit:
Return
}
// Schedule the retrieved blocks for ordered import
For _, block := range blocks {
If announce := f.completing[block.Hash()]; announce != nil {
F.enqueue(announce.origin, block)
}
}

#### notification processing
When NewBlockHashesMsg is received, the hash value of the block that is not yet in the local blockchain will be sent to the notify channel by calling the fetcher's Notify method.


// Notify announces the fetcher of the potential availability of a new block in
// the network.
Func (f *Fetcher) Notify(peer string, hash common.Hash, number uint64, time time.Time,
headerFetcher headerRequesterFn, bodyFetcher bodyRequesterFn) error {
Block := &amp;announce{
Hash: hash,
Number: number,
Time: time,
Origin: peer,
fetchHeader: headerFetcher,
fetchBodies: bodyFetcher,
}
Select {
Case f.notify &lt;- block:
Return nil
Case &lt;-f.quit:
Return errTerminated
}
}

The processing in the loop, mainly to check and then joined the announced container waiting for timing processing.

Case notification := &lt;-f.notify:
// A block was announced, make sure the peer isn't DOSing us
propAnnounceInMeter.Mark(1)

Count := f.announces[notification.origin] + 1
If count &gt; hashLimit { //hashLimit 256 A remote has only 256 announcements
log.Debug("Peer exceeded outstanding announces", "peer", notification.origin, "limit", hashLimit)
propAnnounceDOSMeter.Mark(1)
Break
}
// If we have a valid block number, check that it's potentially useful
// Check if the potential is useful. According to the block number and the distance of the blockchain in the region, too big and too small is meaningless to us.
If notification.number &gt; 0 {
If dist := int64(notification.number) - int64(f.chainHeight()); dist &lt; -maxUncleDist || dist &gt; maxQueueDist {
log.Debug("Peer discarded announcement", "peer", notification.origin, "number", notification.number, "hash", notification.hash, "distance", dist)
propAnnounceDropMeter.Mark(1)
Break
}
}
// All is well, schedule the announce if block's not yet downloading
// Check if we already exist.
If _, ok := f.fetching[notification.hash]; ok {
Break
}
If _, ok := f.completing[notification.hash]; ok {
Break
}
F.announces[notification.origin] = count
F.announced[notification.hash] = append(f.announced[notification.hash], notification)
If f.announceChangeHook != nil &amp;&amp; len(f.announced[notification.hash]) == 1 {
f.announceChangeHook(notification.hash, true)
}
If len(f.announced) == 1 {
f.rescheduleFetch(fetchTimer)
}

#### Enqueue Processing
The Receiver Enqueue method is called when NewBlockMsg is received. This method will send the currently received block to the inject channel. You can see that this method generates an inject object and sends it to the inject channel.

// Enqueue tries to fill gaps the the fetcher's future import queue.
Func (f *Fetcher) Enqueue(peer string, block *types.Block) error {
Op := &amp;inject{
Origin: peer,
Block: block,
}
Select {
Case f.inject &lt;- op:
Return nil
Case &lt;-f.quit:
Return errTerminated
}
}

Inject channel processing is very simple, directly join the queue to wait for import

Case op := &lt;-f.inject:
// A direct block insertion was requested, try and fill any pending gaps
propBroadcastInMeter.Mark(1)
F.enqueue(op.origin, op.block)

Enqueue

// enqueue schedules a new future import operation, if the block to be imported
// has not yet been seen.
Func (f *Fetcher) enqueue(peer string, block *types.Block) {
Hash := block.Hash()

// Ensure the peer isn't DOSing us
Count := f.queues[peer] + 1
If count &gt; blockLimit { blockLimit 64 If there are too many blocks in the cached counterpart.
log.Debug("Discarded propagated block, exceeded allowance", "peer", peer, "number", block.Number(), "hash", hash, "limit", blockLimit)
propBroadcastDOSMeter.Mark(1)
f.forgetHash(hash)
Return
}
// Discard any past or too distant blocks
// Too far from our blockchain.
If dist := int64(block.NumberU64()) - int64(f.chainHeight()); dist &lt; -maxUncleDist || dist &gt; maxQueueDist {
log.Debug("Discarded propagated block, too far away", "peer", peer, "number", block.Number(), "hash", hash, "distance", dist)
propBroadcastDropMeter.Mark(1)
f.forgetHash(hash)
Return
}
// Schedule the block for future importing
// Insert into the queue.
If _, ok := f.queued[hash]; !ok {
Op := &amp;inject{
Origin: peer,
Block: block,
}
F.queues[peer] = count
F.queued[hash] = op
f.queue.Push(op, -float32(block.NumberU64()))
If f.queueChangeHook != nil {
f.queueChangeHook(op.block.Hash(), true)
}
log.Debug("Queued propagated block", "peer", peer, "number", block.Number(), "hash", hash, "queued", f.queue.Size())
}
}

#### Processing of timers
There are two timers in total. fetchTimer and completeTimer are responsible for getting the block header and getting the block body respectively.

State transition announced --fetchTimer(fetch header)---&gt; fetching --(headerFilter)--&gt; fetched --completeTimer(fetch body)--&gt;completing --(bodyFilter)--&gt; enqueue --task.done- -&gt; forgetHash

Found a problem. The completed container may leak. If a body request for a hash is sent. But the request failed and the other party did not return. At this time the completing container was not cleaned up. Is it possible to cause problems?

Case &lt;-fetchTimer.C:
// At least one block's timer ran out, check for needing retrieval
Request := make(map[string][]common.Hash)

For hash, announces := range f.announced {
// TODO What is the meaning of the time limit here?
// The earliest received announce and passed the arrificTimeout-gatherSlack for such a long time.
If time.Since(announces[0].time) &gt; arriveTimeout-gatherSlack {
// Pick a random peer to retrieve from, reset all others
// announces represent multiple announcements from multiple peers in the same block
Announce := announces[rand.Intn(len(announces))]
f.forgetHash(hash)

// If the block still didn't arrive, queue for fetching
If f.getBlock(hash) == nil {
Request[announce.origin] = append(request[announce.origin], hash)
F.fetching[hash] = announce
}
}
}
// Send out all block header requests
// Send all requests.
For peer, hashes := range request {
log.Trace("Fetching scheduled headers", "peer", peer, "list", hashes)

// Create a closure of the fetch and schedule in on a new thread
fetchHeader, hashes := f.fetching[hashes[0]].fetchHeader, hashes
Go func() {
If f.fetchingHook != nil {
f.fetchingHook(hashes)
}
For _, hash := range hashes {
headerFetchMeter.Mark(1)
fetchHeader(hash) // Suboptimal, but protocol doesn't allow batch header retrievals
}
}()
}
// Schedule the next fetch if blocks are still pending
f.rescheduleFetch(fetchTimer)

Case &lt;-completeTimer.C:
// At least one header's timer ran out, retrieve everything
Request := make(map[string][]common.Hash)

For hash, announces := range f.fetched {
// Pick a random peer to retrieve from, reset all others
Announce := announces[rand.Intn(len(announces))]
f.forgetHash(hash)

// If the block still didn't arrive, queue for completion
If f.getBlock(hash) == nil {
Request[announce.origin] = append(request[announce.origin], hash)
F.completing[hash] = announce
}
}
// Send out all block body requests
For peer, hashes := range request {
log.Trace("Fetching scheduled bodies", "peer", peer, "list", hashes)

// Create a closure of the fetch and schedule in on a new thread
If f.completingHook != nil {
f.completingHook(hashes)
}
bodyFetchMeter.Mark(int64(len(hashes)))
Go f.completing[hashes[0]].fetchBodies(hashes)
}
// Schedule the next fetch if blocks are still pending
f.rescheduleComplete(completeTimer)



#### Some other methods

Fetcher insert method. This method inserts the given block into the local blockchain.

// insert spawns a new goroutine to run a block insertion into the chain. If the
// block's number is at the same height as the current import phase, if updates
// the phase states accordingly.
Func (f *Fetcher) insert(peer string, block *types.Block) {
Hash := block.Hash()

// Run the import on a new thread
log.Debug("Importing propagated block", "peer", peer, "number", block.Number(), "hash", hash)
Go func() {
Defer func() { f.done &lt;- hash }()

// If the parent's unknown, abort insertion
Parent := f.getBlock(block.ParentHash())
If parent == nil {
log.Debug("Unknown parent of propagated block", "peer", peer, "number", block.Number(), "hash", hash, "parent", block.ParentHash())
Return
}
// Quickly validate the header and propagate the block if it passes
// If the block header passes validation, the block is broadcast immediately. NewBlockMsg
Switch err := f.verifyHeader(block.Header()); err {
Case nil:
// All ok, quickly propagate to our peers
propBroadcastOutTimer.UpdateSince(block.ReceivedAt)
Go f.broadcastBlock(block, true)

Case consensus.ErrFutureBlock:
// Weird future block, don't fail, but neither propagate

Default:
// Something went very wrong, drop the peer
log.Debug("Propagated block verification failed", "peer", peer, "number", block.Number(), "hash", hash, "err", err)
f.dropPeer(peer)
Return
}
// Run the actual import and log any issues
If _, err := f.insertChain(types.Blocks{block}); err != nil {
log.Debug("Propagated block import failed", "peer", peer, "number", block.Number(), "hash", hash, "err", err)
Return
}
// If Import succeeded, broadcast the block
// If the insertion is successful, then the broadcast block, the second parameter is false. Then only the hash of the block will be broadcast. NewBlockHashesMsg
propAnnounceOutTimer.UpdateSince(block.ReceivedAt)
Go f.broadcastBlock(block, false)

// Invoke the testing hook if needed
If f.importedHook != nil {
f.importedHook(block)
}
}()
}</pre></body></html>