
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>## chain_indexer Blockchain Index

Chain_indexer.go source code analysis

Chain_indexer, as its name implies, is used to create an index for a blockchain. In the eth protocol, I introduced the function of BloomIndexer. In fact, BloomIndexer is a special implementation of chain_indexer. It can be understood as a derived class. The main function is actually implemented in chain_indexer. Although it is a derived class, chain_indexer is actually only used by BloomIndexer. That is, an index is created for the Bloom filter of the blockchain to quickly respond to the user's log search function. Let's analyze the code in this section.



### data structure

// ChainIndexerBackend defines the methods needed to process chain segments in
// the background and write the segment results into the database. These can be
// used to create filter blooms or CHTs.
// ChainIndexerBackend defines the method for handling blockchain fragments and writes the processing results to the database. These can be used to create Bloom filters or CHTs.
// BloomIndexer actually implements this interface ChainIndexerBackend The CHTs here don't know what it is.
Type ChainIndexerBackend interface {
// Reset initiates the processing of a new chain segment, potentially terminating
// any partially completed operations (in case of a reorg).
// The Reset method is used to initialize a new blockchain fragment and may terminate any unfinished operations.
Reset(section uint64)

// Process crunches through the next header in the chain segment. The caller
// will ensure a sequential order of headers.
// Process the next block header in the blockchain fragment. The caller will ensure the sequential order of the block headers.
Process(header *types.Header)

// Commit finalizes the section metadata and stores it into the database.
Complete the metadata of the blockchain fragment and store it in the database.
Commit() error
}

// ChainIndexer does a post-processing job for equally sized sections of the
// canonical chain (like BlooomBits and CHT structures). A ChainIndexer is
// connected to the blockchain through the event system by starting a
// ChainEventLoop in a goroutine.
// ChainIndexer performs an equal-sized segment of the blockchain. ChainIndexer communicates with the blockchain through the event system in the ChainEventLoop method.
// Further child ChainIndexers can be added which use the output of the parent
// section indexer. These child indexers receive new head notifications only
// after an entire section has been finished or in case of rollbacks that might
//Affected already finished sections.
//More can add more subchained indexers that use the output of the parent section indexer. These sub-chained indexers receive new header notifications only after the entire portion has been completed or where it may affect the rollback of the completed portion.

Type ChainIndexer struct {
chainDb ethdb.Database // Chain database to index the data from the database where the blockchain is located
indexDb ethdb.Database // Prefixed table-view of the db to write index metadata into index stored database
Backend ChainIndexerBackend // Background processor generating the index data content The back end of the index generation.
Children []*ChainIndexer // Child indexers to cascade chain updates to subindex

Active uint32 // Flag whether the event loop was started
Update chan struct{} // Notification channel that headers should be processed
Quit chan chan error // Quit channel to tear down running goroutines

sectionSize uint64 // Number of blocks in a single chain segment to the size of the process section. The default is 4096 blocks for a section
confirmsReq uint64 // Number of confirmations before processing a completed segment Number of confirmations before processing the completed segment

storedSections uint64 // Number of sections successfully indexed into the database
knownSections uint64 // Number of sections known to be complete (block wise)
cascadedHead uint64 // Block number of the last completed section cascaded to subindexers The block number cascaded to the last completed part of the subindex

Throttling time.Duration // Disk throttling to prevent a heavy upgrade from hogging resources disk limit to prevent massive upgrades of large amounts of resources

Log log.Logger
Lock sync.RWMutex
}


Constructor NewChainIndexer,

This method is called inside eth/bloombits.go
Const (
// bloomConfirms is the number of confirmation blocks before a bloom section is
// studied probably final and its rotated bits are calculated.
// bloomConfirms is used to indicate the number of blocks, indicating that after so many blocks, the bloom section is considered to have not changed.
bloomConfirms = 256

// bloomThrottling is the time to wait between processing two consecutive index
// sections. It's useful during chain upgrades to prevent disk overload.
// bloomThrottling is the wait time between processing two consecutive index segments. It is useful to prevent disk overload during blockchain upgrades.
bloomThrottling = 100 * time.Millisecond
)

Func NewBloomIndexer(db ethdb.Database, size uint64) *core.ChainIndexer {
Backend := &amp;BloomIndexer{
Db: db,
Size: size,
}
// You can see that indexDb and chainDb are actually the same database, but each key of indexDb is prefixed with a BloomBitsIndexPrefix.
Table := ethdb.NewTable(db, string(core.BloomBitsIndexPrefix))

Return core.NewChainIndexer(db, table, backend, size, bloomConfirms, bloomThrottling, "bloombits")
}


// NewChainIndexer creates a new chain indexer to do background processing on
// chain segments of a given size after certain number of confirmations passed.
// The throttling parameter might be used to prevent database thrashing.

Func NewChainIndexer(chainDb, indexDb ethdb.Database, backend ChainIndexerBackend, section, confirm uint64, throttling time.Duration, kind string) *ChainIndexer {
c := &amp;ChainIndexer{
chainDb: chainDb,
indexDb: indexDb,
Backend: backend,
Update: make(chan struct{}, 1),
Quit: make(chan chan error),
sectionSize: section,
confirmsReq: confirm,
Throttling: throttling,
Log: log.New("type", kind),
}
// Initialize database dependent fields and start the updater
c.loadValidSections()
Go c.updateLoop()

Return c
}

loadValidSections, used to load our previous processing information from the database, storedSections indicates where we have processed.

// loadValidSections reads the number of valid sections from the index database
// and caches is into the local state.
Func (c *ChainIndexer) loadValidSections() {
Data, _ := c.indexDb.Get([]byte("count"))
If len(data) == 8 {
c.storedSections = binary.BigEndian.Uint64(data[:])
}
}


updateLoop, the main event loop, is used to call backend to handle the blockchain section. It should be noted that all the main index nodes and all child indexers will start the goroutine method.

Func (c *ChainIndexer) updateLoop() {
Var (
Updating bool
Updated time.Time
)
For {
Select {
Case errc := &lt;-c.quit:
// Chain indexer terminating, report no failure and abort
Errc &lt;- nil
Return

Case &lt;-c.update: //When you need to use backend processing, other goroutines will send messages to this channel.
// Section headers completed (or rolled back), update the index
c.lock.Lock()
If c.knownSections &gt; c.storedSections { // If the currently known Section is larger than the already stored Section
// Periodically print an upgrade log message to the user
// Print the log information every 8 seconds.
If time.Since(updated) &gt; 8*time.Second {
If c.knownSections &gt; c.storedSections+1 {
Updating = true
c.log.Info("Upgrading chain index", "percentage", c.storedSections*100/c.knownSections)
}
Updated = time.Now()
}
// Cache the current section count and head to allow unlocking the mutex
Section := c.storedSections
Var oldHead common.Hash
If section &gt; 0 { // section - 1 The subscript representing the section starts at 0.
// sectionHead is used to get the hash value of the last block of the section.
oldHead = c.sectionHead(section - 1)
}
// Process the newly defined section in the background
c.lock.Unlock()
// processing returns the hash value of the last block of the new section
newHead, err := c.processSection(section, oldHead)
If err != nil {
c.log.Error("Section processing failed", "error", err)
}
c.lock.Lock()

// If throughput succeeded and no reorgs occcurred, mark the section completed
If err == nil &amp;&amp; oldHead == c.sectionHead(section-1) {
c.setSectionHead(section, newHead) // Update the state of the database
c.setValidSections(section + 1) // Update database status
If c.storedSections == c.knownSections &amp;&amp; updating {
Updating = false
c.log.Info("Finished upgrading chain index")
}
// cascadedHead is the height of the last block of the updated section
// What is the usage?
c.cascadedHead = c.storedSections*c.sectionSize - 1
For _, child := range c.children {
c.log.Trace("Cascading chain index update", "head", c.cascadedHead)
child.newHead(c.cascadedHead, false)
}
} else { // If the processing fails, it will not be retried until there is a new notification.
// If processing failed, don't retry until further notification
c.log.Debug("Chain index processing failed", "section", section, "err", err)
c.knownSections = c.storedSections
}
}
// If there are still further sections to process, reschedule
// If there are still sections waiting to be processed, wait for the throttling time to process. Avoid disk overload.
If c.knownSections &gt; c.storedSections {
time.AfterFunc(c.throttling, func() {
Select {
Case c.update &lt;- struct{}{}:
Default:
}
})
}
c.lock.Unlock()
}
}
}


Start method. This method is called when the eth protocol is started. This method receives two parameters, one is the current block header, and the other is the event subscriber, through which the blockchain change information can be obtained.

eth.bloomIndexer.Start(eth.blockchain.CurrentHeader(), eth.blockchain.SubscribeChainEvent)

// Start creates a goroutine to feed chain head events into the indexer for
// cascading background processing. Children do not need to be started, they
// are notified about new events by their parents.

// The subchain does not need to be started. Thought their parent node will notify them.
Func (c *ChainIndexer) Start(currentHeader *types.Header, chainEventer func(ch chan&lt;- ChainEvent) event.Subscription) {
Go c.eventLoop(currentHeader, chainEventer)
}

// eventLoop is a secondary - optional - event loop of the indexer which is only
// started for the outermost indexer to push chain head events into a processing
// queue.

// The eventLoop loop will only be called on the outermost index node. All Child indexers will not be started by this method.

Func (c *ChainIndexer) eventLoop(currentHeader *types.Header, chainEventer func(ch chan&lt;- ChainEvent) event.Subscription) {
// Mark the chain indexer as active, requiring an additional teardown
atomic.StoreUint32(&amp;c.active, 1)

Events := make(chan ChainEvent, 10)
Sub := chainEventer(events)
Defer sub.Unsubscribe()

// Fire the initial new head event to start any outstanding processing
// Set our actual block height to trigger previously unfinished operations.
c.newHead(currentHeader.Number.Uint64(), false)

Var (
prevHeader = currentHeader
prevHash = currentHeader.Hash()
)
For {
Select {
Case errc := &lt;-c.quit:
// Chain indexer terminating, report no failure and abort
Errc &lt;- nil
Return

Case ev, ok := &lt;-events:
// Received a new event, ensure it's not nil (closing) and update
If !ok {
Errc := &lt;-c.quit
Errc &lt;- nil
Return
}
Header := ev.Block.Header()
If header.ParentHash != prevHash { //If there is a fork, then we first
// Find a common ancestor, and the index from the public ancestor needs to be rebuilt.
c.newHead(FindCommonAncestor(c.chainDb, prevHeader, header).Number.Uint64(), true)
}
// set a new head
c.newHead(header.Number.Uint64(), false)

prevHeader, prevHash = header, header.Hash()
}
}
}


The newHead method notifies the indexer of the new blockchain header, or needs to rebuild the index, the newHead method will trigger


// newHead notifies the indexer about new chain heads and/or reorgs.
Func (c *ChainIndexer) newHead(head uint64, reorg bool) {
c.lock.Lock()
Defer c.lock.Unlock()

// If a reorg happened, invalidate all sections until that point
If reorg { // Need to rebuild the index All sections starting from head need to be rebuilt.
// Revert the known section number to the reorg point
Changed:= head / c.sectionSize
If changed &lt; c.knownSections {
c.knownSections = changed
}
// Revert the stored sections from the database to the reorg point
// Restore the stored portion from the database to the index rebuild point
If changed &lt; c.storedSections {
c.setValidSections(changed)
}
// Update the new head number to te finalized section end and notify children
// Generate a new head and notify all sub-indexes
Head = changed * c.sectionSize

If head &lt; c.cascadedHead {
c.cascadedHead = head
For _, child := range c.children {
child.newHead(c.cascadedHead, true)
}
}
Return
}
// No reorg, calculate the number of newly known sections and update if high enough
Var sections uint64
If head &gt;= c.confirmsReq {
Sections = (head + 1 - c.confirmsReq) / c.sectionSize
If sections &gt; c.knownSections {
c.knownSections=ties

Select {
Case c.update &lt;- struct{}{}:
Default:
}
}
}
}


Parent-child index data relationship
The listener of the parent Indexer load event then passes the result to the updateLoop of the child Indexer via newHead.

![image](picture/chainindexer_1.png)

The setValidSections method writes the number of sections that are currently stored. If the value passed in is less than the amount already stored, delete the corresponding section from the database.

// setValidSections writes the number of valid sections to the index database
Func (c *ChainIndexer) setValidSections(sections uint64) {
// Set the current number of valid sections in the database
Var data [8]byte
binary.BigEndian.PutUint64(data[:], sections)
c.indexDb.Put([]byte("count"), data[:])

// Remove any reorged sections, caching the valids in the mean time
For c.storedSections &gt; sections {
c.storedSections--
c.removeSectionHead(c.storedSections)
}
c.storedSections = sections // needed if new &gt; old
}


processSection

// processSection processes an entire section by calling backend functions while
// ensuring the continuity of the passed headers. Since the chain mutex is not
//held while processing, the continuity can be broken by a long reorg, in which
// case the function returns with an error.

//processSection handles the entire section by calling the backend function, while ensuring the continuity of the passed header files. Since the link mutex is not maintained during processing, the continuity may be re-interrupted, in which case the function returns an error.
Func (c *ChainIndexer) processSection(section uint64, lastHead common.Hash) (common.Hash, error) {
c.log.Trace("Processing new chain section", "section", section)

// Reset and partial processing
c.backend.Reset(section)

For number := section * c.sectionSize; number &lt; (section+1)*c.sectionSize; number++ {
Hash := GetCanonicalHash(c.chainDb, number)
If hash == (common.Hash{}) {
Return common.Hash{}, fmt.Errorf("canonical block #%d unknown", number)
}
Header := GetHeader(c.chainDb, hash, number)
If header == nil {
Return common.Hash{}, fmt.Errorf("block #%d [%x...] not found", number, hash[:4])
} else if header.ParentHash != lastHead {
Return common.Hash{}, fmt.Errorf("chain reorged during section processing")
}
c.backend.Process(header)
lastHead = header.Hash()
}
If err := c.backend.Commit(); err != nil {
c.log.Error("Section commit failed", "error", err)
Return common.Hash{}, err
}
Return lastHead, nil
}
</pre></body></html>