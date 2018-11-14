
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>## Elongfang Bloom filter

The block header of Ethereum contains an area called logsBloom. This area stores the Bloom filter for all the receipts in the current block, for a total of 2048 bits. That is 256 bytes.

And the receipt of one of our transactions contains a lot of log records. Each log record contains the address of the contract, multiple Topics. There is also a Bloom filter in our receipt, which records all the log information.

![image](picture/bloom_1.png)

If we look at the formal definition of log records in the Yellow Book.

O stands for our log record, Oa stands for the address of the logger, Oto, Ot1 stands for the Topics of the log, and Od stands for time.

![image](picture/bloom_2.png)

Oa is 20 bytes, Ot is 32 bytes, Od is a lot of bytes.

![image](picture/bloom_3.png)


We define a Bloom filter function M to convert a log object into a 256-byte hash.

![image](picture/bloom_4.png)

M3:2045 is a special function that sets three of the 2048 bit bits to one. For the specific method, please refer to the formula below.

![image](picture/bloom_5.png)

For any input value, first ask his KEC output, and then take the value of [0,1] [2,3], [4,5] of the KEC output to modulo 2048, and get three values. These three values ​​are the subscripts that need to be set in the output 2048. That is to say, for any input, if the value of its corresponding three subscripts is not 1, then it is definitely not in this block. If, if the corresponding three bits are all 1, it cannot be stated that it is in this block. This is the characteristic of the Bloom filter.

The Bloom filter in the receipt is the union of the Bloom filter outputs for all logs.

At the same time, the logBloom in the block header is the union of the Bloom filters of all the receipts.

## ChainIndexer and BloomIndexer
I first saw ChainIndexer, I don't really understand what it is. In fact, as you can see from the name, it is the index of Chain. In eth we have seen the BloomIndexer, which is the index of the Bloom filter.

The ability to find the specified Log is provided in our protocol.

The user can find the specified Log, the starting block number, and the ending block number by passing the following parameters, filtering according to the address specified by the contract Addresses, and filtering according to the specified Topics.

// FilterCriteria represents a request to create a new filter.
Type FilterCriteria struct {
FromBlock *big.Int
ToBlock *big.Int
Addresses []common.Address
Topics [][]common.Hash
}

If the interval between the start and end is large, it is inefficient to directly retrieve the logBloom area of ​​each block header in turn. Because each block header is stored separately, it may require a lot of random access to the disk.

So the Ethereum protocol maintains a set of indexes locally to speed up the process.

The general principle is. Each 4096 block is called a Section, and the logBloom in a Section is stored together. For each Section, use a two-dimensional data, A[2048][4096]. The first dimension 2048 represents the length of the bloom filter of 2048 bytes. The second dimension 4096 represents all the blocks in a Section, and each location represents one of the blocks in order.

- A[0][0]=blockchain[section*4096+0].logBloom[0],
- A[0][1]=blockchain[section*4096+1].logBloom[0],
- A[0][4096]=blockchain[section*4096+1].logBloom[0],
- A[1][0]=blockchain[section*4096+0].logBloom[1],
- A[1][1024]=blockchain[section*4096+1024].logBloom[1],
- A[2047][1]=blockchain[section*4096+1].logBloom[2047],

If Section is filled, it will be written as 2048 KV.
![image](picture/bloom_6.png)


## bloombit.go Code Analysis

This code is relatively non-independent. If you look at this code alone, it's a bit confusing, because it only implements some interfaces. The specific processing logic is not here, but in the core. But here I will analyze the information I have mentioned before. Subsequent more detailed logic is analyzed in detail when analyzing the core code.

The service thread startBloomHandlers, this method is to respond to the specific query request, given the specified Section and bit to query from the levelDB and then return. Looking at it alone is a bit confusing. The call to this method is more complicated. It involves a lot of logic in the core. I will not elaborate here first. Until there is this method.

Type Retrieval struct {
Bit uint //Bit value 0-2047 represents the value of which bit you want to get
Sections []uint64 // Those Sections
Bitsets [][]byte // Return value The result of the query.
}
// startBloomHandlers starts a batch of goroutines to accept bloom bit database
//reviews from possibly a range of filters and serving the data to satisfy.
Func (eth *Ethereum) startBloomHandlers() {
For i := 0; i &lt; bloomServiceThreads; i++ {
Go func() {
For {
Select {
Case &lt;-eth.shutdownChan:
Return

Case request := &lt;-eth.bloomRequests: // request is a channel
Task := &lt;-request //Get a task from the channel

task.Bitsets = make([][]byte, len(task.Sections))
For i, section := range task.Sections {
Head := core.GetCanonicalHash(eth.chainDb, (section+1)*params.BloomBitsBlocks-1)
Blob, err := bitutil.DecompressBytes(core.GetBloomBits(eth.chainDb, task.Bit, section, head), int(params.BloomBitsBlocks)/8)
If err != nil {
Panic(err)
}
task.Bitsets[i] = blob
}
Request &lt;- task //return results via request channel
}
}
}()
}
}


### data structure
The process of building the index for the main user of the BloomIndexer object is an interface implementation of core.ChainIndexer, so only some necessary interfaces are implemented. The logic for creating an index is also in core.ChainIndexer.



// BloomIndexer implements a core.ChainIndexer, building up a rotated bloom bits index
// for the Ethereum header bloom filters, permitting blazing fast filtering.
Type BloomIndexer struct {
Size uint64 // section size to generate bloombits for

Db ethdb.Database // database instance to write index data and metadata into
Gen *bloombits.Generator // generator to rotate the bloom bits crating the bloom index

Section uint64 // Section is the section number being processed currently
Head common.Hash // Head is the hash of the last header processed
}

// NewBloomIndexer returns a chain indexer that generate bloom bits data for the
// canonical chain for fast logs filtering.
Func NewBloomIndexer(db ethdb.Database, size uint64) *core.ChainIndexer {
Backend := &amp;BloomIndexer{
Db: db,
Size: size,
}
Table := ethdb.NewTable(db, string(core.BloomBitsIndexPrefix))

Return core.NewChainIndexer(db, table, backend, size, bloomConfirms, bloomThrottling, "bloombits")
}

Reset implements the ChainIndexerBackend method and starts a new section.

// Reset implements core.ChainIndexerBackend, starting a new bloombits index
// section.
Func (b *BloomIndexer) Reset(section uint64) {
Gen, err := bloombits.NewGenerator(uint(b.size))
If err != nil {
Panic(err)
}
B.gen, b.section, b.head = gen, section, common.Hash{}
}

Process implements ChainIndexerBackend, adding a new block header to index

// Process implements core.ChainIndexerBackend, adding a new header's bloom into
// the index.
Func (b *BloomIndexer) Process(header *types.Header) {
b.gen.AddBloom(uint(header.Number.Uint64()-b.section*b.size), header.Bloom)
B.head = header.Hash()
}

The Commit method implements ChainIndexerBackend, persists and writes to the database.

// Commit implements core.ChainIndexerBackend, finalizing the bloom section and
// writing it out into the database.
Func (b *BloomIndexer) Commit() error {
Batch := b.db.NewBatch()

For i := 0; i &lt; types.BloomBitLength; i++ {
Bits, err := b.gen.Bitset(uint(i))
If err != nil {
Return err
}
core.WriteBloomBits(batch, uint(i), b.section, b.head, bitutil.CompressBytes(bits))
}
Return batch.Write()
}

## filter/api.go Source Code Analysis

The eth/filter package contains a function to provide filtering to the user. The user can filter the transaction or block by calling, and then continuously obtain the result. If there is no operation for 5 minutes, the filter will be deleted.


The structure of the filter.

Var (
Deadline = 5 * time.Minute // consider a filter inactive if it has not been polled for within deadline
)

// filter is a helper struct that holds meta information over the filter type
// and associated subscription in the event system.
Type filter struct {
Typ Type // The type of filter, what type of data is filtered
Deadline *time.Timer // filter is inactiv when deadline triggers A timer is triggered when the timer sounds.
Hashes []common.Hash // filtered hash results
Crit FilterCriteria //Filter
Logs [] *types.Log / / filtered log information
s *Subscription // associated subscription in event system The subscriber in the event system.
}

Construction method

// PublicFilterAPI offers support to create and manage filters. This will allow external clients to retrieve various
// information related to the Ethereum protocol such als blocks, transactions and logs.
// PublicFilterAPI is used to create and manage filters. Allow external clients to get some information about the Ethereum protocol, such as block information, transaction information and log information.
Type PublicFilterAPI struct {
Backend backend
Mux *event.TypeMux
Quit chan struct{}
chainDb ethdb.Database
Events *EventSystem
filtersMu sync.Mutex
Filters map[rpc.ID]*filter
}

// NewPublicFilterAPI returns a new PublicFilterAPI instance.
Func NewPublicFilterAPI(backend Backend, lightMode bool) *PublicFilterAPI {
Api := &amp;PublicFilterAPI{
Backend: backend,
Mux: backend.EventMux(),
chainDb: backend.ChainDb(),
Events: NewEventSystem(backend.EventMux(), backend, lightMode),
Filters: make(map[rpc.ID]*filter),
}
Go api.timeoutLoop()

Return api
}

### Timeout check

// timeoutLoop runs every 5 minutes and deletes filters that have not been recently used.
// Tt is started when the api is created.
// Check every 5 minutes. If the filter expires, delete it.
Func (api *PublicFilterAPI) timeoutLoop() {
Ticker := time.NewTicker(5 * time.Minute)
For {
&lt;-ticker.C
api.filtersMu.Lock()
For id, f := range api.filters {
Select {
Case &lt;-f.deadline.C:
f.s.Unsubscribe()
Delete(api.filters, id)
Default:
Continue
}
}
api.filtersMu.Unlock()
}
}



NewPendingTransactionFilter to create a PendingTransactionFilter. This method is used for channels that cannot create long connections (such as HTTP). If a channel that can establish long links (such as WebSocket) can be processed using the send subscription mode provided by rpc, there is no need for a continuous round. Inquire

// NewPendingTransactionFilter creates a filter that fetches pending transaction hashes
// as transactions enter the pending state.
//
// It is part of the filter package because this filter can be used throug the
// `eth_getFilterChanges` polling method that is also used for log filters.
//
// https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_newpendingtransactionfilter
Func (api *PublicFilterAPI) NewPendingTransactionFilter() rpc.ID {
Var (
pendingTxs = make(chan common.Hash)
// subscribe to this message in the event system
pendingTxSub = api.events.SubscribePendingTxEvents(pendingTxs)
)

api.filtersMu.Lock()
Api.filters[pendingTxSub.ID] = &amp;filter{typ: PendingTransactionsSubscription, deadline: time.NewTimer(deadline), hashes: make([]common.Hash, 0), s: pendingTxSub}
api.filtersMu.Unlock()

Go func() {
For {
Select {
Case ph := &lt;-pendingTxs: // Received pendingTxs, stored in the hashes container of the filter.
api.filtersMu.Lock()
If f, found := api.filters[pendingTxSub.ID]; found {
F.hashes = append(f.hashes, ph)
}
api.filtersMu.Unlock()
Case &lt;-pendingTxSub.Err():
api.filtersMu.Lock()
Delete(api.filters, pendingTxSub.ID)
api.filtersMu.Unlock()
Return
}
}
}()

Return pendingTxSub.ID
}

Polling: GetFilterChanges

// GetFilterChanges returns the logs for the filter with the given id since
// last time it was called. This can be used for polling.
// GetFilterChanges is used to return all filtering information for all specified ids from the last call to the present. This can be used for polling.
// For pending transaction and block filters the result is []common.Hash.
// (pending)Log filters return []Log.
// For the pending transaction and block filters, the return result type is []common.Hash. For the pending Log filter, the []Log is returned.
// https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_getfilterchanges
Func (api *PublicFilterAPI) GetFilterChanges(id rpc.ID) (interface{}, error) {
api.filtersMu.Lock()
Defer api.filtersMu.Unlock()

If f, found := api.filters[id]; found {
If !f.deadline.Stop() { // If the timer has been triggered but the filter has not been removed, then we will first receive the value of the timer and then reset the timer.
// timer expired but filter is not yet removed in timeout loop
// receive timer value and reset timer
&lt;-f.deadline.C
}
f.deadline.Reset(deadline)

Switch f.typ {
Case PendingTransactionsSubscription, BlocksSubscription:
Hashes := f.hashes
F.hashes = nil
Return returnHashes(hashes), nil
Case LogsSubscription:
Logs := f.logs
F.logs = nil
Return returnLogs(logs), nil
}
}

Return []interface{}{}, fmt.Errorf("filter not found")
}



For a channel that can establish a long connection, you can directly use the rpc send subscription mode, so that the client can directly receive the filtering information without calling the polling method. You can see that this mode is not added to the filters container, and there is no timeout management. In other words, two modes are supported.

// NewPendingTransactions creates a subscription that is triggered each time a transaction
// enters the transaction pool and was signed from one of the transactions this nodes manages.
Func (api *PublicFilterAPI) NewPendingTransactions(ctx context.Context) (*rpc.Subscription, error) {
Notifier, supported := rpc.NotifierFromContext(ctx)
If !supported {
Return &amp;rpc.Subscription{}, rpc.ErrNotificationsUnsupported
}

rpcSub := notifier.CreateSubscription()

Go func() {
txHashes := make(chan common.Hash)
pendingTxSub := api.events.SubscribePendingTxEvents(txHashes)

For {
Select {
Case h := &lt;-txHashes:
notifier.Notify(rpcSub.ID, h)
Case &lt;-rpcSub.Err():
pendingTxSub.Unsubscribe()
Return
Case &lt;-notifier.Closed():
pendingTxSub.Unsubscribe()
Return
}
}
}()

Return rpcSub, nil
}


The log filtering function filters the logs according to the parameters specified by FilterCriteria, starts the block, ends the block, addresses and Topics, and introduces a new object filter.

// FilterCriteria represents a request to create a new filter.
Type FilterCriteria struct {
FromBlock *big.Int
ToBlock *big.Int
Addresses []common.Address
Topics [][]common.Hash
}

// GetLogs returns logs matching the given argument that are stored within the state.
//
// https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_getlogs
Func (api *PublicFilterAPI) GetLogs(ctx context.Context, crit FilterCriteria) ([]*types.Log, error) {
// Convert the RPC block numbers into internal representations
If crit.FromBlock == nil {
crit.FromBlock = big.NewInt(rpc.LatestBlockNumber.Int64())
}
If crit.ToBlock == nil {
crit.ToBlock = big.NewInt(rpc.LatestBlockNumber.Int64())
}
// Create and run the filter to get all the logs
// Create a Filter object and call filter.Logs
Filter := New(api.backend, crit.FromBlock.Int64(), crit.ToBlock.Int64(), crit.Addresses, crit.Topics)

Logs, err := filter.Logs(ctx)
If err != nil {
Return nil, err
}
Return returnLogs(logs), err
}


## filter.go
There is a Filter object defined in fiter.go. This object is mainly used to perform log filtering based on the block's BloomIndexer and Bloom filter.

### data structure
// Backend, this backend is actually implemented in the core. The main algorithm of the Bloom filter is implemented in the core.
Type Backend interface {
ChainDb() ethdb.Database
EventMux() *event.TypeMux
HeaderByNumber(ctx context.Context, blockNr rpc.BlockNumber) (*types.Header, error)
GetReceipts(ctx context.Context, blockHash common.Hash) (types.Receipts, error)

SubscribeTxPreEvent(chan&lt;- core.TxPreEvent) event.Subscription
SubscribeChainEvent(ch chan&lt;- core.ChainEvent) event.Subscription
SubscribeRemovedLogsEvent(ch chan&lt;- core.RemovedLogsEvent) event.Subscription
SubscribeLogsEvent(ch chan&lt;- []*types.Log) event.Subscription

BloomStatus() (uint64, uint64)
ServiceFilter(ctx context.Context, session *bloombits.MatcherSession)
}

// Filter can be used to retrieve and filter logs.
Type Filter struct {
Backend Backend // backend

Db ethdb.Database // database
Begin, end int64 // start to end the block
Addresses []common.Address // Filter the address
Topics [][]common.Hash // Filter topics

Matcher *bloombits.Matcher // Bloom filter matcher
}

The constructor adds both address and topic to the filters container. Then build a bloombits.NewMatcher(size, filters). This function is implemented in the core and will not be explained for the time being.

// new creates a new filter which uses a bloom filter on blocks to figure out whether
// a particular block is interesting or not.
Func New(backend Backend, begin, end int64, addresses []common.Address, topics [][]common.Hash) *Filter {
// Flatten the address and topic filter clauses into a single bloombits filter
// system. Since the bloombits are not positional, nil topics are permitted,
// which get flattened into a nil byte slice.
Var filters [][][]byte
If len(addresses) &gt; 0 {
Filter := make([][]byte, len(addresses))
For i, address := range addresses {
Filter[i] = address.Bytes()
}
Filters = append(filters, filter)
}
For _, topicList := range topics {
Filter := make([][]byte, len(topicList))
For i, topic := range topicList {
Filter[i] = topic.Bytes()
}
Filters = append(filters, filter)
}
// Assemble and return the filter
Size, _ := backend.BloomStatus()

Return &amp;Filter{
Backend: backend,
Begin: begin,
End: end,
Addresses: addresses,
Topics: topics,
Db: backend.ChainDb(),
Matcher: bloombits.NewMatcher(size, filters),
}
}


Logs performs filtering

// Logs searches the blockchain for matching log entries, returning all from the
// first block that contains matches, updating the start of the filter accordingly.
Func (f *Filter) Logs(ctx context.Context) ([]*types.Log, error) {
// Figure out the limits of the filter range
Header, _ := f.backend.HeaderByNumber(ctx, rpc.LatestBlockNumber)
If header == nil {
Return nil, nil
}
Head := header.Number.Uint64()

If f.begin == -1 {
F.begin = int64(head)
}
End := uint64(f.end)
If f.end == -1 {
End = head
}
// Gather all indexed logs, and finish with non indexed ones
Var (
Logs []*types.Log
Err error
)
Size, sections := f.backend.BloomStatus()
// indexed is the maximum size of the block in which the index was created. If the scope of the filter falls on the part where the index was created.
// Then perform an index search.
If indexed := sections * size; indexed &gt; uint64(f.begin) {
If indexed &gt; end {
Logs, err = f.indexedLogs(ctx, end)
} else {
Logs, err = f.indexedLogs(ctx, indexed-1)
}
If err != nil {
Return logs, err
}
}
// Perform a non-indexed search for the rest.
Rest, err := f.unindexedLogs(ctx, end)
Logs = append(logs, rest...)
Return logs, err
}


Index search

// indexedLogs returns the logs matching the filter criteria based on the bloom
// bits indexed available locally or via the network.
Func (f *Filter) indexedLogs(ctx context.Context, end uint64) ([]*types.Log, error) {
// Create a matcher session and request servicing from the backend
Matches := make(chan uint64, 64)
// start matcher
Session, err := f.matcher.Start(uint64(f.begin), end, matches)
If err != nil {
Return nil, err
}
Defer session.Close(time.Second)
// Perform a filtering service. These are all in the core. The code for the subsequent analysis core will be analyzed.

f.backend.ServiceFilter(ctx, session)

// Iterate over the matches until exhausted or context closed
Var logs []*types.Log

For {
Select {
Case number, ok := &lt;-matches:
// Abort if all matches have been fulfilled
If !ok { // did not receive the value and the channel has been closed
F.begin = int64(end) + 1 //Update the begin. For the following non-indexed search
Return logs, nil
}
// Retrieve the suggested block and pull any truly matching logs
Header, err := f.backend.HeaderByNumber(ctx, rpc.BlockNumber(number))
If header == nil || err != nil {
Return logs, err
}
Found, err := f.checkMatches(ctx, header) //Find matching values
If err != nil {
Return logs, err
}
Logs = append(logs, found...)

Case &lt;-ctx.Done():
Return logs, ctx.Err()
}
}
}

checkMatches, get all the receipts and get all the logs from the receipt. Execute the filterLogs method.

// checkMatches checks if the receipts belonging to the given header contain any log events that
// match the filter criteria. This function is called when the bloom filter signals a potential match.
Func (f *Filter) checkMatches(ctx context.Context, header *types.Header) (logs []*types.Log, err error) {
// Get the logs of the block
Receipts, err := f.backend.GetReceipts(ctx, header.Hash())
If err != nil {
Return nil, err
}
Var unfiltered []*types.Log
For _, receipt := range receipts {
Unfiltered = append(unfiltered, ([]*types.Log)(receipt.Logs)...)
}
Logs = filterLogs(unfiltered, nil, nil, f.addresses, f.topics)
If len(logs) &gt; 0 {
Return logs, nil
}
Return nil, nil
}

filterLogs, this method finds the matching in the given logs. And return.

// filterLogs creates a slice of logs matching the given criteria.
Func filterLogs(logs []*types.Log, fromBlock, toBlock *big.Int, addresses []common.Address, topics [][]common.Hash) []*types.Log {
Var ret []*types.Log
Logs:
For _, log := range logs {
If fromBlock != nil &amp;&amp; fromBlock.Int64() &gt;= 0 &amp;&amp; fromBlock.Uint64() &gt; log.BlockNumber {
Continue
}
If toBlock != nil &amp;&amp; toBlock.Int64() &gt;= 0 &amp;&amp; toBlock.Uint64() &lt; log.BlockNumber {
Continue
}

If len(addresses) &gt; 0 &amp;&amp; !includes(addresses, log.Address) {
Continue
}
// If the to filtered topics is greater than the amount of topics in logs, skip.
If len(topics) &gt; len(log.Topics) {
Continue Logs
}
For i, topics := range topics {
Match := len(topics) == 0 // empty rule set == wildcard
For _, topic := range topics {
If log.Topics[i] == topic {
Match = true
Break
}
}
If !match {
Continue Logs
}
}
Ret = append(ret, log)
}
Return ret
}

unindexedLogs, a non-indexed query that loops through all the blocks. First use the header.Bloom inside the block to see if it is possible. If it exists, use checkMatches to retrieve all matches.

// indexedLogs returns the logs matching the filter criteria based on raw block
// iteration and bloom matching.
Func (f *Filter) unindexedLogs(ctx context.Context, end uint64) ([]*types.Log, error) {
Var logs []*types.Log

For ; f.begin &lt;= int64(end); f.begin++ {
Header, err := f.backend.HeaderByNumber(ctx, rpc.BlockNumber(f.begin))
If header == nil || err != nil {
Return logs, err
}
If bloomFilter(header.Bloom, f.addresses, f.topics) {
Found, err := f.checkMatches(ctx, header)
If err != nil {
Return logs, err
}
Logs = append(logs, found...)
}
}
Return logs, nil
}

## to sum up
The filter source package mainly implements two functions.
- Provides a filter RPC that publishes subscription patterns. Used to provide real-time filtering of transactions, blocks, logs, etc. to the rpc client.
- Provides a log filtering mode based on bloomIndexer, which allows you to quickly perform Bloom filtering on a large number of blocks. Filtering operations for historical logs are also provided.</pre></body></html>