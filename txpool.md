
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Txpool is mainly used to store the currently committed transactions waiting to be written to the block, both remote and local.

There are two types of transactions in txpool.
1. Submitted but not yet executable, placed in the queue waiting to be executed (for example, nonce is too high).
2. Wait for execution, put it in pending and wait for execution.

From the test case of txpool, the main functions of txpool are as follows.

1. The function of transaction verification, including insufficient balance, insufficient Gas, Nonce is too low, value value is legal, can not be negative.
2. Ability to cache transactions where Nonce is in a higher state than the current local account. Stored in the queue field. If it is a transaction that can be executed, it is stored in the pending field.
3. The same Nonce transaction for the same user will only retain the one with the largest GasPrice. Other inserts were unsuccessful.
4. If the account has no money, the transaction of the corresponding account in queue and pending will be deleted.
5. If the balance of the account is less than the amount of some transactions, the corresponding transaction will be deleted, and the valid transaction will be moved from pending to queue. Prevent being broadcast.
6. txPool supports some restrictions on PriceLimit (remove the lowest GasPrice limit), PriceBump (the percentage of the price of the transaction replacing the same Nonce) AccountSlots (the minimum value of the pending slot for each account) GlobalSlots (the maximum value of the global pending queue) AccountQueue (the minimum value of the queue for each account) GlobalQueue (the maximum value of the global queueing) Lifetime (the longest waiting time in the queue)
7. Replace the priority of GasPrice with limited resources.
8. The local transaction will be stored on the disk using the journal function and will be re-imported after reboot. Remote trading will not.




data structure

// TxPool contains all currently known transactions.
// enter the pool when they are received from the network or submitted
//local. They exit the pool when they are included in the blockchain.
// TxPool contains the currently known transaction, the current network receives the transaction, or the locally submitted transaction is added to the TxPool.
// Removed when they have been added to the blockchain.
// The pool separates processable transactions (which can be applied to the
// current state) and future transactions.
// two states over time as they are received and processed.
// TxPool is divided into executable transactions (which can be applied to the current state) and future transactions. The transaction switches between these two states,
Type TxPool struct {
Config TxPoolConfig
Chainconfig *params.ChainConfig
Chain blockChain
gasPrice *big.Int //The lowest GasPrice limit
txFeed event.Feed //Subscribe to TxPool messages via txFeed
Scope event.SubscriptionScope
chainHeadCh chan ChainHeadEvent // Subscribe to the block header message and receive a notification here when a new block header is generated.
chainHeadSub event.Subscription // The subscriber to the block header message.
Signer types.Signer // encapsulates transaction signature processing.
Mu sync.RWMutex

currentState *state.StateDB // Current state in the blockchain head
pendingState *state.ManagedState // Pending state tracking virtual nonces
currentMaxGas *big.Int // Current gas limit for transaction caps GasLimit

Locals *accountSet // Set of local transaction to exepmt from evicion rules
Journal *txJournal // Journal of local transaction to back up to disk

Pending map[common.Address]*txList // All currently processable transactions All currently available transactions
Queue map[common.Address]*txList // Queued but non-processable transactions Transactions that are currently not processed
Beats map[common.Address]time.Time // Last heartbeat from each known account The time of the last heartbeat information for each known account
All map[common.Hash]*types.Transaction // All transactions to allow lookups can find all transactions
Priced *txPricedList // All transactions sorted by price

Wg sync.WaitGroup // for shutdown sync

Homestead bool // home version
}



Construct


// NewTxPool creates a new transaction pool to gather, sort and filter inbound
// trnsactions from the network.
Func NewTxPool(config TxPoolConfig, chainconfig *params.ChainConfig, chain blockChain) *TxPool {
// Sanitize the input to ensure no vulnerable gas prices are set
Config = (&amp;config).sanitize()

// Create the transaction pool with its initial settings
Pool := &amp;TxPool{
Config: config,
Chainconfig: chainconfig,
Chain: chain,
Signer: types.NewEIP155Signer(chainconfig.ChainId),
Pending: make(map[common.Address]*txList),
Queue: make(map[common.Address]*txList),
Beats: make(map[common.Address]time.Time),
All: make(map[common.Hash]*types.Transaction),
chainHeadCh: make(chan ChainHeadEvent, chainHeadChanSize),
gasPrice: new(big.Int).SetUint64(config.PriceLimit),
}
Pool.locals = newAccountSet(pool.signer)
Pool.priced = newTxPricedList(&amp;pool.all)
Pool.reset(nil, chain.CurrentBlock().Header())

// If local transactions and journaling is enabled, load from disk
// If the local transaction is allowed and the configured Journal directory is not empty, load the log from the specified directory.
// Then rotate the transaction log. Since the old transaction may have expired, the received transaction is written to the log after the add method is called.
//
If !config.NoLocals &amp;&amp; config.Journal != "" {
Pool.journal = newTxJournal(config.Journal)

If err := pool.journal.load(pool.AddLocal); err != nil {
log.Warn("Failed to load transaction journal", "err", err)
}
If err := pool.journal.rotate(pool.local()); err != nil {
log.Warn("Failed to rotate transaction journal", "err", err)
}
}
// Subscribe events from blockchain Subscribe to events from the blockchain.
pool.chainHeadSub = pool.chain.SubscribeChainHeadEvent(pool.chainHeadCh)

// Start the event loop and return
pool.wg.Add(1)
Go pool.loop()

Return pool
}

The reset method retrieves the current state of the blockchain and ensures that the contents of the transaction pool are valid with respect to the current blockchain state. The main features include:

1. Because the block header is replaced, some transactions in the original block are invalidated due to the replacement of the block header. This part of the transaction needs to be re-added to the txPool to wait for the new block to be inserted.
2. Generate new currentState and pendingState
3. Because of the change in state. Move some of the transactions in pending to the queue
4. Because the state changes, move the transaction in the queue into the pending.

Reset code

// reset retrieves the current state of the blockchain and guarantee the content
// of the transaction pool is valid with regard to the chain state.
Func (pool *TxPool) reset(oldHead, newHead *types.Header) {
// If we're reorging an old state, reinject all dropped transactions
Var reinject types.Transactions

If oldHead != nil &amp;&amp; oldHead.Hash() != newHead.ParentHash {
// If the reorg is too deep, avoid doing it (will happen during fast sync)
oldNum := oldHead.Number.Uint64()
newNum := newHead.Number.Uint64()

If depth := uint64(math.Abs(float64(oldNum) - float64(newNum))); depth &gt; 64 { //If the old head and the new head are too far apart, then cancel the rebuild
log.Warn("Skipping deep transaction reorg", "depth", depth)
} else {
// Reorg seems under enough to pull in all transactions into memory
Var discarded, included types.Transactions

Var (
Rem = pool.chain.GetBlock(oldHead.Hash(), oldHead.Number.Uint64())
Add = pool.chain.GetBlock(newHead.Hash(), newHead.Number.Uint64())
)
// If the old height is greater than the new one, then you need to delete all of it.
For rem.NumberU64() &gt; add.NumberU64() {
Discarded = append(discarded, rem.Transactions()...)
If rem = pool.chain.GetBlock(rem.ParentHash(), rem.NumberU64()-1); rem == nil {
log.Error("Unrooted old chain seen by tx pool", "block", oldHead.Number, "hash", oldHead.Hash())
Return
}
}
// If the new height is greater than the old one, then it needs to be increased.
For add.NumberU64() &gt; rem.NumberU64() {
Included = append(included, add.Transactions()...)
If add = pool.chain.GetBlock(add.ParentHash(), add.NumberU64()-1); add == nil {
log.Error("Unrooted new chain seen by tx pool", "block", newHead.Number, "hash", newHead.Hash())
Return
}
}
// The height is the same. If the hashes are different, then you need to find them backwards and find the nodes of their same hash roots.
For rem.Hash() != add.Hash() {
Discarded = append(discarded, rem.Transactions()...)
If rem = pool.chain.GetBlock(rem.ParentHash(), rem.NumberU64()-1); rem == nil {
log.Error("Unrooted old chain seen by tx pool", "block", oldHead.Number, "hash", oldHead.Hash())
Return
}
Included = append(included, add.Transactions()...)
If add = pool.chain.GetBlock(add.ParentHash(), add.NumberU64()-1); add == nil {
log.Error("Unrooted new chain seen by tx pool", "block", newHead.Number, "hash", newHead.Hash())
Return
}
}
// Find out all the values ​​that exist in the discard, but not in the included.
// Need to wait for these transactions to be reinserted into the pool.
Reinject = types.TxDifference(discarded, included)
}
}
// Initialize the internal state to the current head
If newHead == nil {
newHead = pool.chain.CurrentBlock().Header() // Special case during testing
}
Statedb, err := pool.chain.StateAt(newHead.Root)
If err != nil {
log.Error("Failed to reset txpool state", "err", err)
Return
}
pool.currentState = statedb
pool.pendingState = state.ManageState(statedb)
pool.currentMaxGas = newHead.GasLimit

// Inject any transactions discarded due to reorgs
log.Debug("Reinjecting stale transactions", "count", len(reinject))
pool.addTxsLocked(reinject, false)

// validate the pool of pending transactions, this will remove
// any transactions that have been included in the block or
// have been invalidated because of another transaction (e.g.
// higher gas price)
// Verifying transactions in the pending transaction pool will remove all transactions in the existing blockchain, or because other transactions result in unavailable transactions (such as having a higher gasPrice)
// demote downgrades some of the transactions in pending to be downgraded to the queue.
pool.demoteUnexecutables()

// Update all accounts to the latest known pending nonce
// Update the nonnce of all accounts according to the nonce of the pending queue
For addr, list := range pool.pending {
Txs := list.Flatten() // Heavy but will be cached and is needed by the miner anyway
pool.pendingState.SetNonce(addr, txs[len(txs)-1].Nonce()+1)
}
// Check the queue and move transactions over to the pending if possible
// or remove those that have become invalid
// Check the queue and move the transaction to pending as much as possible, or delete those that have expired
// promote upgrade
pool.promoteExecutables(nil)
}
addTx

// addTx enqueues a single transaction into the pool if it is valid.
Func (pool *TxPool) addTx(tx *types.Transaction, local bool) error {
pool.mu.Lock()
Defer pool.mu.Unlock()

// Try to inject the transaction and update any state
Replace, err := pool.add(tx, local)
If err != nil {
Return err
}
// If we added a new transaction, run promotion checks and return
If !replace {
From, _ := types.Sender(pool.signer, tx) // already validated
pool.promoteExecutables([]common.Address{from})
}
Return nil
}

addTxsLocked

// addTxsLocked attempts to queue a batch of transactions if they are valid,
// assuming the transaction pool lock is already held.
// addTxsLocked attempts to put a valid transaction into the queue. When calling this function, it is assumed that the lock has been acquired.
Func (pool *TxPool) addTxsLocked(txs []*types.Transaction, local bool) error {
// Add the batch of transaction, tracking the accepted ones
Dirty := make(map[common.Address]struct{})
For _, tx := range txs {
If replace, err := pool.add(tx, local); err == nil {
If !replace { // replace is the meaning of replacement. If it is not replaced, then the status is updated and there is a possibility that it can be processed next.
From, _ := types.Sender(pool.signer, tx) // already validated
Dirty[from] = struct{}{}
}
}
}
// Only reprocess the internal state if something was actually added
If len(dirty) &gt; 0 {
Addrs := make([]common.Address, 0, len(dirty))
For addr, _ := range dirty {
Addrs = append(addrs, addr)
}
// passed in the modified address,
pool.promoteExecutables(addrs)
}
Return nil
}
demoteUnexecutables deletes invalid or processed transactions from pending, and other unexecutable transactions are moved to the future queue.

// demoteUnexecutables removes invalid and processed transactions from the pools
// executable/pending queue and any subsequent transactions that become unexecutable
// are moved back into the future queue.
Func (pool *TxPool) demoteUnexecutables() {
// Iterate over all accounts and demote any non-executable transactions
For addr, list := range pool.pending {
Nonce := pool.currentState.GetNonce(addr)

// Drop all transactions that are distinguished too old (low nonce)
// Delete all nonce transactions that are smaller than the current address and delete them from pool.all.
For _, tx := range list.Forward(nonce) {
Hash := tx.Hash()
log.Trace("Removed old pending transaction", "hash", hash)
Delete(pool.all, hash)
pool.priced.Removed()
}
// Drop all transactions that are too costly (low balance or out of gas), and queue any invalids back for later
// Remove all too expensive transactions. The user's balance may not be enough. Or out of gas
Drops, invalids := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
For _, tx := range drops {
Hash := tx.Hash()
log.Trace("Removed unpayable pending transaction", "hash", hash)
Delete(pool.all, hash)
pool.priced.Removed()
pendingNofundsCounter.Inc(1)
}
For _, tx := range invalids {
Hash := tx.Hash()
log.Trace("Demoting pending transaction", "hash", hash)
pool.enqueueTx(hash, tx)
}
// If there's a gap in front, warn (should never happen) and postpone all transactions
// If there is a hole (nonce hole), then all transactions need to be placed in the future queue.
// This step should indeed not happen because Filter has already processed invalids. There should be no transactions for invalids, that is, there are no holes.
If list.Len() &gt; 0 &amp;&amp; list.txs.Get(nonce) == nil {
For _, tx := range list.Cap(0) {
Hash := tx.Hash()
log.Error("Demoting invalidated transaction", "hash", hash)
pool.enqueueTx(hash, tx)
}
}
// Delete the entire queue entry if it became empty.
If list.Empty() {
Delete(pool.pending, addr)
Delete(pool.beats, addr)
}
}
}

enqueueTx inserts a new transaction into the future queue. This method assumes that the pool's lock has been acquired.

// enqueueTx inserts a new transaction into the non-executable transaction queue.
//
// Note, this method assumes the pool lock is held!
Func (pool *TxPool) enqueueTx(hash common.Hash, tx *types.Transaction) (bool, error) {
// Try to insert the transaction into the future queue
From, _ := types.Sender(pool.signer, tx) // already validated
If pool.queue[from] == nil {
Pool.queue[from] = newTxList(false)
}
Insert, old := pool.queue[from].Add(tx, pool.config.PriceBump)
If !inserted {
// An older transaction was better, discard this
queuedDiscardCounter.Inc(1)
Return false, ErrReplaceUnderpriced
}
// Discard any previous transaction and mark this
If old != nil {
Delete(pool.all, old.Hash())
pool.priced.Removed()
queuedReplaceCounter.Inc(1)
}
Pool.all[hash] = tx
pool.priced.Put(tx)
Return old != nil, nil
}

The promoteExecutables method inserts transactions that have become executable from the future queue into the pending queue. Through this process, all invalid transactions (nonce too low, insufficient balance) will be deleted.

// promoteExecutables moves transactions that have become processable from the
// future queue to the set of pending transactions. During this process, all
// invalidated transactions (low nonce, low balance) are deleted.
Func (pool *TxPool) promoteExecutables(accounts []common.Address) {
// Gather all the accounts potentially needing updates
// accounts store all the accounts that need to be updated. If the account is passed in as nil, it represents all known accounts.
If accounts == nil {
Accounts = make([]common.Address, 0, len(pool.queue))
For addr, _ := range pool.queue {
Accounts = append(accounts, addr)
}
}
// Iterate over all accounts and promote any executable transactions
For _, addr := range accounts {
List := pool.queue[addr]
If list == nil {
Continue // Just in case someone calls with a non existing account
}
// Drop all transactions that are distinguished too old (low nonce)
// delete all nonce too low transactions
For _, tx := range list.Forward(pool.currentState.GetNonce(addr)) {
Hash := tx.Hash()
log.Trace("Removed old queued transaction", "hash", hash)
Delete(pool.all, hash)
pool.priced.Removed()
}
// Drop all transactions that are too costly (low balance or out of gas)
// Delete all transactions with insufficient balance.
Drops, _ := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
For _, tx := range drops {
Hash := tx.Hash()
log.Trace("Removed unpayable queued transaction", "hash", hash)
Delete(pool.all, hash)
pool.priced.Removed()
queuedNofundsCounter.Inc(1)
}
// Gather all executable transactions and promote them
// Get all the trades that can be executed, and promoteTx to join the pending
For _, tx := range list.Ready(pool.pendingState.GetNonce(addr)) {
Hash := tx.Hash()
log.Trace("Promoting queued transaction", "hash", hash)
pool.promoteTx(addr, hash, tx)
}
// Drop all transactions over the allowed limit
// Delete all transactions that exceed the limit.
If !pool.locals.contains(addr) {
For _, tx := range list.Cap(int(pool.config.AccountQueue)) {
Hash := tx.Hash()
Delete(pool.all, hash)
pool.priced.Removed()
queuedRateLimitCounter.Inc(1)
log.Trace("Removed cap-exceeding queued transaction", "hash", hash)
}
}
// Delete the entire queue entry if it became empty.
If list.Empty() {
Delete(pool.queue, addr)
}
}
// If the pending limit is overflown, start equalizing allowances
Pending := uint64(0)
For _, list := range pool.pending {
Pending += uint64(list.Len())
}
// If the total number of pending exceeds the configuration of the system.
If pending &gt; pool.config.GlobalSlots {

pendingBeforeCap := pending
// Assemble a spam order to penalize large transactors first
Spammers := prque.New()
For addr, list := range pool.pending {
// Only evict transactions from high rollers
// First record all accounts larger than the minimum value of AccountSlots, and some transactions will be removed from these accounts.
// Note that spammers are a priority queue, which is sorted by the number of transactions from large to small.
If !pool.locals.contains(addr) &amp;&amp; uint64(list.Len()) &gt; pool.config.AccountSlots {
spammers.Push(addr, float32(list.Len()))
}
}
// Gradually drop transactions from offenders
Offenders := []common.Address{}
For pending &gt; pool.config.GlobalSlots &amp;&amp; !spammers.Empty() {
/*
Simulate changes in the number of account transactions in the offenders queue.
First cycle [10] end of cycle [10]
Second cycle [10, 9] End of cycle [9,9]
Third cycle [9, 9, 7] End of cycle [7, 7, 7]
Fourth cycle [7, 7 , 7 , 2] End of cycle [2, 2 , 2, 2]
*/
// Retrieve the next offender if not local address
Offender, _ := spammers.Pop()
Offenders = append(offenders, offender.(common.Address))

// Equalize balances until all the same or below threshold
If len(offenders) &gt; 1 { // The first time you enter this loop, there are two accounts with the largest number of transactions in the offenders queue.
// Calculate the equalization threshold for all current offenders
// The transaction amount of the last added account is the threshold of the cost
Threshold := pool.pending[offender.(common.Address)].Len()

// Iteratively reduce all offenders until below limit or threshold reached
// traverse until pending is valid, or the second to last transaction is equal to the last transaction
For pending &gt; pool.config.GlobalSlots &amp;&amp; pool.pending[offenders[len(offenders)-2]].Len() &gt; threshold {
// Traverse all accounts except the last one, and subtract the number of their transactions by 1.
For i := 0; i &lt; len(offenders)-1; i++ {
List := pool.pending[offenders[i]]
For _, tx := range list.Cap(list.Len() - 1) {
// Drop the transaction from the global pools too
Hash := tx.Hash()
Delete(pool.all, hash)
pool.priced.Removed()

// Update the account nonce to the dropped transaction
If nonce := tx.Nonce(); pool.pendingState.GetNonce(offenders[i]) &gt; nonce {
pool.pendingState.SetNonce(offenders[i], nonce)
}
log.Trace("Removed fairness-exceeding pending transaction", "hash", hash)
}
Pending--
}
}
}
}
// If still above threshold, reduce to limit or min allowance
// After the above loop, the number of transactions for all accounts that exceed AccountSlots has changed to the previous minimum.
// If the threshold is still exceeded, then continue to delete one from the offenders each time.
If pending &gt; pool.config.GlobalSlots &amp;&amp; len(offenders) &gt; 0 {
For pending &gt; pool.config.GlobalSlots &amp;&amp; uint64(pool.pending[offenders[len(offenders)-1]].Len()) &gt; pool.config.AccountSlots {
For _, addr := range offenders {
List := pool.pending[addr]
For _, tx := range list.Cap(list.Len() - 1) {
// Drop the transaction from the global pools too
Hash := tx.Hash()
Delete(pool.all, hash)
pool.priced.Removed()

// Update the account nonce to the dropped transaction
If nonce := tx.Nonce(); pool.pendingState.GetNonce(addr) &gt; nonce {
pool.pendingState.SetNonce(addr, nonce)
}
log.Trace("Removed fairness-exceeding pending transaction", "hash", hash)
}
Pending--
}
}
}
pendingRateLimitCounter.Inc(int64(pendingBeforeCap - pending))
} //end if pending &gt; pool.config.GlobalSlots {
// If we've queued more transactions than the hard limit, drop oldest ones
// We have dealt with the restriction of pending, and we need to deal with the limitations of the future queue.
Queued := uint64(0)
For _, list := range pool.queue {
Queued += uint64(list.Len())
}
If queued &gt; pool.config.GlobalQueue {
// Sort all accounts with queued transactions by heartbeat
Addresses := make(addresssByHeartbeat, 0, len(pool.queue))
For addr := range pool.queue {
If !pool.locals.contains(addr) { // don't drop locals
Addresses = append(addresses, addressByHeartbeat{addr, pool.beats[addr]})
}
}
sort.Sort(addresses)

// Drop transactions until the total is below the limit or only locals remain
// From now on, the more the heartbeat is new, the more it will be deleted.
For drop := queued - pool.config.GlobalQueue; drop &gt; 0 &amp;&amp; len(addresses) &gt; 0; {
Addr := addresses[len(addresses)-1]
List := pool.queue[addr.address]

Addresses = addresses[:len(addresses)-1]

// Drop all transactions if they are less than the overflow
If size := uint64(list.Len()); size &lt;= drop {
For _, tx := range list.Flatten() {
pool.removeTx(tx.Hash())
}
Drop -= size
queuedRateLimitCounter.Inc(int64(size))
Continue
}
// Otherwise drop only last few transactions
Txs := list.Flatten()
For i := len(txs) - 1; i &gt;= 0 &amp;&amp; drop &gt; 0; i-- {
pool.removeTx(txs[i].Hash())
Drop--
queuedRateLimitCounter.Inc(1)
}
}
}
}

promoteTx adds a transaction to the pending queue. This method assumes that a lock has been obtained.

// promoteTx adds a transaction to the pending (processable) list of transactions.
//
// Note, this method assumes the pool lock is held!
Func (pool *TxPool) promoteTx(addr common.Address, hash common.Hash, tx *types.Transaction) {
// Try to insert the transaction into the pending queue
If pool.pending[addr] == nil {
Pool.pending[addr] = newTxList(true)
}
List := pool.pending[addr]

Insert, old := list.Add(tx, pool.config.PriceBump)
If !inserted { // If it can't be replaced, there is already an old transaction. Delete.
// An older transaction was better, discard this
Delete(pool.all, hash)
pool.priced.Removed()

pendingDiscardCounter.Inc(1)
Return
}
// Otherwise discard any previous transaction and mark this
If old != nil {
Delete(pool.all, old.Hash())
pool.priced.Removed()

pendingReplaceCounter.Inc(1)
}
// Failsafe to work around direct pending inserts (tests)
If pool.all[hash] == nil {
Pool.all[hash] = tx
pool.priced.Put(tx)
}
// Set the potentially new pending nonce and notify any subsystems of the new tx
// Add the transaction to the queue and send a message to tell all subscribers that the subscriber is inside the eth protocol. It will receive the message and broadcast the message over the network.
Pool.beats[addr] = time.Now()
pool.pendingState.SetNonce(addr, tx.Nonce()+1)
Go pool.txFeed.Send(TxPreEvent{tx})
}


removeTx, delete a transaction, and move all subsequent transactions to the future queue


// removeTx removes a single transaction from the queue, moving all subsequent
// transactions back to the future queue.
Func (pool *TxPool) removeTx(hash common.Hash) {
// Fetch the transaction we wish to delete
Tx, ok := pool.all[hash]
If !ok {
Return
}
Addr, _ := types.Sender(pool.signer, tx) // already validated during insertion

// Remove it from the list of known transactions
Delete(pool.all, hash)
pool.priced.Removed()

// Remove the transaction from the pending lists and reset the account nonce
// Remove the transaction from pending and put the transaction that was invalidated due to the deletion of this transaction into the future queue
// then update the state of pendingState
If pending := pool.pending[addr]; pending != nil {
If removed, invalids := pending.Remove(tx); removed {
// If no more transactions are left, remove the list
If pending.Empty() {
Delete(pool.pending, addr)
Delete(pool.beats, addr)
} else {
// Otherwise postpone any invalidated transactions
For _, tx := range invalids {
pool.enqueueTx(tx.Hash(), tx)
}
}
// Update the account nonce if needed
If nonce := tx.Nonce(); pool.pendingState.GetNonce(addr) &gt; nonce {
pool.pendingState.SetNonce(addr, nonce)
}
Return
}
}
// Transaction is in the future queue
// Remove the transaction from the future queue.
If future := pool.queue[addr]; future != nil {
future.Remove(tx)
If future.Empty() {
Delete(pool.queue, addr)
}
}
}



Loop is a gorout of txPool. It is also the main event loop. Waiting and responding to external blockchain events and various report and transaction eviction events.

// loop is the transaction pool's main event loop, waiting for and reacting to
// outside blockchain events as well as for various reporting and transaction
// eviction events.
Func (pool *TxPool) loop() {
Defer pool.wg.Done()

// Start the stats reporting and transaction eviction tickers
Var prevPending, prevQueued, prevStales int

Report := time.NewTicker(statsReportInterval)
Defer report.Stop()

Evict := time.NewTicker(evictionInterval)
Defer evict.Stop()

Journal := time.NewTicker(pool.config.Rejournal)
Defer journal.Stop()

// Track the previous head headers for transaction reorgs
Head := pool.chain.CurrentBlock()

// Keep waiting and reacting to the various events
For {
Select {
// Handle ChainHeadEvent
// Listen to the event at the block header and get the new block header.
/ / Call the reset method
Case ev := &lt;-pool.chainHeadCh:
If ev.Block != nil {
pool.mu.Lock()
If pool.chainconfig.IsHomestead(ev.Block.Number()) {
Pool.homestead = true
}
Pool.reset(head.Header(), ev.Block.Header())
Head = ev.Block

pool.mu.Unlock()
}
// Be unsubscribed due to system stopped
Case &lt;-pool.chainHeadSub.Err():
Return

// Handle stats reporting ticks report is to print some logs
Case &lt;-report.C:
pool.mu.RLock()
Pending, queued := pool.stats()
Stales := pool.priced.stales
pool.mu.RUnlock()

If pending != prevPending || queued != prevQueued || stales != prevStales {
log.Debug("Transaction pool status report", "executable", pending, "queued", queued, "stales", stales)
prevPending, prevQueued, prevStales = pending, queued, stales
}

// Handle inactive account transaction eviction
// Handling timeout transaction information,
Case &lt;-evict.C:
pool.mu.Lock()
For addr := range pool.queue {
// Skip local transactions from the eviction mechanism
If pool.locals.contains(addr) {
Continue
}
// Any non-locals old enough should be removed
If time.Since(pool.beats[addr]) &gt; pool.config.Lifetime {
For _, tx := range pool.queue[addr].Flatten() {
pool.removeTx(tx.Hash())
}
}
}
pool.mu.Unlock()

/ / Handle local transaction journal rotation processing information written on the transaction log.
Case &lt;-journal.C:
If pool.journal != nil {
pool.mu.Lock()
If err := pool.journal.rotate(pool.local()); err != nil {
log.Warn("Failed to rotate local tx journal", "err", err)
}
pool.mu.Unlock()
}
}
}
}


The add method, validates the transaction and inserts it into the future queue. If the transaction replaces a currently existing transaction, it returns the previous transaction so that the external method does not call the promote method. If a newly added transaction is Marked as local, then its sending account will enter the white list, the associated transaction of this account will not be deleted due to price restrictions or other restrictions.

// add validates a transaction and inserts it into the non-executable queue for
// later pending promotion and execution. If the transaction is a replacement for
// an already pending or queued one, it overwrites the previous and returns this
// so outer code doesn't uselessly call promote.
//
// If a newly added transaction is marked as local, its sending account will be
// whitelisted, preventing any associated transaction from being dropped out of
// the pool due to pricing constraints.
Func (pool *TxPool) add(tx *types.Transaction, local bool) (bool, error) {
// If the transaction is already known, discard it
Hash := tx.Hash()
If pool.all[hash] != nil {
log.Trace("Discarding already known transaction", "hash", hash)
Return false, fmt.Errorf("known transaction: %x", hash)
}
// If the transaction fails basic validation, discard it
// If the transaction cannot pass the basic verification, then discard it
If err := pool.validateTx(tx, local); err != nil {
log.Trace("Discarding invalid transaction", "hash", hash, "err", err)
invalidTxCounter.Inc(1)
Return false, err
}
// If the transaction pool is full, discard underpriced transactions
// If the trading pool is full, then delete some low-priced transactions.
If uint64(len(pool.all)) &gt;= pool.config.GlobalSlots+pool.config.GlobalQueue {
// If the new transaction is underpriced, don't accept it
// If the new transaction itself is cheap, then don't accept it
If pool.priced.Underpriced(tx, pool.locals) {
log.Trace("Discarding underpriced transaction", "hash", hash, "price", tx.GasPrice())
underpricedTxCounter.Inc(1)
Return false, ErrUnderpriced
}
// New transaction is better than our worse ones, make room for it
// Otherwise delete the low value and give him space.
Drop := pool.priced.Discard(len(pool.all)-int(pool.config.GlobalSlots+pool.config.GlobalQueue-1), pool.locals)
For _, tx := range drop {
log.Trace("Discarding freshly underpriced transaction", "hash", tx.Hash(), "price", tx.GasPrice())
underpricedTxCounter.Inc(1)
pool.removeTx(tx.Hash())
}
}
		// If the transaction is replacing an already pending one, do directly
		from, _ := types.Sender(pool.signer, tx) // already validated
		if list := pool.pending[from]; list != nil &amp;&amp; list.Overlaps(tx) {
			// Nonce already pending, check if required price bump is met
			// 如果交易对应的Nonce已经在pending队列了,那么产看是否能够替换.
			inserted, old := list.Add(tx, pool.config.PriceBump)
			if !inserted {
				pendingDiscardCounter.Inc(1)
				return false, ErrReplaceUnderpriced
}
			// New transaction is better, replace old one
			if old != nil {
				delete(pool.all, old.Hash())
				pool.priced.Removed()
				pendingReplaceCounter.Inc(1)
}
			pool.all[tx.Hash()] = tx
			pool.priced.Put(tx)
			pool.journalTx(from, tx)

			log.Trace("Pooled new executable transaction", "hash", hash, "from", from, "to", tx.To())
			return old != nil, nil
}
		// New transaction isn't replacing a pending one, push into queue
		// 新交易不能替换pending里面的任意一个交易,那么把他push到futuren 队列里面.
		replace, err := pool.enqueueTx(hash, tx)
If err != nil {
			return false, err
}
		// Mark local addresses and journal local transactions
		if local {
			pool.locals.add(from)
}
		// 如果是本地的交易,会被记录进入journalTx
		pool.journalTx(from, tx)

		log.Trace("Pooled new future transaction", "hash", hash, "from", from, "to", tx.To())
		return replace, nil
}


validateTx 使用一致性规则来检查一个交易是否有效,并采用本地节点的一些启发式的限制.

	// validateTx checks whether a transaction is valid according to the consensus
	// rules and adheres to some heuristic limits of the local node (price and size).
	func (pool *TxPool) validateTx(tx *types.Transaction, local bool) error {
		// Heuristic limit, reject transactions over 32KB to prevent DOS attacks
		if tx.Size() &gt; 32*1024 {
			return ErrOversizedData
}
		// Transactions can't be negative. This may never happen using RLP decoded
		// transactions but may occur if you create a transaction using the RPC.
		if tx.Value().Sign() &lt; 0 {
			return ErrNegativeValue
}
		// Ensure the transaction doesn't exceed the current block limit gas.
		if pool.currentMaxGas.Cmp(tx.Gas()) &lt; 0 {
			return ErrGasLimit
}
		// Make sure the transaction is signed properly
		// 确保交易被正确签名.
		from, err := types.Sender(pool.signer, tx)
If err != nil {
			return ErrInvalidSender
}
		// Drop non-local transactions under our own minimal accepted gas price
		local = local || pool.locals.contains(from) // account may be local even if the transaction arrived from the network
		// 如果不是本地的交易,并且GasPrice低于我们的设置,那么也不会接收.
		if !local &amp;&amp; pool.gasPrice.Cmp(tx.GasPrice()) &gt; 0 {
			return ErrUnderpriced
}
		// Ensure the transaction adheres to nonce ordering
		// 确保交易遵守了Nonce的顺序
		if pool.currentState.GetNonce(from) &gt; tx.Nonce() {
			return ErrNonceTooLow
}
		// Transactor should have enough funds to cover the costs
		// cost == V + GP * GL
		// 确保用户有足够的余额来支付.
		if pool.currentState.GetBalance(from).Cmp(tx.Cost()) &lt; 0 {
			return ErrInsufficientFunds
}
		intrGas := IntrinsicGas(tx.Data(), tx.To() == nil, pool.homestead)
		// 如果交易是一个合约创建或者调用. 那么看看是否有足够的 初始Gas.
		if tx.Gas().Cmp(intrGas) &lt; 0 {
			return ErrIntrinsicGas
}
Return nil
}
</pre></body></html>