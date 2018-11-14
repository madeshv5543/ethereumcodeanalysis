
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>The core/state package provides a layer of cache for the state trie of Ethereum.

The structure of state is mainly as shown below

![image](picture/state_1.png)

The blue rectangle represents the module and the gray rectangle represents the external module.

- database mainly provides an abstraction of the trie tree, providing a cache of the trie tree and a cache of contract code length.
- The journal mainly provides the operation log and the function of the operation rollback.
- state_object is an abstraction of the account object that provides some functionality for the account.
- statedb mainly provides some of the functions of the state trie.

## database.go
Database.go provides an abstraction of the database.

data structure

// Database wraps access to tries and contract code.
Type Database interface {
// Accessing tries:
// OpenTrie opens the main account trie.
// OpenStorageTrie opens the storage trie of an account.
// OpenTrie opens the trie tree of the main account
// OpenStorageTrie opens a storage trie for an account
OpenTrie(root common.Hash) (Trie, error)
OpenStorageTrie(addrHash, root common.Hash) (Trie, error)
// Accessing contract code:
// access contract code
ContractCode(addrHash, codeHash common.Hash) ([]byte, error)
// The size of the access contract. This method may be called frequently. Because there is a cache.
ContractCodeSize(addrHash, codeHash common.Hash) (int, error)
// CopyTrie returns an independent copy of the given trie.
// CopyTrie returns a separate copy of the specified trie
CopyTrie(Trie) Trie
}

// NewDatabase creates a backing store for state. The returned database is safe for
// concurrent use and retains cached trie nodes in memory.
Func NewDatabase(db ethdb.Database) Database {
Csc, _ := lru.New(codeSizeCacheSize)
Return &amp;cachingDB{db: db, codeSizeCache: csc}
}

Type cachingDB struct {
Db ethdb.Database
Mu sync.Mutex
pastTries []*trie.SecureTrie //trie tree cache
codeSizeCache *lru.Cache //Cache code size cache
}



OpenTrie, look up from the cache. If you find a copy of the trie that returned the cache, otherwise rebuild a tree to return.


Func (db *cachingDB) OpenTrie(root common.Hash) (Trie, error) {
db.mu.Lock()
Defer db.mu.Unlock()

For i := len(db.pastTries) - 1; i &gt;= 0; i-- {
If db.pastTries[i].Hash() == root {
Return cachedTrie{db.pastTries[i].Copy(), db}, nil
}
}
Tr, err := trie.NewSecure(root, db.db, MaxTrieCacheGen)
If err != nil {
Return nil, err
}
Return cachedTrie{tr, db}, nil
}

Func (db *cachingDB) OpenStorageTrie(addrHash, root common.Hash) (Trie, error) {
Return trie.NewSecure(root, db.db, 0)
}


ContractCode and ContractCodeSize, ContractCodeSize has a cache.


Func (db *cachingDB) ContractCode(addrHash, codeHash common.Hash) ([]byte, error) {
Code, err := db.db.Get(codeHash[:])
If err == nil {
db.codeSizeCache.Add(codeHash, len(code))
}
Return code, err
}

Func (db *cachingDB) ContractCodeSize(addrHash, codeHash common.Hash) (int, error) {
If cached, ok := db.codeSizeCache.Get(codeHash); ok {
Return cached.(int), nil
}
Code, err := db.ContractCode(addrHash, codeHash)
If err == nil {
db.codeSizeCache.Add(codeHash, len(code))
}
Return len(code), err
}

The structure of the cachedTrie and the commit method, when the commit is called, the pushTrie method is called to cache the previous Trie tree.

// cachedTrie inserts its trie into a cachingDB on commit.
Type cachedTrie struct {
*trie.SecureTrie
Db *cachingDB
}

Func (m cachedTrie) CommitTo(dbw trie.DatabaseWriter) (common.Hash, error) {
Root, err := m.SecureTrie.CommitTo(dbw)
If err == nil {
m.db.pushTrie(m.SecureTrie)
}
Return root, err
}
Func (db *cachingDB) pushTrie(t *trie.SecureTrie) {
db.mu.Lock()
Defer db.mu.Unlock()

If len(db.pastTries) &gt;= maxPastTries {
Copy(db.pastTries, db.pastTries[1:])
db.pastTries[len(db.pastTries)-1] = t
} else {
db.pastTries = append(db.pastTries, t)
}
}


## journal.go
Journal represents the operation log and provides a corresponding rollback function for the logs of various operations. Some transaction type operations can be done based on this log.

The type definition defines the interface of journalEntry and provides the function of undo. Journal is a list of journalEntry.

Type journalEntry interface {
Undo(*StateDB)
}

Type journal []journalEntry


A variety of different log types and undo methods.

createObjectChange struct { //Create a log of the object. The undo method deletes the created object from the StateDB.
Account *common.Address
}
Func (ch createObjectChange) undo(s *StateDB) {
Delete(s.stateObjects, *ch.account)
Delete(s.stateObjectsDirty, *ch.account)
}
// For the modification of stateObject, the undo method is to change the value to the original object.
resetObjectChange struct {
Prev *stateObject
}
Func (ch resetObjectChange) undo(s *StateDB) {
s.setStateObject(ch.prev)
}
// Suicide changes. Suicide should be to delete the account, but if there is no commit, the object has not been deleted from the stateDB.
SuicideChange struct {
Account *common.Address
Prev bool // whether account had already suicided
Prevbalance *big.Int
}
Func (ch suicideChange) undo(s *StateDB) {
Obj := s.getStateObject(*ch.account)
If obj != nil {
Obj.suicided = ch.prev
obj.setBalance(ch.prevbalance)
}
}

// Changes to individual accounts.
balanceChange struct {
Account *common.Address
Prev *big.Int
}
nonceChange struct {
Account *common.Address
Prev uint64
}
StorageChange struct {
Account *common.Address
Key, prevalue common.Hash
}
codeChange struct {
Account *common.Address
Prevcode, prevhash []byte
}

Func (ch balanceChange) undo(s *StateDB) {
s.getStateObject(*ch.account).setBalance(ch.prev)
}
Func (ch nonceChange) undo(s *StateDB) {
s.getStateObject(*ch.account).setNonce(ch.prev)
}
Func (ch codeChange) undo(s *StateDB) {
s.getStateObject(*ch.account).setCode(common.BytesToHash(ch.prevhash), ch.prevcode)
}
Func (ch storageChange) undo(s *StateDB) {
s.getStateObject(*ch.account).setState(ch.key, ch.prevalue)
}

// I understand that it is a refund processing for DAO events.
Refunding change struct {
Prev *big.Int
}
Func (ch refundChange) undo(s *StateDB) {
S.refund = ch.prev
}
// Added log modification
addLogChange struct {
Txhash common.Hash
}
Func (ch addLogChange) undo(s *StateDB) {
Logs := s.logs[ch.txhash]
If len(logs) == 1 {
Delete(s.logs, ch.txhash)
} else {
S.logs[ch.txhash] = logs[:len(logs)-1]
}
s.logSize--
}
// This is to increase the original byte[] of SHA3 seen by the VM, and increase the correspondence between SHA3 hash -&gt; byte[]
addPreimageChange struct {
Hash common.Hash
}
Func (ch addPreimageChange) undo(s *StateDB) {
Delete(s.preimages, ch.hash)
}

touchChange struct {
Account *common.Address
Prev bool
prevDirty bool
}
Var ripemd = common.HexToAddress("0000000000000000000000000000000000000003")
Func (ch touchChange) undo(s *StateDB) {
If !ch.prev &amp;&amp; *ch.account != ripemd {
s.getStateObject(*ch.account).touched = ch.prev
If !ch.prevDirty {
Delete(s.stateObjectsDirty, *ch.account)
}
}
}



## state_object.go
stateObject represents the Ethereum account being modified.

data structure


Type Storage map[common.Hash]common.Hash

// stateObject represents an Ethereum account which is being modified.
// stateObject indicates the Ethereum account being modified.
// The usage pattern is as follows:
// First you need to obtain a state object.
// Account values ​​can be accessed and modified through the object.
// Finally, call CommitTrie to write the modified storage trie into a database.

The usage patterns are as follows:
First you need to get a state_object.
Account values ​​can be accessed and modified by objects.
Finally, call CommitTrie to write the modified storage trie to the database.

Type stateObject struct {
Address common.Address
addrHash common.Hash // hash of ethereum address of the account hash value of the Ethereum account address
Data Account // This is the actual Ethereum account information
Db *StateDB / / state database

// DB error.
// State objects are used by the consensus core and VM which are
// unable to deal with database-level errors. Any error that occurs
// during a database read is memoized here and will eventually be returned
// by StateDB.Commit.
//
Database error.
The stateObject is used by the core of the consensus algorithm and the VM, and database-level errors cannot be handled inside the code.
Any errors that occur during database reads are stored here and will eventually be returned by StateDB.Commit.
dbErr error

// Write caches. Write cache
Trie Trie // storage trie, which becomes non-nil on first access The user's storage trie becomes non-empty on the first access
Code Code // contract bytecode, which gets set when code is loaded The contract code is set when the code is loaded

cachedStorage Storage // Storage entry cache to avoid duplicate reads A cache of user storage objects used to avoid repeated reads
dirtyStorage Storage // Storage entries that need to be flushed to disk User storage objects that need to be flushed to disk

// Cache flags. Cache flag
// When an object is marked suicided it will be deleted from the trie
// during the "update" phase of the state transition.
// When an object is marked as suicide, it will be removed from the tree during the "update" phase of the state transition.
dirtyCode bool // true if the code was updated is set to true if the code is updated
Suicided bool
Touched bool
Deleted bool
onDirty func(addr common.Address) // Callback method to mark a state object newly dirty will be called the first time it is set to drity.
}

// Account is the Ethereum consensus representation of accounts.
// These objects are stored in the main account trie.
// The account is an account represented by the Ethereum consensus. These objects are stored in the main account trie.
Type Account struct {
Nonce uint64
Balance *big.Int
Root common.Hash // merkle root of the storage trie
CodeHash []byte
}

Constructor

// newObject creates a state object.
Func newObject(db *StateDB, address common.Address, data Account, onDirty func(addr common.Address)) *stateObject {
If data.Balance == nil {
data.Balance = new(big.Int)
}
If data.CodeHash == nil {
data.CodeHash = emptyCodeHash
}
Return &amp;stateObject{
Db: db,
Address: address,
addrHash: crypto.Keccak256Hash(address[:]),
Data: data,
cachedStorage: make(Storage),
dirtyStorage: make(Storage),
onDirty: onDirty,
}
}


The encoding method of RLP only encodes the Account object.

// EncodeRLP implements rlp.Encoder.
Func (c *stateObject) EncodeRLP(w io.Writer) error {
Return rlp.Encode(w, c.data)
}

Some functions that change state.

Func (self *stateObject) markSuicided() {
Self.suicided = true
If self.onDirty != nil {
self.onDirty(self.Address())
self.onDirty = nil
}
}

Func (c *stateObject) touch() {
C.db.journal = append(c.db.journal, touchChange{
Account: &amp;c.address,
Prev: c.touched,
prevDirty: c.onDirty == nil,
})
If c.onDirty != nil {
c.onDirty(c.Address())
c.onDirty = nil
}
C.touched = true
}


Storage processing

// getTrie returns the account's Storage Trie
Func (c *stateObject) getTrie(db Database) Trie {
If c.trie == nil {
Var err error
C.trie, err = db.OpenStorageTrie(c.addrHash, c.data.Root)
If err != nil {
C.trie, _ = db.OpenStorageTrie(c.addrHash, common.Hash{})
c.setError(fmt.Errorf("can't create storage trie: %v", err))
}
}
Return c.trie
}

// GetState returns a value in account storage.
// GetState returns a value of account storage whose type is a hash type.
// Explain that only account storage can store hash values?
// If the cache exists, it will be looked up from the cache, otherwise it will be queried from the database. Then store it in the cache.
Func (self *stateObject) GetState(db Database, key common.Hash) common.Hash {
Value, exists := self.cachedStorage[key]
If exists {
Return value
}
// Load from DB in case it is missing.
Enc, err := self.getTrie(db).TryGet(key[:])
If err != nil {
self.setError(err)
Return common.Hash{}
}
If len(enc) &gt; 0 {
_, content, _, err := rlp.Split(enc)
If err != nil {
self.setError(err)
}
value.SetBytes(content)
}
If (value != common.Hash{}) {
self.cachedStorage[key] = value
}
Return value
}

// SetState updates a value in account storage.
// Set a value to the account storeage The type of key value is a hash type.
Func (self *stateObject) SetState(db Database, key, value common.Hash) {
Self.db.journal = append(self.db.journal, storageChange{
Account: &amp;self.address,
Key: key,
Prevalue: self.GetState(db, key),
})
self.setState(key, value)
}

Func (self *stateObject) setState(key, value common.Hash) {
self.cachedStorage[key] = value
self.dirtyStorage[key] = value

If self.onDirty != nil {
self.onDirty(self.Address())
self.onDirty = nil
}
}


Submit Commit

// CommitTrie the storage trie of the object to dwb.
// This updates the trie root.
// Steps, first open, then modify, then commit or rollback
Func (self *stateObject) CommitTrie(db Database, dbw trie.DatabaseWriter) error {
self.updateTrie(db) // updateTrie writes the modified cache to the Trie tree
If self.dbErr != nil {
Return self.dbErr
}
Root, err := self.trie.CommitTo(dbw)
If err == nil {
self.data.Root = root
}
Return err
}

// updateTrie writes cached storage modifications into the object's storage trie.
Func (self *stateObject) updateTrie(db Database) Trie {
Tr := self.getTrie(db)
For key, value := range self.dirtyStorage {
Delete(self.dirtyStorage, key)
If (value == common.Hash{}) {
self.setError(tr.TryDelete(key[:]))
Continue
}
// Encoding []byte cannot fail, ok to ignore the error.
v, _ := rlp.EncodeToBytes(bytes.TrimLeft(value[:], "\x00"))
self.setError(tr.TryUpdate(key[:], v))
}
Return tr
}

// UpdateRoot sets the trie root to the current root hash of
// Set the root of the account to the current trie tree.
Func (self *stateObject) updateRoot(db Database) {
self.updateTrie(db)
self.data.Root = self.trie.Hash()
}


With some additional features, deepCopy provides a deep copy of state_object.


Func (self *stateObject) deepCopy(db *StateDB, onDirty func(addr common.Address)) *stateObject {
stateObject := newObject(db, self.address, self.data, onDirty)
If self.trie != nil {
stateObject.trie = db.db.CopyTrie(self.trie)
}
stateObject.code = self.code
stateObject.dirtyStorage = self.dirtyStorage.Copy()
stateObject.cachedStorage = self.dirtyStorage.Copy()
stateObject.suicided = self.suicided
stateObject.dirtyCode = self.dirtyCode
stateObject.deleted = self.deleted
Return stateObject
}


## statedb.go

stateDB is used to store everything about merkle trie in Ethereum. StateDB is responsible for caching and storing nested state. This is the general query interface for retrieving contracts and accounts:

data structure

Type StateDB struct {
Db Database // backend database
Trie Trie // trie tree main account trie

// This map holds 'live' objects, which will get modified while processing a state transition.
// The following Map is used to store the currently active objects, which will be modified when the state transitions.
// stateObjects is used to cache objects
// stateObjectsDirty is used to cache the modified object.
stateObjects map[common.Address]*stateObject
stateObjectsDirty map[common.Address]struct{}

// DB error.
// State objects are used by the consensus core and VM which are
// unable to deal with database-level errors. Any error that occurs
// during a database read is memoized here and will eventually be returned
// by StateDB.Commit.
dbErr error

// The refund counter, also used by state transitioning.
//The counter is refunded. The function is still unclear for the time being.
Refund *big.Int

Thash, bhash common.Hash // current transaction hash and block hash
txIndex int // the index of the current transaction
Logs map[common.Hash][]*types.Log // log key is the hash value of the transaction
logSize uint

Preimages map[common.Hash][]byte // Mapping of SHA3-&gt;byte[] calculated by EVM

// Journal of state modifications. This is the backbone of
// Snapshot and RevertToSnapshot.
// Status modification log. This is the backbone of Snapshot and RevertToSnapshot.
Journal journal
validRevisions []revision
nextRevisionId int

Lock sync.Mutex
}


Constructor

// General usage statedb, _ := state.New(common.Hash{}, state.NewDatabase(db))

// Create a new state from a given trie
Func New(root common.Hash, db Database) (*StateDB, error) {
Tr, err := db.OpenTrie(root)
If err != nil {
Return nil, err
}
Return &amp;StateDB{
Db: db,
Trie: tr,
stateObjects: make(map[common.Address]*stateObject),
stateObjectsDirty: make(map[common.Address]struct{}),
Refund: new(big.Int),
Logs: make(map[common.Hash][]*types.Log),
Preimages: make(map[common.Hash][]byte),
}, nil
}

### Processing for Log
State provides Log processing, which is quite unexpected, because Log is actually stored in the blockchain, not stored in the state trie, state provides Log processing, using several functions based on the following. The strange thing is that I haven't seen how to delete the information in the log. If I don't delete it, it should accumulate more. TODO logs delete

The Prepare function is executed at the beginning of the transaction execution.

The AddLog function is executed by the VM during transaction execution. Add a log. At the same time, the log is associated with the transaction, and the information of the partial transaction is added.

GetLogs function, the transaction is completed and taken away.


// Prepare sets the current transaction hash and index and block hash which is
// used when the EVM emits new state logs.
Func (self *StateDB) Prepare(thash, bhash common.Hash, ti int) {
Self.thash = thash
Self.bhash = bhash
self.txIndex = ti
}

Func (self *StateDB) AddLog(log *types.Log) {
Self.journal = append(self.journal, addLogChange{txhash: self.thash})

log.TxHash = self.thash
log.BlockHash = self.bhash
log.TxIndex = uint(self.txIndex)
log.Index = self.logSize
Self.logs[self.thash] = append(self.logs[self.thash], log)
self.logSize++
}
Func (self *StateDB) GetLogs(hash common.Hash) []*types.Log {
Return self.logs[hash]
}

Func (self *StateDB) Logs() []*types.Log {
Var logs []*types.Log
For _, lgs := range self.logs {
Logs = append(logs, lgs...)
}
Return logs
}


### stateObjectProcessing
getStateObject, first obtained from the cache, if not from the trie tree, and loaded into the cache.

// Retrieve a state object given my the address. Returns nil if not found.
Func (self *StateDB) getStateObject(addr common.Address) (stateObject *stateObject) {
// Prefer 'live' objects.
If obj := self.stateObjects[addr]; obj != nil {
If obj.deleted {
Return nil
}
Return obj
}

// Load the object from the database.
Enc, err := self.trie.TryGet(addr[:])
If len(enc) == 0 {
self.setError(err)
Return nil
}
Var data Account
If err := rlp.DecodeBytes(enc, &amp;data); err != nil {
log.Error("Failed to decode state object", "addr", addr, "err", err)
Return nil
}
// Insert into the live set.
Obj := newObject(self, addr, data, self.MarkStateObjectDirty)
self.setStateObject(obj)
Return obj
}

MarkStateObjectDirty, set a stateObject to Dirty. Insert an empty structure directly into the address corresponding to stateObjectDirty.

// MarkStateObjectDirty adds the specified object to the dirty map to avoid costly
// state object cache iteration to find a handful of modified ones.
Func (self *StateDB) MarkStateObjectDirty(addr common.Address) {
self.stateObjectsDirty[addr] = struct{}{}
}


### Snapshot and rollback features
Snapshot can create a snapshot and then roll back to which state via RevertToSnapshot. This function is done by journal. Each step of the modification will add an undo log to the journal. If you need to roll back, you only need to execute the undo log.

// Snapshot returns an identifier for the current revision of the state.
Func (self *StateDB) Snapshot() int {
Id := self.nextRevisionId
self.nextRevisionId++
self.validRevisions = append(self.validRevisions, revision{id, len(self.journal)})
Return id
}

// RevertToSnapshot reverts all state changes made since the given revision.
Func (self *StateDB) RevertToSnapshot(revid int) {
// Find the snapshot in the stack of valid snapshots.
Idx := sort.Search(len(self.validRevisions), func(i int) bool {
Return self.validRevisions[i].id &gt;= revid
})
If idx == len(self.validRevisions) || self.validRevisions[idx].id != revid {
Panic(fmt.Errorf("revision id %v cannot be reverted", revid))
}
Snapshot := self.validRevisions[idx].journalIndex

// Replay the journal to undo changes.
For i := len(self.journal) - 1; i &gt;= snapshot; i-- {
Self.journal[i].undo(self)
}
Self.journal = self.journal[:snapshot]

// Remove invalidated snapshots from the stack.
self.validRevisions = self.validRevisions[:idx]
}

### Get the root hash value of the intermediate state
IntermediateRoot is used to calculate the hash value of the root of the current state trie. This method will be called during the execution of the transaction. Will be stored in the transaction receipt

The Finalise method will call the update method to write the changes stored in the cache layer to the trie database. But this time has not yet written to the underlying database. The commit has not been called yet, the data is still in memory, and it has not been filed.

// Finalise finalises the state by removing the self destructed objects
// and clears the journal as well as the refunds.
Func (s *StateDB) Finalise(deleteEmptyObjects bool) {
For addr := range s.stateObjectsDirty {
stateObject := s.stateObjects[addr]
If stateObject.suicided || (deleteEmptyObjects &amp;&amp; stateObject.empty()) {
s.deleteStateObject(stateObject)
} else {
stateObject.updateRoot(s.db)
s.updateStateObject(stateObject)
}
}
// Invalidate journal because reverting across transactions is not allowed.
s.clearJournalAndRefund()
}

// IntermediateRoot computes the current root hash of the state trie.
// It is called in between transactions to get the root hash that
// goes into transaction receipts.
Func (s *StateDB) IntermediateRoot(deleteEmptyObjects bool) common.Hash {
s.Finalise(deleteEmptyObjects)
Return s.trie.Hash()
}

### commit method
CommitTo is used to submit changes.

// CommitTo writes the state to the given database.
Func (s *StateDB) CommitTo(dbw trie.DatabaseWriter, deleteEmptyObjects bool) (root common.Hash, err error) {
Defer s.clearJournalAndRefund()

// Commit objects to the trie.
For addr, stateObject := range s.stateObjects {
_, isDirty := s.stateObjectsDirty[addr]
Switch {
Case stateObject.suicided || (isDirty &amp;&amp; deleteEmptyObjects &amp;&amp; stateObject.empty()):
// If the object has been removed, don't bother syncing it
// and just mark it for deletion in the trie.
s.deleteStateObject(stateObject)
Case isDirty:
// Write any contract code associated with the state object
If stateObject.code != nil &amp;&amp; stateObject.dirtyCode {
If err := dbw.Put(stateObject.CodeHash(), stateObject.code); err != nil {
Return common.Hash{}, err
}
stateObject.dirtyCode = false
}
// Write any storage changes in the state object to its storage trie.
If err := stateObject.CommitTrie(s.db, dbw); err != nil {
Return common.Hash{}, err
}
// Update the object in the main account trie.
s.updateStateObject(stateObject)
}
Delete(s.stateObjectsDirty, addr)
}
// Write trie changes.
Root, err = s.trie.CommitTo(dbw)
log.Debug("Trie cache stats after commit", "misses", trie.CacheMisses(), "unloads", trie.CacheUnloads())
Return root, err
}



### to sum up
The state package provides the functionality of state management for users and contracts. Manages various state transitions for states and contracts. Cache, trie, database. Log and rollback features.

</pre></body></html>