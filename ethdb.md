
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>All data of go-ethereum is stored in levelDB, the open source KeyValue file database of Google. All the data of the entire blockchain is stored in a levelDB database. LevelDB supports the function of splitting files according to file size, so we see The data of the blockchain is a small file. In fact, these small files are all the same levelDB instance. Here is a simple look at the levelDB's go package code.

LevelDB official website introduction features

**Features**:

- key and value are byte arrays of arbitrary length;
- entry (that is, a K-V record) is stored by default in the lexicographic order of the key, of course, the developer can also override the sort function;
- Basic operation interface provided: Put(), Delete(), Get(), Batch();
- Support batch operations with atomic operations;
- You can create a snapshot of the data panorama and allow you to find the data in the snapshot;
- You can traverse data through a forward (or backward) iterator (iterators implicitly create a snapshot);
- Automatically use Snappy to compress data;
- portability;

**limit**:

- Non-relational data model (NoSQL), does not support sql statements, and does not support indexes;
- Allow only one process to access a specific database at a time;
- There is no built-in C/S architecture, but developers can use the LevelDB library to package a server themselves;


The directory where the source code is located is in the ethereum/ethdb directory. The code is relatively simple, divided into the following three files

- database.go levelDB package code
- memory_database.go Memory-based database for testing, not persisted to file, only for testing
- interface.go defines the interface to the database
- database_test.go test case

## interface.go
Look at the following code, basically defines the basic operations of the KeyValue database, Put, Get, Has, Delete and other basic operations, levelDB does not support SQL, basically can be understood as the Map inside the data structure.

Package ethdb
Const IdealBatchSize = 100 * 1024

// Putter wraps the database write operation supported by both batches and regular databases.
//Putter interface defines the write interface for bulk operations and normal operations.
Type Putter interface {
Put(key []byte, value []byte) error
}

// Database wraps all database operations. All methods are safe for concurrent use.
/ / Database interface defines all database operations, all methods are multi-thread safe.
Type Database interface {
Putter
Get(key []byte) ([]byte, error)
Has(key []byte) (bool, error)
Delete(key []byte) error
Close()
NewBatch() Batch
}

// Batch is a write-only database that commits changes to its host database
// when Write is called. Batch cannot be used concurrently.
/ / Batch operation interface, can not be used simultaneously by multiple threads, when the Write method is called, the database will commit the changes written.
Type Batch interface {
Putter
ValueSize() int // amount of data in the batch
Write() error
}

## memory_database.go
This is basically a Map structure that encapsulates a memory. Then a lock is used to protect the resources of multiple threads.

Type MemDatabase struct {
Db map[string][]byte
Lock sync.RWMutex
}

Func NewMemDatabase() (*MemDatabase, error) {
Return &amp;MemDatabase{
Db: make(map[string][]byte),
}, nil
}

Func (db *MemDatabase) Put(key []byte, value []byte) error {
db.lock.Lock()
Defer db.lock.Unlock()
Db.db[string(key)] = common.CopyBytes(value)
Return nil
}
Func (db *MemDatabase) Has(key []byte) (bool, error) {
db.lock.RLock()
Defer db.lock.RUnlock()

_, ok := db.db[string(key)]
Return ok, nil
}

Then there is the operation of Batch. It is also relatively simple, and you can understand it at a glance.


Type kv struct{ k, v []byte }
Type memBatch struct {
Db *MemDatabase
Writes []kv
Size int
}
Func (b *memBatch) Put(key, value []byte) error {
B.writes = append(b.writes, kv{common.CopyBytes(key), common.CopyBytes(value)})
B.size += len(value)
Return nil
}
Func (b *memBatch) Write() error {
b.db.lock.Lock()
Defer b.db.lock.Unlock()

For _, kv := range b.writes {
B.db.db[string(kv.k)] = kv.v
}
Return nil
}


## database.go
This is the code used by the actual ethereum client, which encapsulates the levelDB interface.


Import (
"strconv"
"strings"
"sync"
"time"

"github.com/ethereum/go-ethereum/log"
"github.com/ethereum/go-ethereum/metrics"
"github.com/syndtr/goleveldb/leveldb"
"github.com/syndtr/goleveldb/leveldb/errors"
"github.com/syndtr/goleveldb/leveldb/filter"
"github.com/syndtr/goleveldb/leveldb/iterator"
"github.com/syndtr/goleveldb/leveldb/opt"
Gometrics "github.com/rcrowley/go-metrics"
)

The leveldb package of github.com/syndtr/goleveldb/leveldb is used, so some of the documentation used can be found there. It can be seen that the data structure mainly adds a lot of Mertrics to record the database usage, and adds some situations that quitChan uses to handle the stop time, which will be analyzed later. If the following code may be in doubt, it should be filtered again: filter.NewBloomFilter(10) This can be temporarily ignored. This is an option for performance optimization in levelDB, you can ignore it.


Type LDBDatabase struct {
Fn string // filename for reporting
Db *leveldb.DB // LevelDB instance

getTimer gometrics.Timer // Timer for measuring the database get request counts and latencies
putTimer gometrics.Timer // Timer for measuring the database put request counts and latencies
...metrics

quitLock sync.Mutex // Mutex protecting the quit channel access
quitChan chan chan error // Quit channel to stop the metrics collection before closing the database

Log log.Logger // Contextual logger tracking the database path
}

// NewLDBDatabase returns a LevelDB wrapped object.
Func NewLDBDatabase(file string, cache int, handles int) (*LDBDatabase, error) {
Logger := log.New("database", file)
// Ensure we have some minimal caching and file guarantees
If cache &lt; 16 {
Cache = 16
}
If handles &lt; 16 {
Handles = 16
}
logger.Info("Allocated cache and file handles", "cache", cache, "handles", handles)
// Open the db and recover any potential corruptions
Db, err := leveldb.OpenFile(file, &amp;opt.Options{
OpenFilesCacheCapacity: handles,
BlockCacheCapacity: cache / 2 * opt.MiB,
WriteBuffer: cache / 4 * opt.MiB, // Two of these are used internally
Filter: filter.NewBloomFilter(10),
})
If _, corrupted := err.(*errors.ErrCorrupted); corrupted {
Db, err = leveldb.RecoverFile(file, nil)
}
// (Re)check for errors and abort if opening of the db failed
If err != nil {
Return nil, err
}
Return &amp;LDBDatabase{
Fn: file,
Db: db,
Log: logger,
}, nil
}


Take a look at the Put and Has code below, because the code behind github.com/syndtr/goleveldb/leveldb is supported for multi-threaded access, so the following code is protected without using locks. Most of the code here is directly called leveldb package, so it is not detailed. One of the more interesting places is the Metrics code.

// Put puts the given key / value to the queue
Func (db *LDBDatabase) Put(key []byte, value []byte) error {
// Measure the database put latency, if requested
If db.putTimer != nil {
Defer db.putTimer.UpdateSince(time.Now())
}
// Generate the data to write to disk, update the meter and write
//value = rle.Compress(value)

If db.writeMeter != nil {
db.writeMeter.Mark(int64(len(value)))
}
Return db.db.Put(key, value, nil)
}

Func (db *LDBDatabase) Has(key []byte) (bool, error) {
Return db.db.Has(key, nil)
}

### Processing of Metrics
Previously, when I created NewLDBDatabase, I didn't initialize a lot of internal Mertrics. At this time, Mertrics is nil. Initializing Mertrics is in the Meter method. The external passed in a prefix parameter, and then created a variety of Mertrics (how to create Merter, will be analyzed later on the Meter topic), and then created quitChan. Finally, a thread is called to call the db.meter method.

// Meter configures the database metrics collectors and
Func (db *LDBDatabase) Meter(prefix string) {
// Short circuit metering if the metrics system is disabled
If !metrics.Enabled {
Return
}
// Initialize all the metrics collector at the requested prefix
db.getTimer = metrics.NewTimer(prefix + "user/gets")
db.putTimer = metrics.NewTimer(prefix + "user/puts")
db.delTimer = metrics.NewTimer(prefix + "user/dels")
db.missMeter = metrics.NewMeter(prefix + "user/misses")
db.readMeter = metrics.NewMeter(prefix + "user/reads")
db.writeMeter = metrics.NewMeter(prefix + "user/writes")
db.compTimeMeter = metrics.NewMeter(prefix + "compact/time")
db.compReadMeter = metrics.NewMeter(prefix + "compact/input")
db.compWriteMeter = metrics.NewMeter(prefix + "compact/output")

// Create a quit channel for the periodic collector and run it
db.quitLock.Lock()
db.quitChan = make(chan chan error)
db.quitLock.Unlock()

Go db.meter(3 * time.Second)
}

This method gets the internal counters of leveldb every 3 seconds and then publishes them to the metrics subsystem. This is an infinite loop method until quitChan receives an exit signal.

// meter periodically retrieves internal leveldb counters and reports them to
// the metrics subsystem.
// This is how a stats table look like (currently):
//The following comment is the string we returned by calling db.db.GetProperty("leveldb.stats"). Subsequent code needs to parse the string and write the information to the Meter.

// Compactions
// Level | Tables | Size(MB) | Time(sec) | Read(MB) | Write(MB)
// -------+------------+---------------+----------- ----+---------------+---------------
// 0 | 0 | 0.00000 | 1.27969 | 0.00000 | 12.31098
// 1 | 85 | 109.27913 | 28.09293 | 213.92493 | ​​214.26294
// 2 | 523 | 1000.37159 | 7.26059 | 66.86342 | 66.77884
// 3 | 570 | 1113.18458 | 0.00000 | 0.00000 | 0.00000

Func (db *LDBDatabase) meter(refresh time.Duration) {
// Create the counters to store current and previous values
Counters := make([][]float64, 2)
For i := 0; i &lt; 2; i++ {
Counters[i] = make([]float64, 3)
}
// Iterate ad infinitum and collect the stats
For i := 1; ; i++ {
// Retrieve the database stats
Stats, err := db.db.GetProperty("leveldb.stats")
If err != nil {
db.log.Error("Failed to read database stats", "err", err)
Return
}
// Find the compaction table, skip the header
Lines := strings.Split(stats, "\n")
For len(lines) &gt; 0 &amp;&amp; strings.TrimSpace(lines[0]) != "Compactions" {
Lines = lines[1:]
}
If len(lines) &lt;= 3 {
db.log.Error("Compaction table not found")
Return
}
Lines = lines[3:]

// Iterate over all the table rows, and accumulate the entries
For j := 0; j &lt; len(counters[i%2]); j++ {
Counters[i%2][j] = 0
}
For _, line := range lines {
Parts := strings.Split(line, "|")
If len(parts) != 6 {
Break
}
For idx, counter := range parts[3:] {
Value, err := strconv.ParseFloat(strings.TrimSpace(counter), 64)
If err != nil {
db.log.Error("Compaction entry parsing failed", "err", err)
Return
}
Counters[i%2][idx] += value
}
}
// Update all the requested meters
If db.compTimeMeter != nil {
db.compTimeMeter.Mark(int64((counters[i%2][0] - counters[(i-1)%2][0]) * 1000 * 1000 * 1000))
}
If db.compReadMeter != nil {
db.compReadMeter.Mark(int64((counters[i%2][1] - counters[(i-1)%2][1]) * 1024 * 1024))
}
If db.compWriteMeter != nil {
db.compWriteMeter.Mark(int64((counters[i%2][2] - counters[(i-1)%2][2]) * 1024 * 1024))
}
// Sleep a bit, then repeat the stats collection
Select {
Case errc := &lt;-db.quitChan:
// Quit requesting, stop hammering the database
Errc &lt;- nil
Return

Case &lt;-time.After(refresh):
// Timeout, gather a new set of stats
}
}
}

</pre></body></html>