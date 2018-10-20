All data of go-ethereum is stored in levelDB, the open source KeyValue file database of Google. All the data of the entire blockchain is stored in a levelDB database. LevelDB supports the function of splitting files according to file size, so we see The data of the blockchain is a small file. In fact, these small files are all the same levelDB instance. Here is a simple look at the levelDB's go package code.

LevelDB official website introduction features

**Features**：

- Both key and value are byte arrays of arbitrary length;
- The entry (that is, a KV record) is stored by default in the lexicographic order of the key. Of course, the developer can also override the sort function.
- Basic operation interface provided: Put(), Delete(), Get(), Batch();
- Support batch operations with atomic operations;
- You can create a snapshot of the data panorama and allow you to find the data in the snapshot;
- You can traverse the data through a forward (or backward) iterator (the iterator implicitly creates a snapshot);
- Automatically use Snappy to compress data;
- portability;

**Limitations**：

- Non-relational data model (NoSQL), does not support sql statement, does not support indexing;
- Allow only one process to access a specific database at a time;
- There is no built-in Client/Server architecture, but developers can use the LevelDB library to package a server themselves;

The directory where the source code is located is in the ethereum/ethdb directory. The code is relatively simple, divided into the following three files

- database.go levelDB package
- memory_database.go Memory-based database for testing, not persisted as a file, only for testing
- interface.go defines the interface of the database
- database_test.go test case for golang

## interface.go

Look at the following code, basically defines the basic operations of the KeyValue database, Put, Get, Has, Delete and other basic operations, levelDB does not support SQL, basically can be understood as the Map inside the data structure.

```go
package ethdb
const IdealBatchSize = 100 * 1024

// Putter wraps the database write operation supported by both batches and regular databases.
type Putter interface {
	Put(key []byte, value []byte) error
}

// Database wraps all database operations. All methods are safe for concurrent use.
type Database interface {
	Putter
	Get(key []byte) ([]byte, error)
	Has(key []byte) (bool, error)
	Delete(key []byte) error
	Close()
	NewBatch() Batch
}

// Batch is a write-only database that commits changes to its host database
// when Write is called. Batch cannot be used concurrently.
type Batch interface {
	Putter
	ValueSize() int // amount of data in the batch
	Write() error
}
```

## memory_database.go

This is basically a Map structure that encapsulates a memory. Then a lock is used to protect the resources of multiple threads.

```go
type MemDatabase struct {
	db   map[string][]byte
	lock sync.RWMutex
}

func NewMemDatabase() (*MemDatabase, error) {
	return &MemDatabase{
		db: make(map[string][]byte),
	}, nil
}

func (db *MemDatabase) Put(key []byte, value []byte) error {
	db.lock.Lock()
	defer db.lock.Unlock()
	db.db[string(key)] = common.CopyBytes(value)
	return nil
}
func (db *MemDatabase) Has(key []byte) (bool, error) {
	db.lock.RLock()
	defer db.lock.RUnlock()

	_, ok := db.db[string(key)]
	return ok, nil
}
```

Then there is the operation of Batch. It is also relatively simple, and you can understand it at a glance.

```go
type kv struct{ k, v []byte }
type memBatch struct {
	db     *MemDatabase
	writes []kv
	size   int
}
func (b *memBatch) Put(key, value []byte) error {
	b.writes = append(b.writes, kv{common.CopyBytes(key), common.CopyBytes(value)})
	b.size += len(value)
	return nil
}
func (b *memBatch) Write() error {
	b.db.lock.Lock()
	defer b.db.lock.Unlock()

	for _, kv := range b.writes {
		b.db.db[string(kv.k)] = kv.v
	}
	return nil
}
```

## database.go

This is the code used by the actual ethereum client, which encapsulates the levelDB interface.

```go
import (
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
	gometrics "github.com/rcrowley/go-metrics"
)
```

The leveldb package of github.com/syndtr/goleveldb/leveldb is used, so some of the documentation used can be found there. It can be seen that the data structure mainly adds a lot of Mertrics to record the database usage, and adds some situations that quitChan uses to handle the stop time, which will be analyzed later. If the following code may be in doubt, it should be filtered again: filter.NewBloomFilter(10) This can be temporarily ignored. This is an option for performance optimization in levelDB, you can ignore it.

```go
type LDBDatabase struct {
	fn string      // filename for reporting
	db *leveldb.DB // LevelDB instance

	getTimer       gometrics.Timer // Timer for measuring the database get request counts and latencies
	putTimer       gometrics.Timer // Timer for measuring the database put request counts and latencies
	...metrics

	quitLock sync.Mutex      // Mutex protecting the quit channel access
	quitChan chan chan error // Quit channel to stop the metrics collection before closing the database

	log log.Logger // Contextual logger tracking the database path
}

// NewLDBDatabase returns a LevelDB wrapped object.
func NewLDBDatabase(file string, cache int, handles int) (*LDBDatabase, error) {
	logger := log.New("database", file)
	// Ensure we have some minimal caching and file guarantees
	if cache < 16 {
		cache = 16
	}
	if handles < 16 {
		handles = 16
	}
	logger.Info("Allocated cache and file handles", "cache", cache, "handles", handles)
	// Open the db and recover any potential corruptions
	db, err := leveldb.OpenFile(file, &opt.Options{
		OpenFilesCacheCapacity: handles,
		BlockCacheCapacity:     cache / 2 * opt.MiB,
		WriteBuffer:            cache / 4 * opt.MiB, // Two of these are used internally
		Filter:                 filter.NewBloomFilter(10),
	})
	if _, corrupted := err.(*errors.ErrCorrupted); corrupted {
		db, err = leveldb.RecoverFile(file, nil)
	}
	// (Re)check for errors and abort if opening of the db failed
	if err != nil {
		return nil, err
	}
	return &LDBDatabase{
		fn:  file,
		db:  db,
		log: logger,
	}, nil
}
```

Take a look at the Put and Has code below, because the code behind github.com/syndtr/goleveldb/leveldb is supported for multi-threaded access, so the following code is protected without using locks (because it implemented mutex lock internally). Most of the code here is directly called leveldb package, so it is not detailed. One of the more interesting places is the Metrics code.

```go
// Put puts the given key / value to the queue
func (db *LDBDatabase) Put(key []byte, value []byte) error {
	// Measure the database put latency, if requested
	if db.putTimer != nil {
		defer db.putTimer.UpdateSince(time.Now())
	}
	// Generate the data to write to disk, update the meter and write
	//value = rle.Compress(value)

	if db.writeMeter != nil {
		db.writeMeter.Mark(int64(len(value)))
	}
	return db.db.Put(key, value, nil)
}

func (db *LDBDatabase) Has(key []byte) (bool, error) {
	return db.db.Has(key, nil)
}
```

### Metrics processing

Previously, when I created NewLDBDatabase, I didn't initialize a lot of internal Mertrics. At this time, Mertrics is nil. Initializing Mertrics is in the Meter method. The external passed in a prefix parameter, and then created a variety of Mertrics (how to create Merter, will be analyzed later on the Meter topic), and then created quitChan. Finally, a thread is called to call the db.meter method.

```go
// Meter configures the database metrics collectors and
func (db *LDBDatabase) Meter(prefix string) {
	// Short circuit metering if the metrics system is disabled
	if !metrics.Enabled {
		return
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

	go db.meter(3 * time.Second)
}
```

This method gets the internal counters of leveldb every 3 seconds and then publishes them to the metrics subsystem. This is an infinite loop method until quitChan receives an exit signal.

```go
// meter periodically retrieves internal leveldb counters and reports them to
// the metrics subsystem.
// This is how a stats table look like (currently):
// The following comment is the string we call db.db.GetProperty("leveldb.stats"), and the subsequent code needs to parse the string and write the information to the Meter.

//   Compactions
//    Level |   Tables   |    Size(MB)   |    Time(sec)  |    Read(MB)   |   Write(MB)
//   -------+------------+---------------+---------------+---------------+---------------
//      0   |          0 |       0.00000 |       1.27969 |       0.00000 |      12.31098
//      1   |         85 |     109.27913 |      28.09293 |     213.92493 |     214.26294
//      2   |        523 |    1000.37159 |       7.26059 |      66.86342 |      66.77884
//      3   |        570 |    1113.18458 |       0.00000 |       0.00000 |       0.00000

func (db *LDBDatabase) meter(refresh time.Duration) {
	// Create the counters to store current and previous values
	counters := make([][]float64, 2)
	for i := 0; i < 2; i++ {
		counters[i] = make([]float64, 3)
	}
	// Iterate ad infinitum and collect the stats
	for i := 1; ; i++ {
		// Retrieve the database stats
		stats, err := db.db.GetProperty("leveldb.stats")
		if err != nil {
			db.log.Error("Failed to read database stats", "err", err)
			return
		}
		// Find the compaction table, skip the header
		lines := strings.Split(stats, "\n")
		for len(lines) > 0 && strings.TrimSpace(lines[0]) != "Compactions" {
			lines = lines[1:]
		}
		if len(lines) <= 3 {
			db.log.Error("Compaction table not found")
			return
		}
		lines = lines[3:]

		// Iterate over all the table rows, and accumulate the entries
		for j := 0; j < len(counters[i%2]); j++ {
			counters[i%2][j] = 0
		}
		for _, line := range lines {
			parts := strings.Split(line, "|")
			if len(parts) != 6 {
				break
			}
			for idx, counter := range parts[3:] {
				value, err := strconv.ParseFloat(strings.TrimSpace(counter), 64)
				if err != nil {
					db.log.Error("Compaction entry parsing failed", "err", err)
					return
				}
				counters[i%2][idx] += value
			}
		}
		// Update all the requested meters
		if db.compTimeMeter != nil {
			db.compTimeMeter.Mark(int64((counters[i%2][0] - counters[(i-1)%2][0]) * 1000 * 1000 * 1000))
		}
		if db.compReadMeter != nil {
			db.compReadMeter.Mark(int64((counters[i%2][1] - counters[(i-1)%2][1]) * 1024 * 1024))
		}
		if db.compWriteMeter != nil {
			db.compWriteMeter.Mark(int64((counters[i%2][2] - counters[(i-1)%2][2]) * 1024 * 1024))
		}
		// Sleep a bit, then repeat the stats collection
		select {
		case errc := <-db.quitChan:
			// Quit requesting, stop hammering the database
			errc <- nil
			return

		case <-time.After(refresh):
			// Timeout, gather a new set of stats
		}
	}
}
```
