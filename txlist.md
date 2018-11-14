
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>## nonceHeap
nonceHeap implements a data structure of heap.Interface to implement a heap data structure. In the documentation for heap.Interface, the default implementation is the smallest heap.

If h is an array, as long as the data in the array meets the following requirements. Then think that h is a minimum heap.

!h.Less(j, i) for 0 &lt;= i &lt; h.Len() and 2*i+1 &lt;= j &lt;= 2*i+2 and j &lt; h.Len()
// Treat the array as a full binary tree, the first element is the root of the tree, and the second and third elements are the two branches of the root of the tree.
// This is pushed down in turn. Then if the root of the tree is i then its two branches are 2*i+2 and 2*i + 2.
// The definition of the smallest heap is that an arbitrary tree root cannot be larger than its two branches. This is the definition of the code description above.
Definition of heap.Interface

We only need to define the data structure that satisfies the following interfaces, and we can use some methods of heap to implement the heap structure.
Type Interface interface {
sort.Interface
Push(x interface{}) // add x as element Len() Increase x to the end
Pop() interface{} // remove and return element Len() - 1. Remove and return the last element
}

Code analysis of nonceHeap.

// nonceHeap is a heap.Interface implementation over 64bit unsigned integers for
// retrieving sorted transactions from the likelihood gapped future queue.
Type nonceHeap []uint64

Func (h nonceHeap) Len() int { return len(h) }
Func (h nonceHeap) Less(i, j int) bool { return h[i] &lt; h[j] }
Func (h nonceHeap) Swap(i, j int) { h[i], h[j] = h[j], h[i] }

Func (h *nonceHeap) Push(x interface{}) {
*h = append(*h, x.(uint64))
}

Func (h *nonceHeap) Pop() interface{} {
Old := *h
n := len(old)
x := old[n-1]
*h = old[0 : n-1]
Return x
}


## txSortedMap

txSortedMap stores all transactions under the same account.

structure

// txSortedMap is a nonce-&gt;transaction hash map with a heap based index to allow
// iterating over the contents in a nonce-incrementing way.
// txSortedMap is a hashmap of a nonce-&gt;transaction with a heap-based index.
// Allow content to be iterated in a nonce increment.

Type Transactions []*Transaction

Type txSortedMap struct {
Items map[uint64]*types.Transaction // Hash map storing the transaction data
Index *nonceHeap // Heap of nonces of all the stored transactions (non-strict mode)
Cache types.Transactions // Cache of the transactions already sorted is used to cache transactions that have been sorted.
}

Put and Get, Get are used to get the transaction of the specified nonce, Put is used to insert the transaction into the map.

// Get retrieves the current transactions associated with the given nonce.
Func (m *txSortedMap) Get(nonce uint64) *types.Transaction {
Return m.items[nonce]
}

// Put inserts a new transaction into the map, also updating the map's nonce
// index. If a transaction already exists with the same nonce, it's overwritten.
// Insert a new transaction into the map and update the map's nonce index. If a transaction already exists, it is overwritten. At the same time any cached data will be deleted.
Func (m *txSortedMap) Put(tx *types.Transaction) {
Nonce := tx.Nonce()
If m.items[nonce] == nil {
heap.Push(m.index, nonce)
}
M.items[nonce], m.cache = tx, nil
}

Forward is used to delete all transactions where the nonce is less than threshold. Then return all the removed transactions.

// Forward removes all transactions from the map with a nonce lower than the
// provided threshold. Every removed transaction is returned for any post-removal
// maintenance.
Func (m *txSortedMap) Forward(threshold uint64) types.Transactions {
Var removed types.Transactions

// Pop off heap items until the threshold is reached
For m.index.Len() &gt; 0 &amp;&amp; (*m.index)[0] &lt; threshold {
Nonce := heap.Pop(m.index).(uint64)
Removed = append(removed, m.items[nonce])
Delete(m.items, nonce)
}
// If we had a cached order, shift the front
// cache is a well-ordered transaction.
If m.cache != nil {
M.cache = m.cache[len(removed):]
}
Return removed
}

Filter, delete all transactions that cause the filter function call to return true, and return those transactions.

// Filter iterates over the list of transactions and removes all of them for which
// the specified function evaluates to true.
Func (m *txSortedMap) Filter(filter func(*types.Transaction) bool) types.Transactions {
Var removed types.Transactions

// Collect all the transactions to filter out
For nonce, tx := range m.items {
If filter(tx) {
Removed = append(removed, tx)
Delete(m.items, nonce)
}
}
// If transactions were removed, the heap and cache are ruined
// If the transaction is deleted, the heap and cache are destroyed
If len(removed) &gt; 0 {
*m.index = make([]uint64, 0, len(m.items))
For nonce := range m.items {
*m.index = append(*m.index, nonce)
}
// need to rebuild the heap
heap.Init(m.index)
/ / Set the cache to nil
M.cache = nil
}
Return removed
}

Cap has a limit on the number of items and returns all transactions that exceed the limit.

// Cap places a hard limit on the number of items, returning all transactions
// exceeding that limit.
// Cap has a limit on the number of items and returns all transactions that exceed the limit.
Func (m *txSortedMap) Cap(threshold int) types.Transactions {
// Short circuit if the number of items is under the limit
If len(m.items) &lt;= threshold {
Return nil
}
// Otherwise gather and drop the highest nonce'd transactions
Var drops types.Transactions

sort.Sort(*m.index) //Sort from small to large Delete from the end.
For size := len(m.items); size &gt; threshold; size-- {
Drops = append(drops, m.items[(*m.index)[size-1]])
Delete(m.items, (*m.index)[size-1])
}
*m.index = (*m.index)[:threshold]
// Rebuild the heap
heap.Init(m.index)

// If we had a cache, shift the back
If m.cache != nil {
M.cache = m.cache[:len(m.cache)-len(drops)]
}
Return drops
}

Remove

// Remove deletes a transaction from the maintained map, returning whether the
// transaction was found.
//
Func (m *txSortedMap) Remove(nonce uint64) bool {
// Short circuit if no transaction is present
_, ok := m.items[nonce]
If !ok {
Return false
}
// Otherwise delete the transaction and fix the heap index
For i := 0; i &lt; m.index.Len(); i++ {
If (*m.index)[i] == nonce {
heap.Remove(m.index, i)
Break
}
}
Delete(m.items, nonce)
M.cache = nil

Return true
}

Ready function

// Ready retrieves a sequentially increasing list of transactions starting at the
// provided nonce that is ready for processing. The returned transactions will be
// removed from the list.
// Ready returns a continuous transaction starting from the specified nonce. The returned trade will be deleted.
// Note, all transactions with nonces lower than start will also be returned to
// prevent getting into and invalid state. This is not something that should ever
// happen but better to be self correcting than failing!
// Note that please note that all transactions with a nonce below start will also be returned to prevent entry and invalidation.
// This is not something that should happen, but self-correction rather than failure!
Func (m *txSortedMap) Ready(start uint64) types.Transactions {
// Short circuit if no services are available
If m.index.Len() == 0 || (*m.index)[0] &gt; start {
Return nil
}
// Otherwise start accumulating incremental transactions
Var ready types.Transactions
// From the very beginning, one by one,
For next := (*m.index)[0]; m.index.Len() &gt; 0 &amp;&amp; (*m.index)[0] == next; next++ {
Ready = append(ready, m.items[next])
Delete(m.items, next)
heap.Pop(m.index)
}
M.cache = nil

Return ready
}

Flatten, returns a list of transactions based on nonce sorting. And cached into the cache field, so that it can be used repeatedly without modification.

// Len returns the length of the transaction map.
Func (m *txSortedMap) Len() int {
Return len(m.items)
}

// Flatten creates a nonce-sorted slice of transactions based on the loosely
// sorted internal representation. The result of the sorting is cached in case
// it's requested again before any modifications are made to the contents.
Func (m *txSortedMap) Flatten() types.Transactions {
// If the sorting was not cached yet, create and cache it
If m.cache == nil {
M.cache = make(types.Transactions, 0, len(m.items))
For _, tx := range m.items {
M.cache = append(m.cache, tx)
}
sort.Sort(types.TxByNonce(m.cache))
}
// Copy the cache to prevent accidental modifications
Txs := make(types.Transactions, len(m.cache))
Copy(txs, m.cache)
Return txs
}

## txList
txList is a list of transactions belonging to the same account, sorted by nonce. Can be used to store continuous executable transactions. For non-continuous transactions, there are some small different behaviors.

structure

// txList is a "list" of transactions belonging to an account, sorted by account
// samece. The same type can be used both for storing contiguous transactions for
// the executable/pending queue; and for storing gapped transactions for the non-
// executable/future queue, with minor behavioral changes.
Type txList struct {
Strict bool // whether nonces are strictly continuous or not nonces is strictly continuous or non-continuous
Txs *txSortedMap // Heap indexed sorted hash map of the transactions

Costcap *big.Int // Price of the highest costing transaction (reset only if exceeds balance) The highest value of GasPrice * GasLimit in all transactions
Gascap *big.Int // Gas limit of the highest spending transaction (reset only if exceeds block limit) The highest value of GasPrice in all transactions
}
Overlaps Returns whether a given transaction has a transaction with the same nonce.

// Overlaps returns whether the transaction specified has the same nonce as one
// already contained within the list.
//
Func (l *txList) Overlaps(tx *types.Transaction) bool {
Return l.txs.Get(tx.Nonce()) != nil
}
Add performs an operation that replaces the old transaction if the new transaction is higher than the GasPrice value of the old transaction by a certain priceBump.

//Add tries to insert a new transaction into the list, returning whether the
// transaction was accepted, and if yes, any previous transaction it replaced.
// Add attempts to insert a new transaction, returning whether the transaction was received, and if received, any previous transactions will be replaced.
// If the new transaction is accepted into the list, the lists' cost and gas
//thresholds are also potentially updated.
// If a new transaction is received, the total cost and gas limits will be updated.
Func (l *txList) Add(tx *types.Transaction, priceBump uint64) (bool, *types.Transaction) {
// If there's an older better transaction, abort
// If there is an old transaction. And the price of the new deal is higher than the old one. Then replace.
Old := l.txs.Get(tx.Nonce())
If old != nil {
Threshold := new(big.Int).Div(new(big.Int).Mul(old.GasPrice(), big.NewInt(100+int64(priceBump))), big.NewInt(100))
If threshold.Cmp(tx.GasPrice()) &gt;= 0 {
Return false, nil
}
}
// Otherwise overwrite the old transaction with the current one
l.txs.Put(tx)
If cost := tx.Cost(); l.costcap.Cmp(cost) &lt; 0 {
L.costcap = cost
}
If gas := tx.Gas(); l.gascap.Cmp(gas) &lt; 0 {
L.gascap = gas
}
Return true, old
}
Forward Deletes all transactions where the nonce is less than a certain value.

// Forward removes all transactions from the list with a nonce lower than the
// provided threshold. Every removed transaction is returned for any post-removal
// maintenance.
Func (l *txList) Forward(threshold uint64) types.Transactions {
Return l.txs.Forward(threshold)
}

Filter,

// Filter removes all transactions from the list with a cost or gas limit higher
// than the provided thresholds. Every removed transaction is returned for any
// post-removal maintenance. Strict-mode invalidated transactions are also
// returned.
// Filter removes all transactions that are higher than the provided cost or gasLimit. The removed trade will be returned for further processing. In strict mode, all invalid transactions are also returned.
//
// This method uses the cached costcap and gascap to quickly decide if there's even
// a point in calculating all the costs or if the balance covers all. If the threshold
// is lower than the costgas cap, the caps will be reset to a new high after removing
// the newly invalidated transactions.
// This method uses the cached costcap and gascap to quickly decide if it needs to traverse all transactions. If the limit is less than the costcap and gascap of the cache, the costcap and gascap values ​​are updated after the illegal transaction is removed.

Func (l *txList) Filter(costLimit, gasLimit *big.Int) (types.Transactions, types.Transactions) {
// If all transactions are below the threshold, short circuit
// If all transactions are less than the limit, return directly.
If l.costcap.Cmp(costLimit) &lt;= 0 &amp;&amp; l.gascap.Cmp(gasLimit) &lt;= 0 {
Return nil, nil
}
L.costcap = new(big.Int).Set(costLimit) // Lower the caps to the thresholds
L.gascap = new(big.Int).Set(gasLimit)

// Filter out all the transactions above the account's funds
Removed := l.txs.Filter(func(tx *types.Transaction) bool { return tx.Cost().Cmp(costLimit) &gt; 0 || tx.Gas().Cmp(gasLimit) &gt; 0 })

// If the list was strict, filter anything above the lowest nonce
Var invalids types.Transactions

If l.strict &amp;&amp; len(removed) &gt; 0 {
// All transactions where nonce is greater than the smallest removed nonce are invalidated by the task.
// In strict mode, this transaction is also removed.
Lowest := uint64(math.MaxUint64)
For _, tx := range removed {
If nonce := tx.Nonce(); lowest &gt; nonce {
Lowest = nonce
}
}
Invalids = l.txs.Filter(func(tx *types.Transaction) bool { return tx.Nonce() &gt; lowest })
}
Return removed, invalids
}

The Cap function is used to return more than the number of transactions. If the number of transactions exceeds threshold, then the subsequent transaction is removed and returned.

// Cap places a hard limit on the number of items, returning all transactions
// exceeding that limit.
Func (l *txList) Cap(threshold int) types.Transactions {
Return l.txs.Cap(threshold)
}

Remove, delete the transaction for the given Nonce, if in strict mode, also delete all transactions with nonce greater than the given Nonce and return.

// Remove deletes a transaction from the maintained list, returning whether the
// transaction was found, and also returning any transaction invalidated due to
// the deletion (strict mode only).
Func (l *txList) Remove(tx *types.Transaction) (bool, types.Transactions) {
// Remove the transaction from the set
Nonce := tx.Nonce()
If removed := l.txs.Remove(nonce); !removed {
Return false, nil
}
// In strict mode, filter out non-executable transactions
If l.strict {
Return true, l.txs.Filter(func(tx *types.Transaction) bool { return tx.Nonce() &gt; nonce })
}
Return true, nil
}

Ready, len, Empty, Flatten directly calls the corresponding method of txSortedMap.

// Ready retrieves a sequentially increasing list of transactions starting at the
// provided nonce that is ready for processing. The returned transactions will be
// removed from the list.
//
// Note, all transactions with nonces lower than start will also be returned to
// prevent getting into and invalid state. This is not something that should ever
// happen but better to be self correcting than failing!
Func (l *txList) Ready(start uint64) types.Transactions {
Return l.txs.Ready(start)
}

// Len returns the length of the transaction list.
Func (l *txList) Len() int {
Return l.txs.Len()
}

// Empty returns whether the list of transactions is empty or not.
Func (l *txList) Empty() bool {
Return l.Len() == 0
}

// Flatten creates a nonce-sorted slice of transactions based on the loosely
// sorted internal representation. The result of the sorting is cached in case
// it's requested again before any modifications are made to the contents.
Func (l *txList) Flatten() types.Transactions {
Return l.txs.Flatten()
}


## priceHeap
priceHeap is a minimal heap, built according to the size of the price.

// priceHeap is a heap.Interface implementation over transactions for retrieving
// price-sorted transactions to discard when the pool fills up.
Type priceHeap []*types.Transaction

Func (h priceHeap) Len() int { return len(h) }
Func (h priceHeap) Less(i, j int) bool { return h[i].GasPrice().Cmp(h[j].GasPrice()) &lt; 0 }
Func (h priceHeap) Swap(i, j int) { h[i], h[j] = h[j], h[i] }

Func (h *priceHeap) Push(x interface{}) {
*h = append(*h, x.(*types.Transaction))
}

Func (h *priceHeap) Pop() interface{} {
Old := *h
n := len(old)
x := old[n-1]
*h = old[0 : n-1]
Return x
}


## txPricedList
Data structure and construction, txPricedList is a price-based sorting heap that allows transactions to be processed in a price-incremental manner.


// txPricedList is a price-sorted heap to allow operating on transactions pool
// contents in a price-incrementing way.
Type txPricedList struct {
All *map[common.Hash]*types.Transaction // Pointer to the map of all transactions This is a pointer to a map of all transactions
Items *priceHeap // Heap of prices of all the stored transactions
Stales int // Number of stale price points to (re-heap trigger)
}

// newTxPricedList creates a new price-sorted transaction heap.
Func newTxPricedList(all *map[common.Hash]*types.Transaction) *txPricedList {
Return &amp;txPricedList{
All: all,
Items: new(priceHeap),
}
}

Put

// Put inserts a new transaction into the heap.
Func (l *txPricedList) Put(tx *types.Transaction) {
heap.Push(l.items, tx)
}

Removed

// Removed notifies the prices transaction list that an old transaction dropped
// from the pool. The list will just keep a counter of stale objects and update
// the heap if a small enough ratio of transactions go stale.
// Removed is used to tell txPricedList that an old transaction has been deleted. txPricedList uses a counter to determine when to update the heap information.
Func (l *txPricedList) Removed() {
// Bump the stale counter, but exit if still too low (&lt; 25%)
L.stales++
If l.stales &lt;= len(*l.items)/4 {
Return
}
// Seems we've reached a critical number of stale transactions, reheap
Reheap := make(priceHeap, 0, len(*l.all))

L.stales, l.items = 0, &amp;reheap
For _, tx := range *l.all {
*l.items = append(*l.items, tx)
}
heap.Init(l.items)
}

Cap is used to find all transactions below the given price threshold. Remove them from the priceList and return.

// Cap finds all the transactions below the given price threshold, drops them
// from the priced list and returs them for further removal from the entire pool.
Func (l *txPricedList) Cap(threshold *big.Int, local *accountSet) types.Transactions {
Drop := make(types.Transactions, 0, 128) // Remote underpriced transactions to drop
Save := make(types.Transactions, 0, 64) // Local underpriced transactions to keep

For len(*l.items) &gt; 0 {
// Discard stale transactions if found during cleanup
Tx := heap.Pop(l.items).(*types.Transaction)
If _, ok := (*l.all)[tx.Hash()]; !ok {
/ / If you find a deleted, then update the states counter
L.stales--
Continue
}
// Stop the discards if we've reached the threshold
If tx.GasPrice().Cmp(threshold) &gt;= 0 {
// If the price is not less than the threshold, then exit
Save = append(save, tx)
Break
}
// Non stale transaction found, discard unless local
If local.containsTx(tx) { //local transactions will not be deleted
Save = append(save, tx)
} else {
Drop = append(drop, tx)
}
}
For _, tx := range save {
heap.Push(l.items, tx)
}
Return drop
}


Underpriced, check if tx is cheaper or cheaper than the cheapest deal in current txPricedList.

// Underpriced checks whether a transaction is cheaper than (or as cheap as) the
// lowest priced transaction currently being tracked.
Func (l *txPricedList) Underpriced(tx *types.Transaction, local *accountSet) bool {
// Local transactions cannot be underpriced
If local.containsTx(tx) {
Return false
}
// Discard stale price points if found at the heap start
For len(*l.items) &gt; 0 {
Head := []*types.Transaction(*l.items)[0]
If _, ok := (*l.all)[head.Hash()]; !ok {
L.stales--
heap.Pop(l.items)
Continue
}
Break
}
// Check if the transaction is underpriced or not
If len(*l.items) == 0 {
log.Error("Pricing query for empty pool") // This cannot happen, print to catch programming errors
Return false
}
Cheapest : : []*types.Transaction(*l.items)[0]
Return cheapest.GasPrice().Cmp(tx.GasPrice()) &gt;= 0
}

Discard, find a certain number of the cheapest deals, remove them from the current list and return.

// Discard finds a number of most underpriced transactions, removes them from the
// priced list and returns them for further removal from the entire pool.
Func (l *txPricedList) Discard(count int, local *accountSet) types.Transactions {
Drop := make(types.Transactions, 0, count) // Remote underpriced transactions to drop
Save := make(types.Transactions, 0, 64) // Local underpriced transactions to keep

For len(*l.items) &gt; 0 &amp;&amp; count &gt; 0 {
// Discard stale transactions if found during cleanup
Tx := heap.Pop(l.items).(*types.Transaction)
If _, ok := (*l.all)[tx.Hash()]; !ok {
L.stales--
Continue
}
// Non stale transaction found, discard unless local
If local.containsTx(tx) {
Save = append(save, tx)
} else {
Drop = append(drop, tx)
Count--
}
}
For _, tx := range save {
heap.Push(l.items, tx)
}
Return drop
}


## accountSet
accountSet is a collection of accounts and an object that handles signatures.

// accountSet is simply a set of addresses to check for existence, and a signer
// capable of deriving addresses from transactions.
Type accountSet struct {
Accounts map[common.Address]struct{}
Signer types.Signer
}

// newAccountSet creates a new address set with an associated signer for sender
// derivations.
Func newAccountSet(signer types.Signer) *accountSet {
Return &amp;accountSet{
Accounts: make(map[common.Address]struct{}),
Signer: signer,
}
}

// contains checks if a given address is contained within the set.
Func (as *accountSet) contains(addr common.Address) bool {
_, exist := as.accounts[addr]
Return exist
}

// containsTx checks if the sender of a given tx is within the set. If the sender
// cannot be derived, this method returns false.
// containsTx checks if the sender of the given tx is in the collection. This method returns false if the sender cannot be evaluated.
Func (as *accountSet) containsTx(tx *types.Transaction) bool {
If addr, err := types.Sender(as.signer, tx); err == nil {
Return as.contains(addr)
}
Return false
}

// add inserts a new address into the set to track.
Func (as *accountSet) add(addr common.Address) {
As.accounts[addr] = struct{}{}
}


## txJournal

txJournal is a circular log of transactions that is designed to store locally created transactions to allow unexecuted transactions to continue running after the node is restarted.
structure


// txJournal is a rotating log of transactions with the aim of storing locally
//create transactions to allow non-executed ones to survive node restarts.
Type txJournal struct {
Path string // Filesystem path to store the transactions at The file system path used to store the transaction.
Writer io.WriteCloser // Output stream to write new transactions into The output stream used to write new transactions.
}

newTxJournal, used to create a new transaction log.

// newTxJournal creates a new transaction journal to
Func newTxJournal(path string) *txJournal {
Return &amp;txJournal{
Path: path,
}
}

The load method parses the transaction from disk and then calls the add callback method.

// load parses a transaction journal dump from disk, loading its contents into
// the specified pool.
Func (journal *txJournal) load(add func(*types.Transaction) error) error {
// Skip the parsing if the journal file doens't exist at all
If _, err := os.Stat(journal.path); os.IsNotExist(err) {
Return nil
}
// Open the journal for loading any past transactions
Input, err := os.Open(journal.path)
If err != nil {
Return err
}
Defer input.Close()

// Inject all transactions from the journal into the pool
Stream := rlp.NewStream(input, 0)
Total, dropped := 0, 0

Var failure error
For {
// Parse the next transaction and terminate on error
Tx := new(types.Transaction)
If err = stream.Decode(tx); err != nil {
If err != io.EOF {
Failure = err
}
Break
}
// Import the transaction and bump the appropriate progress counters
Total++
If err = add(tx); err != nil {
log.Debug("Failed to add journaled transaction", "err", err)
Dropped++
Continue
}
}
log.Info("Loaded local transaction journal", "transactions", total, "dropped", dropped)

Return failure
}
Insert method, call rlp.Encode to write to the writer

// insert adds the specified transaction to the local disk journal.
Func (journal *txJournal) insert(tx *types.Transaction) error {
If journal.writer == nil {
Return errNoActiveJournal
}
If err := rlp.Encode(journal.writer, tx); err != nil {
Return err
}
Return nil
}

The rotate method regenerates the transaction based on the current transaction pool.

// rotate regenerates the transaction journal based on the current contents of
// the transaction pool.
Func (journal *txJournal) rotate(all map[common.Address]types.Transactions) error {
// Close the current journal (if any is open)
If journal.writer != nil {
If err := journal.writer.Close(); err != nil {
Return err
}
Journal.writer = nil
}
// Generate a new journal with the contents of the current pool
Replacement, err := os.OpenFile(journal.path+".new", os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0755)
If err != nil {
Return err
}
Journaled := 0
For _, txs := range all {
For _, tx := range txs {
If err = rlp.Encode(replacement, tx); err != nil {
replacement.Close()
Return err
}
}
Journaled += len(txs)
}
replacement.Close()

// Replace the live journal with the newly generated one
If err = os.Rename(journal.path+".new", journal.path); err != nil {
Return err
}
Sink, err := os.OpenFile(journal.path, os.O_WRONLY|os.O_APPEND, 0755)
If err != nil {
Return err
}
Journal.writer = sink
log.Info("Regenerated local transaction journal", "transactions", journaled, "accounts", len(all))

Return nil
}

Close

// close flushes the transaction journal contents to disk and closes the file.
Func (journal *txJournal) close() error {
Var err error

If journal.writer != nil {
Err = journal.writer.Close()
Journal.writer = nil
}
Return err
}
</pre></body></html>