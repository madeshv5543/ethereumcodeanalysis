
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>From the test case, the main function points of blockchain are as follows.

Import.
2. The function of GetLastBlock.
3. If there are multiple blockchains, you can select the one with the most difficulty as the canonical blockchain.
4. BadHashes can manually disable the hash value of some blocks. In blocks.go.
5. If BadHashes is newly configured, then when the block is started, it will be automatically disabled and enter the valid state.
6. The wrong nonce will be rejected.
7. Support for Fast importing.
8. Light vs Fast vs Full processing has the same effect on the processing block header.

It can be seen that the main function of the blockchain is to maintain the state of the blockchain, including block verification, insertion and status query.

Glossary:

What is the canonical blockchain?

Because in the process of creating the block, some forks may be generated in a short time. In our database, it is actually a block tree. We think that one of the most difficult paths is considered to be our specification. Blockchain. There are many blocks that can form a blockchain, but not a canonical blockchain.

Database structure:

Is the hash value of the block the same as the hash value of the block header? The hash value of the so-called block is actually the block value of the Header.
// key -&gt; value
// + represents the connection

"LastHeader" The latest block header used in HeaderChain
"LastBlock" is the latest block header used in BlockChain
"LastFast" latest fast sync block header

"h"+num+"n" -&gt; hash is used to store the height of the canonical blockchain and the hash value of the block header

"h" + num + hash -&gt; header height + hash value -&gt; block header

"h" + num + hash + "t" -&gt; td height + hash value -&gt; total difficulty

"H" + hash -&gt; num block hash -&gt; height

"b" + num + hash -&gt; block body height + hash value -&gt; block body

"r" + num + hash -&gt; block receipts height + hash value -&gt; block receipt

"l" + hash -&gt; transaction/receipt lookup metadata

Key | value|description|insert|delete|
---- | --- |---------------|-----------|-----------|
"LastHeader" | hash | The latest block header used in HeaderChain | When a block is considered to be the latest block of a current specification block | When there is an updated blockchain or a forked sibling block chain replacement It
"LastBlock" | hash | The latest block header used in BlockChain | When a block is considered to be the current latest specification of a blockchain | When there is an updated blockchain or a forked sibling blockchain replacement It
"LastFast" | hash | The latest block header used in BlockChain | When the block is considered to be the current latest specification of the blockchain | When there is an updated blockchain or a forked sibling block chain replaced it
"h"+num+"n"|hash|The height of the blockchain used to store the specification and the hash value of the block header are used in HeaderChain|When the block is in the canonical blockchain|When the block is not in the canonical area Blockchain
"h" + num + hash + "t"|td|Total difficulty|WriteBlockAndState is called after the validation and execution of a block (whether or not it is normal) |SetHead method. This method will only be called in two cases. 1 is that the current blockchain contains badhashs, and all blocks starting from badhashs need to be deleted. 2. The current block is in the wrong state and needs to be reset to genesis.
"H" + hash | num| Height of the block Use |WriteBlockAndState in HeaderChain to be called after |SetHead after verifying and executing a block, same as above
"b" + num + hash|block body| block data | WriteBlockAndState or InsertReceiptChain|SetHead is deleted, ibid.
"r" + num + hash|block receipts|block receipt|WriteBlockAndState or InsertReceiptChain|same as above
"l" + txHash | {hash,num,TxIndex|transaction hash can find blocks and transactions | when blocks are added to the canonical blockchain | when blocks are removed from the canonical blockchain



data structure


// BlockChain represents the canonical chain given a database with a genesis
// block. The Blockchain manages chain imports, reverts, chain reorganisations.
// BlockChain represents a canonical chain specified by a database containing the creation block. BlockChain manages the chain insertion, restoration, reconstruction, etc.
// Importing blocks in to the block chain happens according to the set of rules
//defined by the two stage Validator. Processing of blocks is done using the
// Processor which processes the included transaction. The validation of the state
// is done in the second part of the Validator. Failing results in aborting of
// the import.
// Inserting a block requires a two-stage validator specified by a specified set of rules.
// Use the Processor to process the block transaction. The verification of the state is the second stage of the validator. The error will cause the insertion to terminate.
// The BlockChain also helps in returning blocks from **any** chain included
// in the database as well as blocks that represents the canonical chain. It's
// important to note that GetBlock can return any block and does not need to be
//include in the canonical one where as GetBlockByNumber always represents the
// canonical chain.
// Note that GetBlock may return any block that is not in the current canonical blockchain.
// But GetBlockByNumber always returns the block in the current canonical blockchain.
Type BlockChain struct {
Config *params.ChainConfig // chain &amp; network configuration

Hc *HeaderChain // only contains the blockchain of the block header
chainDb ethdb.Database // the underlying database
rmLogsFeed event.Feed // The following are the components of many message notifications
chainFeed event.Feed
chainSideFeed event.Feed
chainHeadFeed event.Feed
logsFeed event.Feed
Scope event.SubscriptionScope
genesisBlock *types.Block // Creation Block

Mu sync.RWMutex // global mutex for locking chain operations
Chainmu sync.RWMutex // blockchain insertion lock
Procmu sync.RWMutex // block processor lock

Checkpoint int // checkpoint counts towards the new checkpoint
currentBlock *types.Block // Current head of the block chain
currentFastBlock *types.Block // Current head of the fast-sync chain (may be above the block chain!) The current fast sync block header.

stateCache state.Database // State database to reuse between imports (contains state cache)
bodyCache *lru.Cache // Cache for the most recent block bodies
bodyRLPCache *lru.Cache // Cache for the most recent block bodies in RLP encoded format
blockCache *lru.Cache // Cache for the most recent entire blocks
futureBlocks *lru.Cache // future blocks are blocks added for later processing The block storage location that can not be inserted temporarily.

Quit chan struct{} // blockchain quit channel
Running int32 // running must be called atomically
// procInterrupt must be atomically called
procInterrupt int32 // interrupt signaler for block processing
Wg sync.WaitGroup // chain processing wait group for shutting down

Engine consensus.Engine // consistency engine
Processor Processor // block processor interface // block processor interface
Validator Validator // block and state validator interface // block and state validator interface
vmConfig vm.Config / / virtual machine configuration

badBlocks *lru.Cache // Bad block cache The buffer of the error block.
}



Construction, NewBlockChain constructs an initialized blockchain using the information available in the database. Initializes the Ethereum default validator and processor (Validator and Processor)


// NewBlockChain returns a fully initialised block chain using information
// available in the database. It initialises the default Ethereum Validator and
// Processor.
Func NewBlockChain(chainDb ethdb.Database, config *params.ChainConfig, engine consensus.Engine, vmConfig vm.Config) (*BlockChain, error) {
bodyCache, _ := lru.New(bodyCacheLimit)
bodyRLPCache, _ := lru.New(bodyCacheLimit)
blockCache, _ := lru.New(blockCacheLimit)
futureBlocks, _ := lru.New(maxFutureBlocks)
badBlocks, _ := lru.New(badBlockLimit)

Bc := &amp;BlockChain{
Config: config,
chainDb: chainDb,
stateCache: state.NewDatabase(chainDb),
Quit: make(chan struct{}),
bodyCache: bodyCache,
bodyRLPCache: bodyRLPCache,
blockCache: blockCache,
futureBlocks: futureBlocks,
Engine: engine,
vmConfig: vmConfig,
badBlocks: badBlocks,
}
bc.SetValidator(NewBlockValidator(config, bc, engine))
bc.SetProcessor(NewStateProcessor(config, bc, engine))

Var err error
Bc.hc, err = NewHeaderChain(chainDb, config, engine, bc.getProcInterrupt)
If err != nil {
Return nil, err
}
bc.genesisBlock = bc.GetBlockByNumber(0) // Get the creation block
If bc.genesisBlock == nil {
Return nil, ErrNoGenesis
}
If err := bc.loadLastState(); err != nil { //Load the latest state
Return nil, err
}
// Check the current state of the block hashes and make sure that we do not have any of the bad blocks in our chain
// Check the current state and verify that there are no illegal blocks on our blockchain.
// BadHashes are some manually configured block hash values ​​that are used for hard forks.
For hash := range BadHashes {
If header := bc.GetHeaderByHash(hash); header != nil {
// get the canonical block corresponding to the offending header's number
// Get the same height of the block header above the canonical blockchain. If the block header is indeed on our canonical blockchain, we need to roll back to the height of the block header - 1
headerByNumber := bc.GetHeaderByNumber(header.Number.Uint64())
// make sure the headerByNumber (if present) is in our current canonical chain
If headerByNumber != nil &amp;&amp; headerByNumber.Hash() == header.Hash() {
log.Error("Found bad hash, rewinding chain", "number", header.Number, "hash", header.ParentHash)
bc.SetHead(header.Number.Uint64() - 1)
log.Error("Chain rewind was successful, resuming normal operation")
}
}
}
// Take ownership of this particular state
Go bc.update()
Return bc, nil
}

loadLastState, loads the latest blockchain state we know in the database. This method assumes that the lock has been acquired.

// loadLastState loads the last known chain state from the database. This method
// assumes that the chain manager mutex is held.
Func (bc *BlockChain) loadLastState() error {
// Restore the last known head block
// return the hash of the latest block we know
Head := GetHeadBlockHash(bc.chainDb)
If head == (common.Hash{}) { // If the acquisition is empty, then the database is considered corrupted. Then set the blockchain to create a block.
// Corrupt or empty database, init from scratch
log.Warn("Empty database, resetting chain")
Return bc.Reset()
}
// Make sure the entire head block is available
// Find the block based on blockHash
currentBlock := bc.GetBlockByHash(head)
If currentBlock == nil {
// Corrupt or empty database, init from scratch
log.Warn("Head block missing, resetting chain", "hash", head)
Return bc.Reset()
}
// Make sure the state associated with the block is available
// Confirm that the world state of this block is correct.
If _, err := state.New(currentBlock.Root(), bc.stateCache); err != nil {
// Dangling block without a state associated, init from scratch
log.Warn("Head state missing, resetting chain", "number", currentBlock.Number(), "hash", currentBlock.Hash())
Return bc.Reset()
}
// Everything seems to be fine, set as the head block
bc.currentBlock = currentBlock

// Restore the last known head header
// Get the latest hash of the block header
currentHeader := bc.currentBlock.Header()
If head := GetHeadHeaderHash(bc.chainDb); head != (common.Hash{}) {
If header := bc.GetHeaderByHash(head); header != nil {
currentHeader = header
}
}
// header chain is set to the current block header.
bc.hc.SetCurrentHeader(currentHeader)

// Restore the last known head fast block
bc.currentFastBlock = bc.currentBlock
If head := GetHeadFastBlockHash(bc.chainDb); head != (common.Hash{}) {
If block := bc.GetBlockByHash(head); block != nil {
bc.currentFastBlock = block
}
}

// Issue a status log for the user is used to print the log.
headerTd := bc.GetTd(currentHeader.Hash(), currentHeader.Number.Uint64())
blockTd := bc.GetTd(bc.currentBlock.Hash(), bc.currentBlock.NumberU64())
fastTd := bc.GetTd(bc.currentFastBlock.Hash(), bc.currentFastBlock.NumberU64())

log.Info("Loaded most recent local header", "number", currentHeader.Number, "hash", currentHeader.Hash(), "td", headerTd)
log.Info("Loaded most recent local full block", "number", bc.currentBlock.Number(), "hash", bc.currentBlock.Hash(), "td", blockTd)
log.Info("Loaded most recent local fast block", "number", bc.currentFastBlock.Number(), "hash", bc.currentFastBlock.Hash(), "td", fastTd)

Return nil
}

The processing of goroutine update is very simple. Timed processing of future blocks.

Func (bc *BlockChain) update() {
futureTimer := time.Tick(5 * time.Second)
For {
Select {
Case &lt;-futureTimer:
bc.procFutureBlocks()
Case &lt;-bc.quit:
Return
}
}
}

Reset method Reset the blockchain.

// Reset purges the entire blockchain, restoring it to its genesis state.
Func (bc *BlockChain) Reset() error {
Return bc.ResetWithGenesisBlock(bc.genesisBlock)
}

// ResetWithGenesisBlock purges the entire blockchain, restoring it to the
// specified genesis state.
Func (bc *BlockChain) ResetWithGenesisBlock(genesis *types.Block) error {
// Dump the entire block chain and purge the caches
// dump the entire blockchain and clear the cache
If err := bc.SetHead(0); err != nil {
Return err
}
bc.mu.Lock()
Defer bc.mu.Unlock()

// Prepare the genesis block and reinitialise the chain
// Reinitialize the blockchain using the creation block
If err := bc.hc.WriteTd(genesis.Hash(), genesis.NumberU64(), genesis.Difficulty()); err != nil {
log.Crit("Failed to write genesis block TD", "err", err)
}
If err := WriteBlock(bc.chainDb, genesis); err != nil {
log.Crit("Failed to write genesis block", "err", err)
}
bc.genesisBlock = genesis
Bc.insert(bc.genesisBlock)
bc.currentBlock = bc.genesisBlock
bc.hc.SetGenesis(bc.genesisBlock.Header())
bc.hc.SetCurrentHeader(bc.genesisBlock.Header())
bc.currentFastBlock = bc.genesisBlock

Return nil
}

SetHead rolls back the local chain to the new header. Everything above a given new header will be deleted and the new header will be set. If the block is lost (non-archive nodes after fast synchronization), the header may be further rewinded.

// SetHead rewinds the local chain to a new head. In the case of headers, everything
// above the new head will be deleted and the new one set. In the case of blocks
// though, the head may be further rewound if block bodies are missing (non-archive
// nodes after a fast sync).
Func (bc *BlockChain) SetHead(head uint64) error {
log.Warn("Rewinding blockchain", "target", head)

bc.mu.Lock()
Defer bc.mu.Unlock()

// Rewind the header chain, deleting all block bodies until then
delFn := func(hash common.Hash, num uint64) {
DeleteBody(bc.chainDb, hash, num)
}
bc.hc.SetHead(head, delFn)
currentHeader := bc.hc.CurrentHeader()

// Clear out any stale content from the caches
bc.bodyCache.Purge()
bc.bodyRLPCache.Purge()
bc.blockCache.Purge()
bc.futureBlocks.Purge()

// Rewind the block chain, ensuring we don't end up with a stateless head block
If bc.currentBlock != nil &amp;&amp; currentHeader.Number.Uint64() &lt; bc.currentBlock.NumberU64() {
bc.currentBlock = bc.GetBlock(currentHeader.Hash(), currentHeader.Number.Uint64())
}
If bc.currentBlock != nil {
If _, err := state.New(bc.currentBlock.Root(), bc.stateCache); err != nil {
// Rewound state missing, rolled back to before pivot, reset to genesis
bc.currentBlock = nil
}
}
// Rewind the fast block in a simpleton way to the target head
If bc.currentFastBlock != nil &amp;&amp; currentHeader.Number.Uint64() &lt; bc.currentFastBlock.NumberU64() {
bc.currentFastBlock = bc.GetBlock(currentHeader.Hash(), currentHeader.Number.Uint64())
}
// If either blocks reached nil, reset to the genesis state
If bc.currentBlock == nil {
bc.currentBlock = bc.genesisBlock
}
If bc.currentFastBlock == nil {
bc.currentFastBlock = bc.genesisBlock
}
If err := WriteHeadBlockHash(bc.chainDb, bc.currentBlock.Hash()); err != nil {
log.Crit("Failed to reset head full block", "err", err)
}
If err := WriteHeadFastBlockHash(bc.chainDb, bc.currentFastBlock.Hash()); err != nil {
log.Crit("Failed to reset head fast block", "err", err)
}
Return bc.loadLastState()
}

InsertChain, insert the blockchain, insert the blockchain to try to insert the given block into the canonical chain, or create a fork. If an error occurs, it will return the index and the specific error message when the error occurred.

// InsertChain attempts to insert the given batch of blocks in to the canonical
// chain or, otherwise, create a fork. If an error is returned it will return
// the index number of the failing block as well an error patterns
// wrong.
//
// After insertion is done, all accumulated events will be fired.
// After the insert is complete, all accumulated events will be triggered.
Func (bc *BlockChain) InsertChain(chain types.Blocks) (int, error) {
n, events, logs, err := bc.insertChain(chain)
bc.PostChainEvents(events, logs)
Return n, err
}

The insertChain method performs blockchain insertion and collects event information. Since defer is required to handle unlocking, this method is treated as a separate method.

// insertChain will execute the actual chain insertion and event aggregation. The
// only reason this method exists as a separate one is to make locking cleaner
// with deferred statements.
Func (bc *BlockChain) insertChain(chain types.Blocks) (int, []interface{}, []*types.Log, error) {
// Do a sanity check that the provided chain is actually ordered and linked
// Make a sound check, the supplied chains are actually ordered and linked to each other
For i := 1; i &lt; len(chain); i++ {
If chain[i].NumberU64() != chain[i-1].NumberU64()+1 || chain[i].ParentHash() != chain[i-1].Hash() {
// Chain broke ancestry, log a messge (programming error) and skip insertion
log.Error("Non contiguous block insert", "number", chain[i].Number(), "hash", chain[i].Hash(),
"parent", chain[i].ParentHash(), "prevnumber", chain[i-1].Number(), "prevhash", chain[i-1].Hash())

Return 0, nil, nil, fmt.Errorf("non contiguous insert: item %d is #%d [%x...], item %d is #%d [%x...] (parent [%x...])" , i-1, chain[i-1].NumberU64(),
Chain[i-1].Hash().Bytes()[:4], i, chain[i].NumberU64(), chain[i].Hash().Bytes()[:4], chain[i ].ParentHash().Bytes()[:4])
}
}
// Pre-checks passed, start the full block imports
bc.wg.Add(1)
Defer bc.wg.Done()

bc.chainmu.Lock()
Defer bc.chainmu.Unlock()

// A queued approach to delivering events. This is generally
// faster than direct delivery and requires much less mutex
// acquiring.
Var (
Stats = insertStats{startTime: mclock.Now()}
Events = make([]interface{}, 0, len(chain))
lastCanon *types.Block
coalescedLogs []*types.Log
)
// Start the parallel header verifier
Headers := make([]*types.Header, len(chain))
Seals := make([]bool, len(chain))

For i, block := range chain {
Headers[i] = block.Header()
Seals[i] = true
}
// Call the consistency engine to verify that the block header is valid.
Abort, results := bc.engine.VerifyHeaders(bc, headers, seals)
Defer close(abort)

// Iterate over the blocks and insert when the verifier permits
For i, block := range chain {
// If the chain is terminating, stop processing blocks
If atomic.LoadInt32(&amp;bc.procInterrupt) == 1 {
log.Debug("Premature abort during blocks processing")
Break
}
// If the header is a banned one, straight out abort
// If the block header is disabled.
If BadHashes[block.Hash()] {
bc.reportBlock(block, nil, ErrBlacklistedHash)
Return i, events, coalescedLogs, ErrBlacklistedHash
}
// Wait for the block's verification to complete
Bstart := time.Now()

Err := &lt;-results
If err == nil { // If there is no error. Verify the body
Err = bc.Validator().ValidateBody(block)
}
If err != nil {
If err == ErrKnownBlock { // If the block has been inserted, continue directly
Stats.ignored++
Continue
}

If err == consensus.ErrFutureBlock {
// Allow up to MaxFuture second in the future blocks. If this limit
// is exceeded the chain is discarded and processed at a later time
// if given.
// If it is a future block, and the time distance of the block is not very long now. Then store it.
Max := big.NewInt(time.Now().Unix() + maxTimeFutureBlocks)
If block.Time().Cmp(max) &gt; 0 {
Return i, events, coalescedLogs, fmt.Errorf("future block: %v &gt; %v", block.Time(), max)
}
bc.futureBlocks.Add(block.Hash(), block)
Stats.queued++
Continue
}

If err == consensus.ErrUnknownAncestor &amp;&amp; bc.futureBlocks.Contains(block.ParentHash()) { If the block does not find the ancestor and the ancestors of this block are included in the future blocks, then it is stored in the future
bc.futureBlocks.Add(block.Hash(), block)
Stats.queued++
Continue
}

bc.reportBlock(block, nil, err)
Return i, events, coalescedLogs, err
}
// Create a new statedb using the parent block and report an
// error if it fails.
Var parent *types.Block
If i == 0 {
Parent = bc.GetBlock(block.ParentHash(), block.NumberU64()-1)
} else {
Parent = chain[i-1]
}
State, err := state.New(parent.Root(), bc.stateCache)
If err != nil {
Return i, events, coalescedLogs, err
}
// Process block using the parent state as reference point.
// Process blocks, generate transactions, receipts, logs, etc.
// Actually called the Process method in state_processor.go.
Receipts, logs, usedGas, err := bc.processor.Process(block, state, bc.vmConfig)
If err != nil {
bc.reportBlock(block, receipts, err)
Return i, events, coalescedLogs, err
}
// Validate the state using the default validator
// Secondary verification, verifying that the status is legal
Err = bc.Validator().ValidateState(block, parent, state, receipts, usedGas)
If err != nil {
bc.reportBlock(block, receipts, err)
Return i, events, coalescedLogs, err
}
// Write the block to the chain and get the status
// Write block and state.
Status, err := bc.WriteBlockAndState(block, receipts, state)
If err != nil {
Return i, events, coalescedLogs, err
}
Switch status {
Case CanonStatTy: // A new block has been inserted.
log.Debug("Inserted new block", "number", block.Number(), "hash", block.Hash(), "uncles", len(block.Uncles()),
"txs", len(block.Transactions()), "gas", block.GasUsed(), "elapsed", common.PrettyDuration(time.Since(bstart)))

coalescedLogs = append(coalescedLogs, logs...)
blockInsertTimer.UpdateSince(bstart)
Events = append(events, ChainEvent{block, block.Hash(), logs})
lastCanon = block

Case SideStatTy: // inserted a forked block
log.Debug("Inserted forked block", "number", block.Number(), "hash", block.Hash(), "diff", block.Difficulty(), "elapsed",
common.PrettyDuration(time.Since(bstart)), "txs", len(block.Transactions()), "gas", block.GasUsed(), "uncles", len(block.Uncles()))

blockInsertTimer.UpdateSince(bstart)
Events = append(events, ChainSideEvent{block})
}
Stats.processed++
stats.usedGas += usedGas.Uint64()
Stats.report(chain, i)
}
// Append a single chain head event if we've progressed the chain
// If we generate a new block header, and the latest block header is equal to lastCanon
// Then we announce a new ChainHeadEvent
If lastCanon != nil &amp;&amp; bc.LastBlockHash() == lastCanon.Hash() {
Events = append(events, ChainHeadEvent{lastCanon})
}
Return 0, events, coalescedLogs, nil
}

WriteBlockAndState, write the block to the blockchain.

// WriteBlock writes the block to the chain.
Func (bc *BlockChain) WriteBlockAndState(block *types.Block, receipts []*types.Receipt, state *state.StateDB) (status WriteStatus, err error) {
bc.wg.Add(1)
Defer bc.wg.Done()

// Calculate the total difficulty of the block
// Calculate the total difficulty of the block to be inserted
Ptd := bc.GetTd(block.ParentHash(), block.NumberU64()-1)
If ptd == nil {
Return NonStatTy, consensus.ErrUnknownAncestor
}
// Make sure no inconsistent state is leaked during insertion
// Make sure there are no inconsistent state leaks during the insert process
bc.mu.Lock()
Defer bc.mu.Unlock()
// Calculate the total difficulty of the blockchain of the current block.
localTd := bc.GetTd(bc.currentBlock.Hash(), bc.currentBlock.NumberU64())
// Calculate the total difficulty of the new blockchain
externTd := new(big.Int).Add(block.Difficulty(), ptd)

// Irrelevant of the canonical status, write the block itself to the database
// The state that has no relationship with the canonical block, written to the database. The hash height of the written block and the corresponding total difficulty.
If err := bc.hc.WriteTd(block.Hash(), block.NumberU64(), externTd); err != nil {
Return NonStatTy, err
}
// Write other block data using a batch.
Batch := bc.chainDb.NewBatch()
If err := WriteBlock(batch, block); err != nil { // Write block
Return NonStatTy, err
}
If _, err := state.CommitTo(batch, bc.config.IsEIP158(block.Number())); err != nil { //Commit
Return NonStatTy, err
}
If err := WriteBlockReceipts(batch, block.Hash(), block.NumberU64(), receipts); err != nil { // Write block receipt
Return NonStatTy, err
}

// If the total difficulty is higher than our known, add it to the canonical chain
// Second clause in the if statement reduces the vulnerability to selfish mining.
// Please refer to http://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf
// If the total difficulty of the new block is higher than our current block, set this block as the canonical block.
// The second expression ((externTd.Cmp(localTd) == 0 &amp;&amp; mrand.Float64() &lt; 0.5))
// is to reduce the possibility of selfish mining.
If externTd.Cmp(localTd) &gt; 0 || (externTd.Cmp(localTd) == 0 &amp;&amp; mrand.Float64() &lt; 0.5) {
// Reorganise the chain if the parent is not the head block
// If the parent block of this block is not the current block, there is a fork. You need to call reorg to reorganize the blockchain.
If block.ParentHash() != bc.currentBlock.Hash() {
If err := bc.reorg(bc.currentBlock, block); err != nil {
Return NonStatTy, err
}
}
// Write the positional metadata for transaction and receipt lookups
// "l" + txHash -&gt; {blockHash,blockNum,txIndex}
// Find the corresponding block and the corresponding transaction based on the hash value of the transaction.
If err := WriteTxLookupEntries(batch, block); err != nil {
Return NonStatTy, err
}
// Write hash preimages
// hash(Keccak-256) -&gt; Corresponding data This function is for testing. If dev mode is enabled,
// or the vmdebug parameter, if you execute the SHA3 command, you will add Preimage
If err := WritePreimages(bc.chainDb, block.NumberU64(), state.Preimages()); err != nil {
Return NonStatTy, err
}
Status = CanonStatTy
} else {
Status = SideStatTy
}
If err := batch.Write(); err != nil {
Return NonStatTy, err
}

// Set new head.
If status == CanonStatTy {
Bc.insert(block)
}
bc.futureBlocks.Remove(block.Hash())
Return status, nil
}


The reorgs method requires a new blockchain to replace the local blockchain as a canonical chain if the total difficulty of the new chain is greater than the total difficulty of the local chain.

// reorgs takes two blocks, an old chain and a new chain and will reconstruct the blocks and inserts them
// to be part of the new canonical chain and accumulates potential missing transactions and post an
// event about them
// reorgs accepts two blocks as arguments, one is the old blockchain, a new blockchain, this method will insert them
// to rebuild a canonical blockchain. At the same time, the potential lost transactions are accumulated and released as events.
Func (bc *BlockChain) reorg(oldBlock, newBlock *types.Block) error {
Var (
newChain types.Blocks
oldChain types.Blocks
commonBlock *types.Block
deletedTxs types.Transactions
deletedLogs []*types.Log
// collectLogs collects the logs that were generated during the
// processing of the block that corresponds with the given hash.
// These logs are later announced as deleted.
// collectLogs will collect the log information we have generated, which will be deleted later (actually not deleted in the database).
collectLogs = func(h common.Hash) {
// Coalesce logs and set 'Removed'.
Receipts := GetBlockReceipts(bc.chainDb, h, bc.hc.GetBlockNumber(h))
For _, receipt := range receipts {
For _, log := range receipt.Logs {
Del := *log
del.Removed = true
deletedLogs = append(deletedLogs, &amp;del)
}
}
}
)

// first reduce whoever is higher bound
If oldBlock.NumberU64() &gt; newBlock.NumberU64() {
// reduce old chain If the old chain is taller than the new one. Then you need to reduce the old chain and make it as high as the new chain.
For ; oldBlock != nil &amp;&amp; oldBlock.NumberU64() != newBlock.NumberU64(); oldBlock = bc.GetBlock(oldBlock.ParentHash(), oldBlock.NumberU64()-1) {
oldChain = append(oldChain, oldBlock)
deletedTxs = append(deletedTxs, oldBlock.Transactions()...)

collectLogs(oldBlock.Hash())
}
} else {
// reduce new chain and append new chain blocks for inserting later on
// If the new chain is higher than the old chain, then reduce the new chain.
For ; newBlock != nil &amp;&amp; newBlock.NumberU64() != oldBlock.NumberU64(); newBlock = bc.GetBlock(newBlock.ParentHash(), newBlock.NumberU64()-1) {
newChain = append(newChain, newBlock)
}
}
If oldBlock == nil {
Return fmt.Errorf("Invalid old chain")
}
If newBlock == nil {
Return fmt.Errorf("Invalid new chain")
}

For { // This for loop needs to find a common ancestor.
If oldBlock.Hash() == newBlock.Hash() {
commonBlock = oldBlock
Break
}

oldChain = append(oldChain, oldBlock)
newChain = append(newChain, newBlock)
deletedTxs = append(deletedTxs, oldBlock.Transactions()...)
collectLogs(oldBlock.Hash())

oldBlock, newBlock = bc.GetBlock(oldBlock.ParentHash(), oldBlock.NumberU64()-1), bc.GetBlock(newBlock.ParentHash(), newBlock.NumberU64()-1)
If oldBlock == nil {
Return fmt.Errorf("Invalid old chain")
}
If newBlock == nil {
Return fmt.Errorf("Invalid new chain")
}
}
// Ensure the user sees large reorgs
If len(oldChain) &gt; 0 &amp;&amp; len(newChain) &gt; 0 {
logFn := log.Debug
If len(oldChain) &gt; 63 {
logFn = log.Warn
}
logFn("Chain split detected", "number", commonBlock.Number(), "hash", commonBlock.Hash(),
"drop", len(oldChain), "dropfrom", oldChain[0].Hash(), "add", len(newChain), "addfrom", newChain[0].Hash())
} else {
log.Error("Impossible reorg, please file an issue", "oldnum", oldBlock.Number(), "oldhash", oldBlock.Hash(), "newnum", newBlock.Number(), "newhash", newBlock. Hash())
}
Var addedTxs types.Transactions
// insert blocks. Order does not matter. Last block will be written in ImportChain itself which creates the new head properly
For _, block := range newChain {
// insert the block in the canonical way, re-writing history
// Insert block Update the key of the record specification blockchain
Bc.insert(block)
// write lookup entries for hash based transaction/receipt searches
// Write the query information for the transaction.
If err := WriteTxLookupEntries(bc.chainDb, block); err != nil {
Return err
}
addedTxs = append(addedTxs, block.Transactions()...)
}

// calculate the difference between deleted and added transactions
Diff := types.TxDifference(deletedTxs, addedTxs)
// When transactions get deleted from the database that means the
// receipts that were created in the fork must also be deleted
// Delete the transaction query information that needs to be deleted.
// This does not delete the blocks, block headers, receipts, etc. that need to be deleted.
For _, tx := range diff {
DeleteTxLookupEntry(bc.chainDb, tx.Hash())
}
If len(deletedLogs) &gt; 0 { // send a message notification
Go bc.rmLogsFeed.Send(RemovedLogsEvent{deletedLogs})
}
If len(oldChain) &gt; 0 {
Go func() {
				for _, block := range oldChain { // 发送消息通知。
					bc.chainSideFeed.Send(ChainSideEvent{Block: block})
}
			}()
}

		return nil
}

</pre></body></html>