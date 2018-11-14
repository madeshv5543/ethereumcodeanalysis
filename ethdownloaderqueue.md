
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>The queue provides the downloader with the scheduling function and the current limit function. Apply the Schedule/ScheduleSkeleton to request the task to be scheduled, then call the ReserveXXX method to retrieve the scheduled task, and execute it in the thread inside the downloader, and call the DeliverXXX method to send the downloaded data to the queue. Finally, through WaitResults to get the completed tasks. There are also some extra controls for the task in the middle. ExpireXXX is used to control whether the task times out. CancelXXX is used to cancel the task.



## Schedule method
The Schedule call requests to perform download scheduling for some block headers. You can see that after doing some legality checks, insert the task into blockTaskPool, receiptTaskPool, receiptTaskQueue, and receiptTaskPool.
TaskPool is a Map that records whether the header's hash exists. The TaskQueue is a priority queue, and the priority is the negative of the height of the block. The higher the height of the block, the higher the priority, and the function of scheduling small tasks first is realized.

// Schedule adds a set of headers for the download queue for scheduling, returning
// the new headers encountered.
// from indicates the block height of the first element in the headers. The return value returns all received headers
Func (q *queue) Schedule(headers []*types.Header, from uint64) []*types.Header {
q.lock.Lock()
Defer q.lock.Unlock()

// Insert all the headers prioritised by the contained block number
Inserts := make([]*types.Header, 0, len(headers))
For _, header := range headers {
// Make sure chain order is honoured and preserved
Hash := header.Hash()
If header.Number == nil || header.Number.Uint64() != from {
log.Warn("Header broke chain ordering", "number", header.Number, "hash", hash, "expected", from)
Break
}
//headerHead stores the last inserted block header, checking if the current block is the correct link.
If q.headerHead != (common.Hash{}) &amp;&amp; q.headerHead != header.ParentHash {
log.Warn("Header broke chain ancestry", "number", header.Number, "hash", hash)
Break
}
// Make sure no duplicate requests are executed
/ / Check the repetition, here directly continue, it is not from the right.
If _, ok := q.blockTaskPool[hash]; ok {
log.Warn("Header already scheduled for block fetch", "number", header.Number, "hash", hash)
Continue
}
If _, ok := q.receiptTaskPool[hash]; ok {
log.Warn("Header already scheduled for receipt fetch", "number", header.Number, "hash", hash)
Continue
}
// Queue the header for content retrieval
q.blockTaskPool[hash] = header
q.blockTaskQueue.Push(header, -float32(header.Number.Uint64()))

If q.mode == FastSync &amp;&amp; header.Number.Uint64() &lt;= q.fastSyncPivot {
// Fast phase of the fast sync, retrieve receipts too
// If it is fast sync mode, and the block height is also less than the pivot point. Then get the receipt
q.receiptTaskPool[hash] = header
q.receiptTaskQueue.Push(header, -float32(header.Number.Uint64()))
}
Inserts = append(inserts, header)
q.headerHead = hash
From++
}
Return inserts
}


## ReserveXXX
The ReserveXXX method is used to retrieve some tasks from the queue to execute. The goroutine in the downloader will call this method to get some tasks to execute. This method directly calls the reserveHeaders method. All ReserveXXX methods call the reserveHeaders method, except for the parameters passed in.

// ReserveBodies reserves a set of body fetches for the given peer, skipping any
//previous failed downloads. Beside the next batch of needed fetches, it also
// returns a flag whether empty blocks were queued requiring processing.
Func (q *queue) ReserveBodies(p *peerConnection, count int) (*fetchRequest, bool, error) {
isNoop := func(header *types.Header) bool {
Return header.TxHash == types.EmptyRootHash &amp;&amp; header.UncleHash == types.EmptyUncleHash
}
q.lock.Lock()
Defer q.lock.Unlock()

Return q.reserveHeaders(p, count, q.blockTaskPool, q.blockTaskQueue, q.blockPendPool, q.blockDonePool, isNoop)
}

reserveHeaders


// reserveHeaders reserves a set of data download operations for a given peer,
// skipping any previously failed ones. This method is a generic version used
// by the individual special reservation functions.
// reserveHeaders reserves some download operations for the specified peer, skipping any previous errors. This method is called separately by the specified retention method.
// Note, this method expects the queue lock to be already held for writing. The
// reason the lock is not obtained in here is because the parameters already need
// to access the queue, so they already need a lock anyway.
// When this method is called, it is assumed that the lock has been acquired. The reason why there is no lock in this method is that the parameter has been passed to the function, so the lock needs to be acquired when calling.
Func (q *queue) reserveHeaders(p *peerConnection, count int, taskPool map[common.Hash]*types.Header, taskQueue *prque.Prque,
pendPool map[string]*fetchRequest, donePool map[common.Hash]struct{}, isNoop func(*types.Header) bool) (*fetchRequest, bool, error) {
// Short circuit if the pool has been depleted, or if the peer's already
// downloading something (sanity check not to corrupt state)
If taskQueue.Empty() {
Return nil, false, nil
}
// If the peer has a download task that is not completed.
If _, ok := pendPool[p.id]; ok {
Return nil, false, nil
}
// Calculate an upper limit on the items we might fetch (i.e. throttling)
// Calculate the upper limit we need to get.
Space := len(q.resultCache) - len(donePool)
// Also need to subtract the number being downloaded.
For _, request := range pendPool {
Space -= len(request.Headers)
}
// Retrieve a batch of tasks, skipping previously failed ones
Send := make([]*types.Header, 0, count)
Skip := make([]*types.Header, 0)

Progress := false
For proc := 0; proc &lt; space &amp;&amp; len(send) &lt; count &amp;&amp; !taskQueue.Empty(); proc++ {
Header := taskQueue.PopItem().(*types.Header)

// If we're the first to request this task, initialise the result container
Index := int(header.Number.Int64() - int64(q.resultOffset))
// index is which part of the resultCache the result should be stored in.
If index &gt;= len(q.resultCache) || index &lt; 0 {
common.Report("index issued went beyond available resultCache space")
Return nil, false, errInvalidChain
}
If q.resultCache[index] == nil { // The first schedule It is possible to schedule multiple times. That may be non-empty here.
Components := 1
If q.mode == FastSync &amp;&amp; header.Number.Uint64() &lt;= q.fastSyncPivot {
// If it is a fast sync, then the component that needs to be downloaded also has a receipt receipt
Components = 2
}
q.resultCache[index] = &amp;fetchResult{
Pending: components,
Header: header,
}
}
// If this fetch task is a noop, skip this fetch operation
If isNoop(header) {
// If the header does not contain a transaction, then you don't need to get the block header.
donePool[header.Hash()] = struct{}{}
Delete(taskPool, header.Hash())

Space, proc = space-1, proc-1
q.resultCache[index].Pending--
Progress = true
Continue
}
// Otherwise unless the peer is known not to have the data, add to the retrieve list
// Lacks represents the data before the node explicitly indicates that there is no such hash.
If p.Lacks(header.Hash()) {
Skip = append(skip, header)
} else {
Send = append(send, header)
}
}
// Merge all the skipped headers back
For _, header := range skip {
taskQueue.Push(header, -float32(header.Number.Uint64()))
}
If progress {
// Wake WaitResults, resultCache was modified
// notify WaitResults, resultCache has changed
q.active.Signal()
}
// Assemble and return the block download request
If len(send) == 0 {
Return nil, progress, nil
}
Request := &amp;fetchRequest{
Peer: p,
Headers: send,
Time: time.Now(),
}
pendPool[p.id] = request

Return request, progress, nil
}

ReserveReceipts can be seen similar to ReserveBodys. However, the queue has changed.

// ReserveReceipts reserves a set of receipt fetches for the given peer, skipping
// any previous failed downloads. Beside the next batch of needed fetches, it
// also returns a flag whether empty receipts were queued requiring importing.
Func (q *queue) ReserveReceipts(p *peerConnection, count int) (*fetchRequest, bool, error) {
isNoop := func(header *types.Header) bool {
Return header.ReceiptHash == types.EmptyRootHash
}
q.lock.Lock()
Defer q.lock.Unlock()

Return q.reserveHeaders(p, count, q.receiptTaskPool, q.receiptTaskQueue, q.receiptPendPool, q.receiptDonePool, isNoop)
}



## DeliverXXX
The Deliver method is called after the data has been downloaded.

// DeliverBodies injects a block body retrieval response into the results queue.
// The method returns the number of blocks bodies accepted from the delivery and
// also wakes any threads waiting for data delivery.
// DeliverBodies inserts the return value of a request block into the results queue
/ / This method returns the number of blocks that are delivered, and will wake up the thread waiting for data
Func (q *queue) DeliverBodies(id string, txLists [][]*types.Transaction, uncleLists [][]*types.Header) (int, error) {
q.lock.Lock()
Defer q.lock.Unlock()

Reconstruct := func(header *types.Header, index int, result *fetchResult) error {
If types.DeriveSha(types.Transactions(txLists[index])) != header.TxHash || types.CalcUncleHash(uncleLists[index]) != header.UncleHash {
Return errInvalidBody
}
result.Transactions = txLists[index]
result.Uncles = uncleLists[index]
Return nil
}
Return q.deliver(id, q.blockTaskPool, q.blockTaskQueue, q.blockPendPool, q.blockDonePool, bodyReqTimer, len(txLists), reconstruct)
}

Deliver method

Func (q *queue) deliver(id string, taskPool map[common.Hash]*types.Header, taskQueue *prque.Prque,
pendPool map[string]*fetchRequest, donePool map[common.Hash]struct{}, reqTimer metrics.Timer,
Results int, reconstruct func(header *types.Header, index int, result *fetchResult) error) (int, error) {

// Short circuit if the data was never requested
// Check if the data has never been requested.
Request := pendPool[id]
If request == nil {
Return 0, errNoFetchesPending
}
reqTimer.UpdateSince(request.Time)
Delete(pendPool, id)

// If no data items were retrieved, mark them as unavailable for the origin peer
If results == 0 {
//If the result is empty. Then identify this peer without this data.
For _, header := range request.Headers {
request.Peer.MarkLacking(header.Hash())
}
}
// Assemble each of the results with their headers and retrieved data parts
Var (
Accepted int
Failure error
Useful bool
)
For i, header := range request.Headers {
// Short circuit assembly if no more fetch results are found
If i &gt;= results {
Break
}
// Reconstruct the next result if contents match up
Index := int(header.Number.Int64() - int64(q.resultOffset))
If index &gt;= len(q.resultCache) || index &lt; 0 || q.resultCache[index] == nil {
Failure = errInvalidChain
Break
}
// Call the passed function to build the data
If err := reconstruct(header, i, q.resultCache[index]); err != nil {
Failure = err
Break
}
donePool[header.Hash()] = struct{}{}
q.resultCache[index].Pending--
Useful = true
Accepted++

// Clean up a successful fetch
// Removed from taskPool. Join donePool
request.Headers[i] = nil
Delete(taskPool, header.Hash())
}
// Return all failed or missing fetches to the queue
// All unsuccessful requests join the taskQueue
For _, header := range request.Headers {
If header != nil {
taskQueue.Push(header, -float32(header.Number.Uint64()))
}
}
// Wake up WaitResults
// If the result changes, notify the WaitResults thread to start.
If accepted &gt; 0 {
q.active.Signal()
}
// If none of the data was good, it's a stale delivery
Switch {
Case failure == nil || failure == errInvalidChain:
Return accepted, failure
Case useful:
Return accepted, fmt.Errorf("partial failure: %v", failure)
Default:
Return accepted, errStaleDelivery
}
}


## ExpireXXX and CancelXXX
### ExpireXXX
The ExpireBodies function gets the lock and then directly calls the expire function.

// ExpireBodies checks for in flight block body requests that exceeded a timeout
//allow, canceling them and returning the responsible peers for penalisation.
Func (q *queue) ExpireBodies(timeout time.Duration) map[string]int {
q.lock.Lock()
Defer q.lock.Unlock()

Return q.expire(timeout, q.blockPendPool, q.blockTaskQueue, bodyTimeoutMeter)
}

Expire function,

// expire is the generic check that move expired tasks from a pending pool back
// into a task pool, returning all entities caught with expired tasks.
// expire is a generic check that moves expired tasks from the pending pool back to the task pool, returning all entities that have captured the expired tasks.

Func (q *queue) expire(timeout time.Duration, pendPool map[string]*fetchRequest, taskQueue *prque.Prque, timeoutMeter metrics.Meter) map[string]int {
// Iterate over the expired requests and return each to the queue
Expiries := make(map[string]int)
For id, request := range pendPool {
If time.Since(request.Time) &gt; timeout {
// Update the metrics with the timeout
timeoutMeter.Mark(1)

// Return any non satisfied requests to the pool
If request.From &gt; 0 {
taskQueue.Push(request.From, -float32(request.From))
}
For hash, index := range request.Hashes {
taskQueue.Push(hash, float32(index))
}
For _, header := range request.Headers {
taskQueue.Push(header, -float32(header.Number.Uint64()))
}
// Add the peer to the expiry report along the the number of failed requests
Expirations := len(request.Hashes)
If expirations &lt; len(request.Headers) {
Expirations = len(request.Headers)
}
Expiries[id] = expirations
}
}
// Remove the expired requests from the pending pool
For id := range expiries {
Delete(pendPool, id)
}
Return expiries
}


### CancelXXX
The Cancle function cancels the assigned task and rejoins the task to the task pool.

// CancelBodies aborts a body fetch request, returning all pending headers to the
// task queue.
Func (q *queue) CancelBodies(request *fetchRequest) {
Q.cancel(request, q.blockTaskQueue, q.blockPendPool)
}

// Cancel aborts a fetch request, returning all pending hashes to the task queue.
Func (q *queue) cancel(request *fetchRequest, taskQueue *prque.Prque, pendPool map[string]*fetchRequest) {
q.lock.Lock()
Defer q.lock.Unlock()

If request.From &gt; 0 {
taskQueue.Push(request.From, -float32(request.From))
}
For hash, index := range request.Hashes {
taskQueue.Push(hash, float32(index))
}
For _, header := range request.Headers {
taskQueue.Push(header, -float32(header.Number.Uint64()))
}
Delete(pendPool, request.Peer.id)
}

## ScheduleSkeleton
The Schedule method passes in a header that has been fetched. Schedule(headers []*types.Header, from uint64). The parameter of the ScheduleSkeleton function is a skeleton, and then requests to fill the skeleton. The so-called skeleton means that I first request a block header every 192 blocks, and then pass the returned header to the ScheduleSkeleton. In the Schedule function, only the queue scheduling block and the receipt of the receipt are required. In the ScheduleSkeleton function, the download of the missing block headers needs to be scheduled.

// ScheduleSkeleton adds a batch of header retrieval tasks to the queue to fill
// up an already retrieved header skeleton.
Func (q *queue) ScheduleSkeleton(from uint64, skeleton []*types.Header) {
q.lock.Lock()
Defer q.lock.Unlock()

// No skeleton retrieval can be in progress, fail hard if so (huge implementation bug)
If q.headerResults != nil {
Panic("skeleton assembly already in progress")
}
// Shedule all the header retrieval tasks for the skeleton assembly
// Because this method will not be called when the skeleton is false. So some initialization work is done here.
q.headerTaskPool = make(map[uint64]*types.Header)
q.headerTaskQueue = prque.New()
q.headerPeerMiss = make(map[string]map[uint64]struct{}) // Reset availability to correct invalid chains
q.headerResults = make([]*types.Header, len(skeleton)*MaxHeaderFetch)
q.headerProced = 0
q.headerOffset = from
q.headerContCh = make(chan bool, 1)

For i, header := range skeleton {
Index := from + uint64(i*MaxHeaderFetch)
// There is a header every MaxHeaderFetch so far
q.headerTaskPool[index] = header
q.headerTaskQueue.Push(index, -float32(index))
}
}

### ReserveHeaders
This method will only be called in the skeleton mode. The task used to reserve the header of the fetch block for the peer.

// ReserveHeaders reserves a set of headers for the given peer, skipping any
// previously failed batches.
Func (q *queue) ReserveHeaders(p *peerConnection, count int) *fetchRequest {
q.lock.Lock()
Defer q.lock.Unlock()

// Short circuit if the peer's already downloading something (sanity check to
// not corrupt state)
If _, ok := q.headerPendPool[p.id]; ok {
Return nil
}
// Retrieve a batch of hashes, skipping previously failed ones
// Get one from the queue and skip the node that failed before.
Send, skip := uint64(0), []uint64{}
For send == 0 &amp;&amp; !q.headerTaskQueue.Empty() {
From, _ := q.headerTaskQueue.Pop()
If q.headerPeerMiss[p.id] != nil {
If _, ok := q.headerPeerMiss[p.id][from.(uint64)]; ok {
Skip = append(skip, from.(uint64))
Continue
}
}
Send = from.(uint64)
}
// Merge all the skipped batches back
For _, from := range skip {
q.headerTaskQueue.Push(from, -float32(from))
}
// Assemble and return the block download request
If send == 0 {
Return nil
}
Request := &amp;fetchRequest{
Peer: p,
From: send,
Time: time.Now(),
}
q.headerPendPool[p.id] = request
Return request
}


### DeliverHeaders


// DeliverHeaders injects a header retrieval response into the header results
// cache. This method either accepts all headers it received, or none of them
// if they do not map correctly to the skeleton.
// This method is either all received for all block headers, or all rejected (if not mapped to a skeleton)
// If the headers are accepted, the method makes an attempt to deliver the set
// of ready headers to the processor to keep the pipeline full. However it will
//not block to prevent stalling other pending deliveries.
// If the block header is received, this method will attempt to deliver them to the headerProcCh pipeline. However, this method does not block delivery. Instead, try to deliver, if you can't deliver it, return.
Func (q *queue) DeliverHeaders(id string, headers []*types.Header, headerProcCh chan []*types.Header) (int, error) {
q.lock.Lock()
Defer q.lock.Unlock()

// Short circuit if the data was never requested
Request := q.headerPendPool[id]
If request == nil {
Return 0, errNoFetchesPending
}
headerReqTimer.UpdateSince(request.Time)
Delete(q.headerPendPool, id)

// Ensure headers can be mapped onto the skeleton chain
Target := q.headerTaskPool[request.From].Hash()

Accepted := len(headers) == MaxHeaderFetch
If accepted { //The length needs to match first, then check if the block number and the hash value of the last block can correspond.
If headers[0].Number.Uint64() != request.From {
log.Trace("First header broke chain ordering", "peer", id, "number", headers[0].Number, "hash", headers[0].Hash(), request.From)
Accepted = false
} else if headers[len(headers)-1].Hash() != target {
log.Trace("Last header broke skeleton structure", "peer", id, "number", headers[len(headers)-1].Number, "hash", headers[len(headers)-1].Hash( ), "expected", target)
Accepted = false
}
}
If accepted {// Check the block number of each block in turn, and whether the link is correct.
For i, header := range headers[1:] {
Hash := header.Hash()
If want := request.From + 1 + uint64(i); header.Number.Uint64() != want {
log.Warn("Header broke chain ordering", "peer", id, "number", header.Number, "hash", hash, "expected", want)
Accepted = false
Break
}
If headers[i].Hash() != header.ParentHash {
log.Warn("Header broke chain ancestry", "peer", id, "number", header.Number, "hash", hash)
Accepted = false
Break
}
}
}
// If the batch of headers wasn't accepted, mark as unavailable
If !accepted { // If it is not received, then mark the peer's failure on this task. The next request will not be delivered to this peer.
log.Trace("Skeleton filling not accepted", "peer", id, "from", request.From)

Miss := q.headerPeerMiss[id]
If miss == nil {
q.headerPeerMiss[id] = make(map[uint64]struct{})
Miss = q.headerPeerMiss[id]
}
Miss[request.From] = struct{}{}

q.headerTaskQueue.Push(request.From, -float32(request.From))
Return 0, errors.New("delivery not accepted")
}
// Clean up a successful fetch and try to deliver any sub-results
Copy(q.headerResults[request.From-q.headerOffset:], headers)
Delete(q.headerTaskPool, request.From)

Ready := 0
For q.headerProced+ready &lt; len(q.headerResults) &amp;&amp; q.headerResults[q.headerProced+ready] != nil {//Calculating this coming header allows the headerResults to have how much data can be delivered.
Ready += MaxHeaderFetch
}
If ready &gt; 0 {
// Headers are ready for delivery, gather them and push forward (non blocking)
Process := make([]*types.Header, ready)
Copy(process, q.headerResults[q.headerProced:q.headerProced+ready])
// try to deliver
Select {
Case headerProcCh &lt;- process:
log.Trace("Pre-scheduled new headers", "peer", id, "count", len(process), "from", process[0].Number)
q.headerProced += len(process)
Default:
}
}
// Check for termination and return
If len(q.headerTaskPool) == 0 {
// This channel is more important. If this channel receives data, all header tasks have been completed.
q.headerContCh &lt;- false
}
Return len(headers), nil
}

RetrieveHeaders, the ScheduleSkeleton function is not called when the last schedule has not been completed. So after the last call is completed, this method will be used to get the result and reset the state.

// RetrieveHeaders retrieves the header chain assemble based on the scheduled
// skeleton.
Func (q *queue) RetrieveHeaders() ([]*types.Header, int) {
q.lock.Lock()
Defer q.lock.Unlock()

Headers, proced := q.headerResults, q.headerProced
q.headerResults, q.headerProced = nil, 0

Return headers, proced
}</pre></body></html>