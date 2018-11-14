
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>## agent
The agent is the object of specific mining. The process it performs is to accept the calculated block header, calculate the mixhash and nonce, and return the mined block header.

The CpuAgent is constructed. Under normal circumstances, the CPU is not used for mining. Generally, mining is performed using a dedicated GPU for mining. The code for GPU mining will not be reflected here.

Type CpuAgent struct {
Mu sync.Mutex

workCh chan *Work // accept the channel for the mining task
Stop chan struct{}
quitCurrentOp chan struct{}
returnCh chan&lt;- *Result // return channel after mining completion

Chain consensus.ChainReader // Get blockchain information
Engine consensus.Engine // consistency engine, here refers to the Pow engine

isMining int32 // isMining indicates whether the agent is currently mining
}

Func NewCpuAgent(chain consensus.ChainReader, engine consensus.Engine) *CpuAgent {
Miner := &amp;CpuAgent{
Chain: chain,
Engine: engine,
Stop: make(chan struct{}, 1),
workCh: make(chan *Work, 1),
}
Return miner
}

Set the return value channel and get the Work channel to facilitate the external value and get the return information.

Func (self *CpuAgent) Work() chan&lt;- *Work { return self.workCh }
Func (self *CpuAgent) SetReturnCh(ch chan&lt;- *Result) { self.returnCh = ch }

Start and message loop, if mining has started, then exit directly, otherwise start update this goroutine
Update accepts tasks from workCh, mines, or accepts exit information and exits.

Func (self *CpuAgent) Start() {
If !atomic.CompareAndSwapInt32(&amp;self.isMining, 0, 1) {
Return // agent already started
}
Go self.update()
}

Func (self *CpuAgent) update() {
Out:
For {
Select {
Case work := &lt;-self.workCh:
self.mu.Lock()
If self.quitCurrentOp != nil {
Close(self.quitCurrentOp)
}
self.quitCurrentOp = make(chan struct{})
Go self.mine(work, self.quitCurrentOp)
self.mu.Unlock()
Case &lt;-self.stop:
self.mu.Lock()
If self.quitCurrentOp != nil {
Close(self.quitCurrentOp)
self.quitCurrentOp = nil
}
self.mu.Unlock()
Break out
}
}
}

Mine, mining, call the consistency engine for mining, if the mining is successful, send the message to returnCh.

Func (self *CpuAgent) mine(work *Work, stop &lt;-chan struct{}) {
If result, err := self.engine.Seal(self.chain, work.Block, stop); result != nil {
log.Info("Successfully sealed new block", "number", result.Number(), "hash", result.Hash())
Self.returnCh &lt;- &amp;Result{work, result}
} else {
If err != nil {
log.Warn("Block sealing failed", "err", err)
}
Self.returnCh &lt;- nil
}
}
GetHashRate, this function returns the current HashRate.

Func (self *CpuAgent) GetHashRate() int64 {
If pow, ok := self.engine.(consensus.PoW); ok {
Return int64(pow.Hashrate())
}
Return 0
}


## remote_agent
Remote_agent provides a set of RPC interfaces that enable remote miners to perform mining functions. For example, I have a mining machine. The inside of the mining machine does not run the Ethereum node. The mining machine first obtains the current task from the remote_agent, and then performs the mining calculation. When the mining is completed, the calculation result is submitted and the mining is completed.

Data structure and construction

Type RemoteAgent struct {
Mu sync.Mutex

quitCh chan struct{}
workCh chan *Work // accept the task
returnCh chan&lt;- *Result // result returns

Chain consensus.ChainReader
Engine consensus.Engine
currentWork *Work // current task
Work map[common.Hash]*Work // The task that has not yet been submitted is being calculated

hashrateMu sync.RWMutex
Hashrate map[common.Hash]hashrate // The hashrate of the task being calculated

Running int32 // running indicates whether the agent is active. Call atomically
}

Func NewRemoteAgent(chain consensus.ChainReader, engine consensus.Engine) *RemoteAgent {
Return &amp;RemoteAgent{
Chain: chain,
Engine: engine,
Work: make(map[common.Hash]*Work),
Hashrate: make(map[common.Hash]hashrate),
}
}

Start and stop

Func (a *RemoteAgent) Start() {
If !atomic.CompareAndSwapInt32(&amp;a.running, 0, 1) {
Return
}
a.quitCh = make(chan struct{})
a.workCh = make(chan *Work, 1)
Go a.loop(a.workCh, a.quitCh)
}

Func (a *RemoteAgent) Stop() {
If !atomic.CompareAndSwapInt32(&amp;a.running, 1, 0) {
Return
}
Close(a.quitCh)
Close(a.workCh)
}
Get the input and output channels, this is the same as agent.go.

Func (a *RemoteAgent) Work() chan&lt;- *Work {
Return a.workCh
}

Func (a *RemoteAgent) SetReturnCh(returnCh chan&lt;- *Result) {
a.returnCh = returnCh
}

The loop method is similar to the work done in agent.go. When the task is received, it is stored in the currentWork field. If you haven't completed a job in 84 seconds, then delete the job. If you haven't received the hashrate report for 10 seconds, delete the trace/.

// loop monitors mining events on the work and quit channels, updating the internal
// state of the rmeote miner until a termination is requested.
//
// Note, the reason the work and quit channels are passed as parameters is because
// RemoteAgent.Start() constantly recreates these channels, so the loop code cannot
// assume data stability in these member fields.
Func (a *RemoteAgent) loop(workCh chan *Work, quitCh chan struct{}) {
Ticker := time.NewTicker(5 * time.Second)
Defer ticker.Stop()

For {
Select {
Case &lt;-quitCh:
Return
Case work := &lt;-workCh:
a.mu.Lock()
a.currentWork = work
a.mu.Unlock()
Case &lt;-ticker.C:
// cleanup
a.mu.Lock()
For hash, work := range a.work {
If time.Since(work.createdAt) &gt; 7*(12*time.Second) {
Delete(a.work, hash)
}
}
a.mu.Unlock()

a.hashrateMu.Lock()
For id, hashrate := range a.hashrate {
If time.Since(hashrate.ping) &gt; 10*time.Second {
Delete(a.hashrate, id)
}
}
a.hashrateMu.Unlock()
}
}
}

GetWork, this method is called by a remote miner to get the current mining task.

Func (a *RemoteAgent) GetWork() ([3]string, error) {
a.mu.Lock()
Defer a.mu.Unlock()

Var res [3]string

If a.currentWork != nil {
Block := a.currentWork.Block

Res[0] = block.HashNoNonce().Hex()
seedHash := ethash.SeedHash(block.NumberU64())
Res[1] = common.BytesToHash(seedHash).Hex()
// Calculate the "target" to be returned to the external miner
n := big.NewInt(1)
n.Lsh(n, 255)
n.Div(n, block.Difficulty())
n.Lsh(n, 1)
Res[2] = common.BytesToHash(n.Bytes()).Hex()

A.work[block.HashNoNonce()] = a.currentWork
Return res, nil
}
Return res, errors.New("No work available yet, don't panic.")
}

SubmitWork, the remote miners call this method to submit the results of the mining. Submit the result to returnCh after verifying the result

// SubmitWork tries to inject a pow solution into the remote agent, returning
// whether the solution was accepted or not (not can be both a bad pow as well as
// any other error, like no work pending).
Func (a *RemoteAgent) SubmitWork(nonce types.BlockNonce, mixDigest, hash common.Hash) bool {
a.mu.Lock()
Defer a.mu.Unlock()

// Make sure the work submitted is present
Work := a.work[hash]
If work == nil {
log.Info("Work submitted but none pending", "hash", hash)
Return false
}
// Make sure the Engine solutions is indeed valid
Result := work.Block.Header()
result.Nonce = nonce
result.MixDigest = mixDigest

If err := a.engine.VerifySeal(a.chain, result); err != nil {
log.Warn("Invalid proof-of-work submitted", "hash", hash, "err", err)
Return false
}
Block := work.Block.WithSeal(result)

// Solutions seems to be valid, return to the miner and notify acceptance
a.returnCh &lt;- &amp;Result{work, block}
Delete(a.work, hash)

Return true
}

SubmitHashrate, submit hash power

Func (a *RemoteAgent) SubmitHashrate(id common.Hash, rate uint64) {
a.hashrateMu.Lock()
Defer a.hashrateMu.Unlock()

A.hashrate[id] = hashrate{time.Now(), rate}
}


## unconfirmed

Unconfirmed is a data structure used to track the user's local mining information, such as dug out a block, then wait for enough subsequent block confirmation (5), then check whether the local mining block is included in the specification. Inside the blockchain.

data structure

// headerRetriever is used by the unconfirmed block set to verify whether a previously
// mined block is part of the canonical chain or not.
// headerRetriever is used by unconfirmed block groups to verify that the previously mined block is part of the specification chain.
Type headerRetriever interface {
// GetHeaderByNumber retrieves the canonical header associated with a block number.
GetHeaderByNumber(number uint64) *types.Header
}

// unconfirmedBlock is a small collection of metadata about a locally mined block
// that is placed into a unconfirmed set for canonical chain inclusion tracking.
// unconfirmedBlock is a collection of small metadata for the local mining block, used to put unconfirmed collections to track whether the locally mined blocks are included in the canonical blockchain.
Type unconfirmedBlock struct {
Index uint64
Hash common.Hash
}

// unconfirmedBlocks implements a data structure to maintain locally mined blocks
// have have not yet adequate enough maturity to guarantee chain inclusion. It is
// used by the miner to provide logs to the user when a previously mined block
// has a modern enough guarantee to not be reorged out of te canonical chain.
// unconfirmedBlocks implements a data structure that manages locally mined blocks that have not yet reached sufficient trust to prove that they have been accepted by the canonical block. It is used to provide information to miners so that they know if the blocks they have previously dug into are included in the canonical blockchain.
Type unconfirmedBlocks struct {
Chain headerRetriever // Blockchain to verify canonical status through blockchain to be verified Use this interface to get the current specification block header information
Depth uint // Depth after which to discard previous blocks How many blocks have passed before discarding the previous block
Blocks *ring.Ring // Block infos to allow canonical chain cross checks // block information to allow specification chain cross checking
Lock sync.RWMutex // Protects the fields from concurrent access
}

// newUnconfirmedBlocks returns new data structure to track currently unconfirmed blocks.
Func newUnconfirmedBlocks(chain headerRetriever, depth uint) *unconfirmedBlocks {
Return &amp;unconfirmedBlocks{
Chain: chain,
Depth: depth,
}
}

Insert the tracking block, when the miner digs into a block, index is the height of the block, and hash is the hash value of the block.


// Insert adds a new block to the set of unconfirmed ones.
Func (set *unconfirmedBlocks) Insert(index uint64, hash common.Hash) {
// If a new block was mined locally, shift out any old enough blocks
// If a local block is dug, move out the block that has exceeded depth
set.Shift(index)

// Create the new item as its own ring
// The operation of the loop queue.
Item := ring.New(1)
item.Value = &amp;unconfirmedBlock{
Index: index,
Hash: hash,
}
// Set as the initial ring or append to the end
set.lock.Lock()
Defer set.lock.Unlock()

If set.blocks == nil {
Set.blocks = item
} else {
// Move to the last element of the loop queue to insert the item
set.blocks.Move(-1).Link(item)
}
// Display a log for the user to notify of a new mined block unconfirmed
log.Info("🔨 mined potential block", "number", index, "hash", hash)
}
The Shift method removes blocks whose index exceeds the incoming index-depth and checks if they are in the canonical blockchain.

// Shift drops all unconfirmed blocks from the set which exceed the unconfirmed sets depth
//allowing, checking them against against canonical chain for inclusion or staleness
// report.
Func (set *unconfirmedBlocks) Shift(height uint64) {
set.lock.Lock()
Defer set.lock.Unlock()

For set.blocks != nil {
// Retrieve the next unconfirmed block and abort if too fresh
// Because the blocks in blocks are in order. At the very beginning, it is definitely the oldest block.
// So only need to check the last block at a time, if it is finished, it will be removed from the loop queue.
Next := set.blocks.Value.(*unconfirmedBlock)
If next.index+uint64(set.depth) &gt; height { // If it is old enough.
Break
}
// Block seems to exceed depth allowance, check for canonical status
// Query the block header of that block height
Header := set.chain.GetHeaderByNumber(next.index)
Switch {
Case header == nil:
log.Warn("Failed to retrieve header of mined block", "number", next.index, "hash", next.hash)
Case header.Hash() == next.hash: // If the block header is equal to our own,
log.Info("🔗 block reached canonical chain", "number", next.index, "hash", next.hash)
Default: // Otherwise it means we are above the side chain.
log.Info("⑂ block became a side fork", "number", next.index, "hash", next.hash)
}
// Drop the block out of the ring
/ / Remove from the loop queue
If set.blocks.Value == set.blocks.Next().Value {
// If the current value is equal to our own, indicating that only the loop queue has only one element, then the setting is not nil
Set.blocks = nil
} else {
// Otherwise move to the end, then delete one and move to the front.
Set.blocks = set.blocks.Move(-1)
set.blocks.Unlink(1)
Set.blocks = set.blocks.Move(1)
}
}
}

## worker.go
The worker contains a lot of agents, including the agent and remote_agent mentioned earlier. The worker is also responsible for building blocks and objects. At the same time, the task is provided to the agent.

data structure:

Agent interface

// Agent can register themself with the worker
Type agent interface {
Work() chan&lt;- *Work
SetReturnCh(chan&lt;- *Result)
Stop()
Start()
GetHashRate() int64
}

The Work structure, which stores the worker's current environment and holds all temporary state information.

// Work is the workers current environment and holds
// all of the current state information
Type Work struct {
Config *params.ChainConfig
Signer types.Signer // signer

State *state.StateDB // apply state changes here state database
Ancestors *set.Set // ancestor set (used for checking uncle parent validity) An ancestor set to check if the ancestor is valid
Family *set.Set // family set (used for checking uncle invalidity) family collection to check the ancestry's ineffectiveness
Uncles *set.Set // uncle set uncles collection
Tcount int // tx count in cycle The number of transactions in this cycle

Block *types.Block // the new block // new block

Header *types.Header // block header
Txs []*types.Transaction // Transaction
Receipts []*types.Receipt // receipt

createdAt time.Time // create time
}

Type Result struct { // result
Work *Work
Block *types.Block
}
Worker

// worker is the main object which takes care of applying messages to the new state
// The worker is the primary object responsible for applying the message to the new state
Type worker struct {
Config *params.ChainConfig
Engine consensus.Engine
Mu sync.Mutex
// update loop
Mux *event.TypeMux
txCh chan core.TxPreEvent // channel used to accept transactions in txPool
txSub event.Subscription // The subscriber used to accept transactions in txPool
chainHeadCh chan core.ChainHeadEvent // channel used to accept the block header
chainHeadSub event.Subscription
chainSideCh chan core.ChainSideEvent // Used to accept a blockchain from the normal blockchain
chainSideSub event.Subscription
Wg sync.WaitGroup

Agent map[Agent]struct{} // all agents
Recv chan *Result // agent will send the result to this channel

Eth Backend // eth's agreement
Chain *core.BlockChain // blockchain
Proc core.Validator // blockchain validator
chainDb ethdb.Database // blockchain database

Coinbase common.Address // The address of the miner
Extra []byte //

snapshotMu sync.RWMutex // Snapshot RWMutex (snapshot read/write lock)
snapshotBlock *types.Block // snapshot block
snapshotState *state.StateDB // Snapshot StateDB

currentMu sync.Mutex
Current *Work

uncleMu sync.Mutex
possibleUncles map[common.Hash]*types.Block //Possible uncle node

Unconfirmed *unconfirmedBlocks // set of locally mined blocks pending canonicalness confirmations

// atomic status counters
Mining int32
atWork int32
}

structure

Func newWorker(config *params.ChainConfig, engine consensus.Engine, coinbase common.Address, eth Backend, mux *event.TypeMux) *worker {
Worker := &amp;worker{
Config: config,
Engine: engine,
Eth: eth,
Mux: mux,
txCh: make(chan core.TxPreEvent, txChanSize), // 4096
chainHeadCh: make(chan core.ChainHeadEvent, chainHeadChanSize), // 10
chainSideCh: make(chan core.ChainSideEvent, chainSideChanSize), // 10
chainDb: eth.ChainDb(),
Recv: make(chan *Result, resultQueueSize), // 10
Chain: eth.BlockChain(),
Proc: eth.BlockChain().Validator(),
possibleUncles: make(map[common.Hash]*types.Block),
Coinbase: coinbase,
Agents: make(map[Agent]struct{}),
Unconfirmed: newUnconfirmedBlocks(eth.BlockChain(), miningLogAtDepth),
}
// Subscribe TxPreEvent for tx pool
worker.txSub = eth.TxPool().SubscribeTxPreEvent(worker.txCh)
// Subscribe events for blockchain
worker.chainHeadSub = eth.BlockChain().SubscribeChainHeadEvent(worker.chainHeadCh)
worker.chainSideSub = eth.BlockChain().SubscribeChainSideEvent(worker.chainSideCh)
Go worker.update()

Go worker.wait()
worker.commitNewWork()

Return worker
}

Update

Func (self *worker) update() {
Defer self.txSub.Unsubscribe()
Defer self.chainHeadSub.Unsubscribe()
Defer self.chainSideSub.Unsubscribe()

For {
// A real event arrived, process interesting content
Select {
// Handle ChainHeadEvent Turns on the mining service as soon as it receives a block header.
Case &lt;-self.chainHeadCh:
self.commitNewWork()

// Handle ChainSideEvent receives chunks that are not in the canonical blockchain and joins the potential uncle's collection
Case ev := &lt;-self.chainSideCh:
self.uncleMu.Lock()
self.possibleUncles[ev.Block.Hash()] = ev.Block
self.uncleMu.Unlock()

// Handle TxPreEvent when receiving transaction information in txPool.
Case ev := &lt;-self.txCh:
// Apply transaction to the pending state if we're not mining
// If there is currently no mining, then apply the transaction to the current state so that the mining task can be started immediately.
If atomic.LoadInt32(&amp;self.mining) == 0 {
self.currentMu.Lock()
Acc, _ := types.Sender(self.current.signer, ev.Tx)
Txs := map[common.Address]types.Transactions{acc: {ev.Tx}}
Txset := types.NewTransactionsByPriceAndNonce(self.current.signer, txs)

self.current.commitTransactions(self.mux, txset, self.chain, self.coinbase)
self.currentMu.Unlock()
}

// System stopped
Case &lt;-self.txSub.Err():
Return
Case &lt;-self.chainHeadSub.Err():
Return
Case &lt;-self.chainSideSub.Err():
Return
}
}
}


commitNewWork submits a new task


Func (self *worker) commitNewWork() {
self.mu.Lock()
Defer self.mu.Unlock()
self.uncleMu.Lock()
Defer self.uncleMu.Unlock()
self.currentMu.Lock()
Defer self.currentMu.Unlock()

Tstart := time.Now()
Parent := self.chain.CurrentBlock()

Tstamp := tstart.Unix()
If parent.Time().Cmp(new(big.Int).SetInt64(tstamp)) &gt;= 0 { // Can't appear less than the time of the parent
Tstamp = parent.Time().Int64() + 1
}
// this will ensure we're not going off too far in the future
// Our time should not be too far away from the present time, then wait for a while,
// I feel that this function is completely for testing. If it is a real mining program, it should not wait.
If now := time.Now().Unix(); tstamp &gt; now+1 {
Wait := time.Duration(tstamp-now) * time.Second
log.Info("Mining too far in the future", "wait", common.PrettyDuration(wait))
time.Sleep(wait)
}

Num := parent.Number()
Header := &amp;types.Header{
ParentHash: parent.Hash(),
Number: num.Add(num, common.Big1),
GasLimit: core.CalcGasLimit(parent),
GasUsed: new(big.Int),
Extra: self.extra,
Time: big.NewInt(tstamp),
}
// Only set the coinbase if we are mining (avoid spurious block rewards)
// Set coinbase only when we are mining (avoid false block rewards? TODO doesn't understand)
If atomic.LoadInt32(&amp;self.mining) == 1 {
header.Coinbase = self.coinbase
}
If err := self.engine.Prepare(self.chain, header); err != nil {
log.Error("Failed to prepare header for mining", "err", err)
Return
}
// If we are care about TheDAO hard-fork check whether to override the extra-data or not
// Decide whether to overwrite additional data based on whether we care about DAO hard forks.
If daoBlock := self.config.DAOForkBlock; daoBlock != nil {
// Check whether the block is among the fork extra-override range
// Check if the block is within the range of DAO hard fork [daoblock,daoblock+limit]
Limit := new(big.Int).Add(daoBlock, params.DAOForkExtraRange)
If header.Number.Cmp(daoBlock) &gt;= 0 &amp;&amp; header.Number.Cmp(limit) &lt; 0 {
// Depending whether we support or oppose the fork, override differently
If self.config.DAOForkSupport { // If you support DAO then set the reserved extra data
header.Extra = common.CopyBytes(params.DAOForkBlockExtra)
} else if bytes.Equal(header.Extra, params.DAOForkBlockExtra) {
header.Extra = []byte{} // If miner opposes, don't let it use the reserved extra-data // Otherwise do not use the reserved extra data
}
}
}
// Could potentially happen if starting to mine in an odd state.
Err := self.makeCurrent(parent, header) // Set the current state with a new block header
If err != nil {
log.Error("Failed to create mining context", "err", err)
Return
}
// Create the current work task and check any fork transitions needed
Work := self.current
If self.config.DAOForkSupport &amp;&amp; self.config.DAOForkBlock != nil &amp;&amp; self.config.DAOForkBlock.Cmp(header.Number) == 0 {
misc.ApplyDAOHardFork(work.state) // Transfer funds from DAO to the specified account.
}
Pending, err := self.eth.TxPool().Pending() //Get blocked funds
If err != nil {
log.Error("Failed to fetch pending transactions", "err", err)
Return
}
// Create a transaction. Follow-up of this method
Txs := types.NewTransactionsByPriceAndNonce(self.current.signer, pending)
// Submit the transaction. This method is followed.
work.commitTransactions(self.mux, txs, self.chain, self.coinbase)

// compute uncles for the new block.
Var (
Uncles []*types.Header
badUncles []common.Hash
)
For hash, uncle := range self.possibleUncles {
If len(uncles) == 2 {
Break
}
If err := self.commitUncle(work, uncle.Header()); err != nil {
log.Trace("Bad uncle found and will be removed", "hash", hash)
log.Trace(fmt.Sprint(uncle))

badUncles = append(badUncles, hash)
} else {
log.Debug("Committing new uncle to block", "hash", hash)
Uncles = append(uncles, uncle.Header())
}
}
For _, hash := range badUncles {
Delete(self.possibleUncles, hash)
}
// Create the new block to seal with the consensus engine
// Create a new block with the given state, Finalize will perform block rewards, etc.
If work.Block, err = self.engine.Finalize(self.chain, header, work.state, work.txs, uncles, work.receipts); err != nil {
log.Error("Failed to finalize block for sealing", "err", err)
Return
}
// We only care about logging if we're actually mining.
//
If atomic.LoadInt32(&amp;self.mining) == 1 {
log.Info("Commit new mining work", "number", work.Block.Number(), "txs", work.tcount, "uncles", len(uncles), "elapsed", common.PrettyDuration(time. Since(tstart)))
self.unconfirmed.Shift(work.Block.NumberU64() - 1)
}
Self.push(work)
}

Push method, if we are not mining, then return directly, otherwise give the task to each agent

// push sends a new work task to currently live miner agents.
Func (self *worker) push(work *Work) {
If atomic.LoadInt32(&amp;self.mining) != 1 {
Return
}
For agent := range self.agents {
atomic.AddInt32(&amp;self.atWork, 1)
If ch := agent.Work(); ch != nil {
Ch &lt;- work
}
}
}

makeCurrent, creating a new environment without the current cycle.

// makeCurrent creates a new environment for the current cycle.
//
Func (self *worker) makeCurrent(parent *types.Block, header *types.Header) error {
State, err := self.chain.StateAt(parent.Root())
If err != nil {
Return err
}
Work := &amp;Work{
Config: self.config,
Signer: types.NewEIP155Signer(self.config.ChainId),
State: state,
Ancestors: set.New(),
Family: set.New(),
Uncles: set.New(),
Header: header,
createdAt: time.Now(),
}

// when 08 is processed ancestors contain 07 (quick block)
For _, ancestor := range self.chain.GetBlocksFromHash(parent.Hash(), 7) {
For _, uncle := range ancestor.Uncles() {
work.family.Add(uncle.Hash())
}
work.family.Add(ancestor.Hash())
work.ancestors.Add(ancestor.Hash())
}

// Keep track of transactions which return errors so they can be removed
Work.tcount = 0
Self.current = work
Return nil
}

commitTransactions

Func (env *Work) commitTransactions(mux *event.TypeMux, txs *types.TransactionsByPriceAndNonce, bc *core.BlockChain, coinbase common.Address) {
// Initialize the total gasPool to env.header.GasLimit because it is a new block in the package
If env.gasPool == nil {
env.gasPool = new(core.GasPool).AddGas(env.header.GasLimit)
}

Var coalescedLogs []*types.Log

For {
// If we don't have enough gas for any further transactions then we're done
// Exit the packaged transaction if all Gas consumption in the current block has been used up
If env.gasPool.Gas() &lt; params.TxGas {
log.Trace("Not enough gas for further transactions", "have", env.gasPool, "want", params.TxGas)
Break
}

// Retrieve the next transaction and abort if all done
// Retrieve the next transaction, exit the commit if the transaction set is empty
Tx := txs.Peek()
If tx == nil {
Break
}
// Error may be ignored here. The error has already been checked
// during transaction acceptance is the transaction pool.
//
// We use the eip155 signer regardless of the current hf.
From, _ := types.Sender(env.signer, tx)
// Check whether the tx is replay protected. If we're not in the EIP155 hf
// phase, start ignoring the sender until we do.
// Please refer to https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md
// After the DAO event, Ethereum splits into ETH and ETC, because the things on the two chains are the same, so at ETC
// The transaction that occurred above can be retrieved on ETH and vice versa. So Vitalik proposed EIP155 to avoid this situation.
If tx.Protected() &amp;&amp; !env.config.IsEIP155(env.header.Number) {
log.Trace("Ignoring reply protected transaction", "hash", tx.Hash(), "eip155", env.config.EIP155Block)

txs.Pop()
Continue
}
// Start executing the transaction
env.state.Prepare(tx.Hash(), common.Hash{}, env.tcount)
// execute the transaction
Err, logs := env.commitTransaction(tx, bc, coinbase, gp)
Switch err {
Case core.ErrGasLimitReached:
// Pop the current out-of-gas transaction without shifting in the next from the account
// Pops up all transactions for the entire account, not processing the user's next transaction.
log.Trace("Gas limit exceeded for current block", "sender", from)
txs.Pop()

Case core.ErrNonceTooLow:
// New head notification data race between the transaction pool and miner, shift
// move to the user's next transaction
log.Trace("Skipping transaction with low nonce", "sender", from, "nonce", tx.Nonce())
txs.Shift()

Case core.ErrNonceTooHigh:
// Reorg notification data race between the transaction pool and miner, skip account =
// Skip this account
log.Trace("Skipping account with hight nonce", "sender", from, "nonce", tx.Nonce())
txs.Pop()

Case nil:
// Everything ok, collect the logs and shift in the next transaction from the same account
coalescedLogs = append(coalescedLogs, logs...)
Env.tcount++
txs.Shift()

Default:
// Strange error, discard the transaction and get the next in line (note, the
// nonce-too-high clause will prevent us from executing in vain).
// Other weird mistakes, skip this transaction.
log.Debug("Transaction failed, account skipped", "hash", tx.Hash(), "err", err)
txs.Shift()
}
}

If len(coalescedLogs) &gt; 0 || env.tcount &gt; 0 {
// make a copy, the state caches the logs and these logs get "upgraded" from pending to mined
// logs by filling in the block hash when the block was mined by the local miner. This can
// cause a race condition if a log was "upgraded" before the PendingLogsEvent is processed.
// Because the log needs to be sent out, and the log needs to be modified after the mining is completed, so copy one copy and avoid contention.
Cpy := make([]*types.Log, len(coalescedLogs))
For i, l := range coalescedLogs {
Cpy[i] = new(types.Log)
*cpy[i] = *l
}
Go func(logs []*types.Log, tcount int) {
If len(logs) &gt; 0 {
mux.Post(core.PendingLogsEvent{Logs: logs})
}
If tcount &gt; 0 {
mux.Post(core.PendingStateEvent{})
}
}(cpy, env.tcount)
}
}

commitTransaction executes ApplyTransaction

Func (env *Work) commitTransaction(tx *types.Transaction, bc *core.BlockChain, coinbase common.Address, gp *core.GasPool) (error, []*types.Log) {
Snap := env.state.Snapshot()

Receipt, _, err := core.ApplyTransaction(env.config, bc, &amp;coinbase, gp, env.state, env.header, tx, env.header.GasUsed, vm.Config{})
If err != nil {
env.state.RevertToSnapshot(snap)
Return err, nil
}
Env.txs = append(env.txs, tx)
Env.receipts = append(env.receipts, receipt)

Return nil, receipt.Logs
}

The wait function is used to accept the results of the mining and then write to the local blockchain, and broadcast it through the eth protocol.

Func (self *worker) wait() {
For {
mustCommitNewWork := true
For result := range self.recv {
atomic.AddInt32(&amp;self.atWork, -1)

If result == nil {
Continue
}
Block := result.Block
Work := result.Work

// Update the block hash in all logs since it is now available and not when the
// receipt/log of individual transactions were created.
For _, r := range work.receipts {
For _, l := range r.Logs {
l.BlockHash = block.Hash()
}
}
For _, log := range work.state.Logs() {
log.BlockHash = block.Hash()
}
Stat, err := self.chain.WriteBlockAndState(block, work.receipts, work.state)
If err != nil {
log.Error("Failed writing block to chain", "err", err)
Continue
}
// check if canon block and write transactions
If stat == core.CanonStatTy { // Description Blockchain that has been inserted into the specification
// implicit by posting ChainHeadEvent
// Because in this state, ChainHeadEvent will be sent, which will trigger the code inside the update. This part of the code will commitNewWork, so there is no need to commit here.
mustCommitNewWork = false
}
// Broadcast the block and announce chain insertion event
// Broadcast the block and declare the blockchain insertion event.
self.mux.Post(core.NewMinedBlockEvent{Block: block})
Var (
Events []interface{}
Logs = work.state.Logs()
)
Events = append(events, core.ChainEvent{Block: block, Hash: block.Hash(), Logs: logs})
If stat == core.CanonStatTy {
Events = append(events, core.ChainHeadEvent{Block: block})
}
self.chain.PostChainEvents(events, logs)

// Insert the block into the set of pending ones to wait for confirmations
// Insert the local tracking list to see the subsequent confirmation status.
self.unconfirmed.Insert(block.NumberU64(), block.Hash())

If mustCommitNewWork { // TODO ?
self.commitNewWork()
}
}
}
}


## miner
Miner is used to manage workers, subscribe to external events, and control the start and stop of workers.

data structure


// Backend wraps all methods required for mining.
Type Backend interface {
AccountManager() *accounts.Manager
BlockChain() *core.BlockChain
TxPool() *core.TxPool
ChainDb() ethdb.Database
}

// Miner creates blocks and searches for proof-of-work values.
Type Miner struct {
Mux *event.TypeMux

Worker *worker

Coinbase common.Address
Mining int32
Eth Backend
Engine consensus.Engine

canStart int32 // can start indicating whether we can start the mining operation
		shouldStart int32 // should start indicates whether we should start after sync
}


构造, 创建了一个CPU agent 启动了miner的update goroutine


	func New(eth Backend, config *params.ChainConfig, mux *event.TypeMux, engine consensus.Engine) *Miner {
		miner := &amp;Miner{
			eth:      eth,
			mux:      mux,
			engine:   engine,
			worker:   newWorker(config, engine, common.Address{}, eth, mux),
			canStart: 1,
}
		miner.Register(NewCpuAgent(eth.BlockChain(), engine))
		go miner.update()

		return miner
}

update订阅了downloader的事件， 注意这个goroutine是一个一次性的循环， 只要接收到一次downloader的downloader.DoneEvent或者 downloader.FailedEvent事件， 就会设置canStart为1. 并退出循环， 这是为了避免黑客恶意的 DOS攻击，让你不断的处于异常状态

	// update keeps track of the downloader events. Please be aware that this is a one shot type of update loop.
	// It's entered once and as soon as `Done` or `Failed` has been broadcasted the events are unregistered and
	// the loop is exited. This to prevent a major security vuln where external parties can DOS you with blocks
	// and halt your mining operation for as long as the DOS continues.
	func (self *Miner) update() {
		events := self.mux.Subscribe(downloader.StartEvent{}, downloader.DoneEvent{}, downloader.FailedEvent{})
	out:
		for ev := range events.Chan() {
			switch ev.Data.(type) {
			case downloader.StartEvent:
				atomic.StoreInt32(&amp;self.canStart, 0)
				if self.Mining() {
					self.Stop()
					atomic.StoreInt32(&amp;self.shouldStart, 1)
					log.Info("Mining aborted due to sync")
}
			case downloader.DoneEvent, downloader.FailedEvent:
				shouldStart := atomic.LoadInt32(&amp;self.shouldStart) == 1

				atomic.StoreInt32(&amp;self.canStart, 1)
				atomic.StoreInt32(&amp;self.shouldStart, 0)
				if shouldStart {
					self.Start(self.coinbase)
}
				// unsubscribe. we're only interested in this event once
				events.Unsubscribe()
				// stop immediately and ignore all further pending events
				break out
}
}
}

Start

	func (self *Miner) Start(coinbase common.Address) {
		atomic.StoreInt32(&amp;self.shouldStart, 1)  // shouldStart 是是否应该启动
		self.worker.setEtherbase(coinbase)	     
		self.coinbase = coinbase

		if atomic.LoadInt32(&amp;self.canStart) == 0 {  // canStart是否能够启动，
			log.Info("Network syncing, will start miner afterwards")
Return
}
		atomic.StoreInt32(&amp;self.mining, 1)

		log.Info("Starting mining operation")
		self.worker.start()  // 启动worker 开始挖矿
		self.worker.commitNewWork()  //提交新的挖矿任务。
}
</pre></body></html>