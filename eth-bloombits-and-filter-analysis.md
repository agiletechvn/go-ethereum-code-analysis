## Ethereum's Bloom filter

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

For any input value, first ask his KEC output, and then take the value of [0,1][2,3], [4,5] of the KEC output to modulo 2048, and get three values. These three values ​​are the subscripts that need to be set in the output 2048. That is to say, for any input, if the value of its corresponding three subscripts is not 1, then it is definitely not in this block. If, if the corresponding three bits are all 1, it cannot be stated that it is in this block. This is the characteristic of the Bloom filter.

The Bloom filter in the receipt is the union of the Bloom filter outputs for all logs.

At the same time, the logBloom in the block header is the union of the Bloom filters of all the receipts.

## ChainIndexer with BloomIndexer

I first saw ChainIndexer, I didn't really understand what it is. In fact, as you can see from the name, it is the index of Chain. In ethereum we have seen the BloomIndexer, which is the index of the Bloom filter.

The ability to find the specified Log is provided in our protocol.

The user can find the specified Log, the starting block number, and the ending block number by passing the following parameters, filtering according to the address specified by the contract Addresses, and filtering according to the specified Topics.

```go
// FilterCriteria represents a request to create a new filter.
type FilterCriteria struct {
	FromBlock *big.Int
	ToBlock   *big.Int
	Addresses []common.Address
	Topics    [][]common.Hash
}
```

If the interval between the start and end is large, it is inefficient to directly retrieve the logBloom area of ​​each block header in turn. Because each block header is stored separately, it may require a lot of random access to the disk.

So the Ethereum protocol maintains a set of indexes locally to speed up the process.

The general principle is. Each 4096 block is called a Section, and the logBloom in a Section is stored together. For each Section, use a two-dimensional data, A[2048][4096]. The first dimension 2048 represents the length of the bloom filter of 2048 bytes. The second dimension 4096 represents all the blocks in a Section, and each location represents one of the blocks in order.

- A[0][0]=blockchain[section*4096+0].logBloom[0],
- A[0][1]=blockchain[section*4096+1].logBloom[0],
- A[0][4096]=blockchain[(section+1)*4096].logBloom[0],
- A[1][0]=blockchain[section*4096+0].logBloom[1],
- A[1][1024]=blockchain[section*4096+1024].logBloom[1],
- A[2047][1]=blockchain[section*4096+1].logBloom[2047],

If Section is filled, it will be written as 2048 KV.
![image](picture/bloom_6.png)

## bloombit.go code analysis

This code is relatively non-independent. If you look at this code alone, it's a bit confusing, because it only implements some interfaces. The specific processing logic is not here, but in the core. But here I will analyze the information I have mentioned before. Subsequent more detailed logic is analyzed in detail when analyzing the core code.

The service thread startBloomHandlers, this method is to respond to the specific query request, given the specified Section and bit to query from the levelDB and then return. Looking at it alone is a bit confusing. The call to this method is more complicated. It involves a lot of logic in the core. I will not elaborate here first. Until there is this method.

```go
type Retrieval struct {
	Bit      uint			//the value of bit 0-2047 represents the value you want to get
	Sections []uint64		// those Section
	Bitsets  [][]byte		// the result of the query
}
// startBloomHandlers starts a batch of goroutines to accept bloom bit database
// retrievals from possibly a range of filters and serving the data to satisfy.
func (eth *Ethereum) startBloomHandlers() {
	for i := 0; i < bloomServiceThreads; i++ {
		go func() {
			for {
				select {
				case <-eth.shutdownChan:
					return

					// request channel
				case request := <-eth.bloomRequests:
					// get task from the channel
						task := <-request

					task.Bitsets = make([][]byte, len(task.Sections))
					for i, section := range task.Sections {
						head := core.GetCanonicalHash(eth.chainDb, (section+1)*params.BloomBitsBlocks-1)
						blob, err := bitutil.DecompressBytes(core.GetBloomBits(eth.chainDb, task.Bit, section, head), int(params.BloomBitsBlocks)/8)
						if err != nil {
							panic(err)
						}
						task.Bitsets[i] = blob
						}
						// return result via request channel
					request <- task
				}
			}
		}()
	}
}
```

### Data structure

The process of building the index for the main user of the BloomIndexer object is an interface implementation of core.ChainIndexer, so only some necessary interfaces are implemented. The logic for creating an index is also in core.ChainIndexer.

```go
// BloomIndexer implements a core.ChainIndexer, building up a rotated bloom bits index
// for the Ethereum header bloom filters, permitting blazing fast filtering.
type BloomIndexer struct {
	size uint64 // section size to generate bloombits for

	db  ethdb.Database       // database instance to write index data and metadata into
	gen *bloombits.Generator // generator to rotate the bloom bits crating the bloom index

	section uint64      // Section is the section number being processed currently  section
	head    common.Hash // Head is the hash of the last header processed
}

// NewBloomIndexer returns a chain indexer that generates bloom bits data for the
// canonical chain for fast logs filtering.
func NewBloomIndexer(db ethdb.Database, size uint64) *core.ChainIndexer {
	backend := &BloomIndexer{
		db:   db,
		size: size,
	}
	table := ethdb.NewTable(db, string(core.BloomBitsIndexPrefix))

	return core.NewChainIndexer(db, table, backend, size, bloomConfirms, bloomThrottling, "bloombits")
}
```

Reset implements the ChainIndexerBackend method and starts a new section.

```go
// Reset implements core.ChainIndexerBackend, starting a new bloombits index
// section.
func (b *BloomIndexer) Reset(section uint64) {
	gen, err := bloombits.NewGenerator(uint(b.size))
	if err != nil {
		panic(err)
	}
	b.gen, b.section, b.head = gen, section, common.Hash{}
}
```

Process implements ChainIndexerBackend, adding a new block header to index

```go
// Process implements core.ChainIndexerBackend, adding a new header's bloom into
// the index.
func (b *BloomIndexer) Process(header *types.Header) {
	b.gen.AddBloom(uint(header.Number.Uint64()-b.section*b.size), header.Bloom)
	b.head = header.Hash()
}
```

The Commit method implements ChainIndexerBackend, persists and writes to the database.

```go
// Commit implements core.ChainIndexerBackend, finalizing the bloom section and
// writing it out into the database.
func(b *BloomIndexer) Commit() error {
    batch: = b.db.NewBatch()
    for i: = 0;i < types.BloomBitLength;i++{
        bits, err: = b.gen.Bitset(uint(i))
        if err != nil {
            return err
        }
        core.WriteBloomBits(batch, uint(i), b.section, b.head, bitutil.CompressBytes(bits))
    }
    return batch.Write()
}
```

## filter/api.go source code analysis

The eth/filter package contains the ability to provide filtering to the user. The user can filter the transaction or block by calling and then continue to get the result. If there is no operation for 5 minutes, the filter will be deleted.

The structure of the filter.

```go
var (
	deadline = 5 * time.Minute // consider a filter inactive if it has not been polled for within deadline
)

// filter is a helper struct that holds meta information over the filter type
// and associated subscription in the event system.
type filter struct {
	typ      Type			// type of filter
	deadline *time.Timer // filter is inactiv when deadline triggers, the timer is triggered
	hashes   []common.Hash //filtered hash results
	crit     FilterCriteria	//filter condition
	logs     []*types.Log    //log information
	s        *Subscription // associated subscription in event system, the subscriber in the event system
}
```

Construction method

```go
// PublicFilterAPI offers support to create and manage filters. This will allow external clients to retrieve various
// information related to the Ethereum protocol such als blocks, transactions and logs.
// PublicFilterAPI - used to create and manage filters, allow external clients to get some information like block, transaction and log information.
type PublicFilterAPI struct {
	backend   Backend
	mux       *event.TypeMux
	quit      chan struct{}
	chainDb   ethdb.Database
	events    *EventSystem
	filtersMu sync.Mutex
	filters   map[rpc.ID]*filter
}

// NewPublicFilterAPI returns a new PublicFilterAPI instance.
func NewPublicFilterAPI(backend Backend, lightMode bool) *PublicFilterAPI {
	api := &PublicFilterAPI{
		backend: backend,
		mux:     backend.EventMux(),
		chainDb: backend.ChainDb(),
		events:  NewEventSystem(backend.EventMux(), backend, lightMode),
		filters: make(map[rpc.ID]*filter),
	}
	go api.timeoutLoop()

	return api
}
```

### Timeout check

```go
// timeoutLoop runs every 5 minutes and deletes filters that have not been recently used.
// Tt is started when the api is created.
//  Check every 5 minutes. If the filter expires, delete it.
func (api *PublicFilterAPI) timeoutLoop() {
  ticker := time.NewTicker(5 * time.Minute)
  for {
    <-ticker.C
    api.filtersMu.Lock()
    for id, f := range api.filters {
      select {
      case <-f.deadline.C:
        f.s.Unsubscribe()
        delete(api.filters, id)
      default:
        continue
      }
    }
    api.filtersMu.Unlock()
  }
}
```

NewPendingTransactionFilter to create a PendingTransactionFilter. This method is used for channels that cannot create long connections (such as HTTP). If a channel that can establish long links (such as WebSocket) can be processed using the send subscription mode provided by rpc, there is no need for a continuous round. Inquire

```go
// NewPendingTransactionFilter creates a filter that fetches pending transaction hashes
// as transactions enter the pending state.
//
// It is part of the filter package because this filter can be used throug the
// `eth_getFilterChanges` polling method that is also used for log filters.
//
// https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_newpendingtransactionfilter
func(api\ * PublicFilterAPI) NewPendingTransactionFilter() rpc.ID {
    var (
        pendingTxs = make(chan common.Hash)
        // Subscribe to this message in the event system
        pendingTxSub = api.events.SubscribePendingTxEvents(pendingTxs)
    )
    api.filtersMu.Lock()
    api.filters[pendingTxSub.ID] = & filter {
        typ: PendingTransactionsSubscription,
        deadline: time.NewTimer(deadline),
        hashes: make([] common.Hash, 0),
        s: pendingTxSub
    }
    api.filtersMu.Unlock()
    go func() {
        for {
            select {
                case ph:
                    = < -pendingTxs: // received pendingTxs，stored in the filter hashes
                        api.filtersMu.Lock()
                    if f, found: = api.filters[pendingTxSub.ID];
                    found {
                        f.hashes = append(f.hashes, ph)
                    }
                    api.filtersMu.Unlock()
                case <-pendingTxSub.Err():
                    api.filtersMu.Lock()
                    delete(api.filters, pendingTxSub.ID)
                    api.filtersMu.Unlock()
                    return
            }
        }
    }()
    return pendingTxSub.ID
}
```

Polling: GetFilterChanges

```go
// GetFilterChanges returns the logs for the filter with the given id since
// last time it was called. This can be used for polling.
// For pending transaction and block filters the result is []common.Hash.
// (pending)Log filters return []Log.
// https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_getfilterchanges
func(api\ * PublicFilterAPI) GetFilterChanges(id rpc.ID)(interface {}, error) {
    api.filtersMu.Lock()
    defer api.filtersMu.Unlock()
    if f, found: = api.filters[id];
    found {
        if !f.deadline.Stop() { // If the timer has been triggered, but the filter has not been removed, then we first receive the value of the timer and then reset the timer.
            // timer expired but filter is not yet removed in timeout loop
            // receive timer value and reset timer
            < -f.deadline.C
        }
        f.deadline.Reset(deadline)
        switch f.typ {
            case PendingTransactionsSubscription, BlocksSubscription:
                hashes: = f.hashes
                f.hashes = nil
                return returnHashes(hashes), nil
            case LogsSubscription:
                logs: = f.logs
                f.logs = nil
                return returnLogs(logs), nil
        }
    }
    return [] interface {} {}, fmt.Errorf("filter not found")
}
```

For a channel that can establish a long connection, you can directly use the rpc send subscription mode, so that the client can directly receive the filtering information without calling the polling method. You can see that this mode is not added to the filters container, and there is no timeout management. In other words, two modes are supported.

```go
// NewPendingTransactions creates a subscription that is triggered each time a transaction
// enters the transaction pool and was signed from one of the transactions this nodes manages.
func(api * PublicFilterAPI) NewPendingTransactions(ctx context.Context)( * rpc.Subscription, error) {
		notifier, supported: = rpc.NotifierFromContext(ctx)
		if !supported {
				return &rpc.Subscription {}, rpc.ErrNotificationsUnsupported
		}

		rpcSub: = notifier.CreateSubscription()

		go func() {
				txHashes: = make(chan common.Hash)
				pendingTxSub: = api.events.SubscribePendingTxEvents(txHashes)

				for {
						select {
								case h:
										= < -txHashes:
												notifier.Notify(rpcSub.ID, h)
								case <-rpcSub.Err():
										pendingTxSub.Unsubscribe()
										return
								case <-notifier.Closed():
										pendingTxSub.Unsubscribe()
										return
						}
				}
		}()

		return rpcSub, nil
}
```

The log filtering function filters the logs according to the parameters specified by FilterCriteria, starts the block, ends the block, addresses and Topics, and introduces a new object filter.

```go
// FilterCriteria represents a request to create a new filter.
type FilterCriteria struct {
		FromBlock * big.Int
		ToBlock * big.Int
		Addresses[] common.Address
		Topics[][] common.Hash
}
// GetLogs returns logs matching the given argument that are stored within the state.
//
// https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_getlogs
func(api * PublicFilterAPI) GetLogs(ctx context.Context, crit FilterCriteria)([] * types.Log, error) {
		// Convert the RPC block numbers into internal representations
		if crit.FromBlock == nil {
				crit.FromBlock = big.NewInt(rpc.LatestBlockNumber.Int64())
		}
		if crit.ToBlock == nil {
						crit.ToBlock = big.NewInt(rpc.LatestBlockNumber.Int64())
				}
				// Create and run the filter to get all the logs
		filter: = New(api.backend, crit.FromBlock.Int64(), crit.ToBlock.Int64(), crit.Addresses, crit.Topics)
		logs, err: = filter.Logs(ctx)
		if err != nil {
				return nil, err
		}
		return returnLogs(logs), err
}
```

## filter.go

There is a Filter object defined in fiter.go. This object is mainly used to perform log filtering based on the block's BloomIndexer and Bloom filter.

### Data structure

```go
type Backend interface {
	ChainDb() ethdb.Database
	EventMux() *event.TypeMux
	HeaderByNumber(ctx context.Context, blockNr rpc.BlockNumber) (*types.Header, error)
	GetReceipts(ctx context.Context, blockHash common.Hash) (types.Receipts, error)

	SubscribeTxPreEvent(chan<- core.TxPreEvent) event.Subscription
	SubscribeChainEvent(ch chan<- core.ChainEvent) event.Subscription
	SubscribeRemovedLogsEvent(ch chan<- core.RemovedLogsEvent) event.Subscription
	SubscribeLogsEvent(ch chan<- []*types.Log) event.Subscription

	BloomStatus() (uint64, uint64)
	ServiceFilter(ctx context.Context, session *bloombits.MatcherSession)
}

// Filter can be used to retrieve and filter logs.
type Filter struct {
	backend Backend				// backend

	db         ethdb.Database	// database
	begin, end int64			// Start, ending block
	addresses  []common.Address	// Filter address
	topics     [][]common.Hash	// Filter topic

	matcher *bloombits.Matcher	// Bloom filter matcher
}
```

The constructor adds both address and topic to the filters container. Then build a bloombits.NewMatcher(size, filters). This function is implemented in the core and will not be explained for the time being.

```go
// New creates a new filter which uses a bloom filter on blocks to figure out whether
// a particular block is interesting or not.
func New(backend Backend, begin, end int64, addresses []common.Address, topics [][]common.Hash) *Filter {
	// Flatten the address and topic filter clauses into a single bloombits filter
	// system. Since the bloombits are not positional, nil topics are permitted,
	// which get flattened into a nil byte slice.
	var filters [][][]byte
	if len(addresses) > 0 {
		filter := make([][]byte, len(addresses))
		for i, address := range addresses {
			filter[i] = address.Bytes()
		}
		filters = append(filters, filter)
	}
	for _, topicList := range topics {
		filter := make([][]byte, len(topicList))
		for i, topic := range topicList {
			filter[i] = topic.Bytes()
		}
		filters = append(filters, filter)
	}
	// Assemble and return the filter
	size, _ := backend.BloomStatus()

	return &Filter{
		backend:   backend,
		begin:     begin,
		end:       end,
		addresses: addresses,
		topics:    topics,
		db:        backend.ChainDb(),
		matcher:   bloombits.NewMatcher(size, filters),
	}
}
```

Logs performs filtering

```go
// Logs searches the blockchain for matching log entries, returning all from the
// first block that contains matches, updating the start of the filter accordingly.
func (f *Filter) Logs(ctx context.Context) ([]*types.Log, error) {
	// Figure out the limits of the filter range
	header, _ := f.backend.HeaderByNumber(ctx, rpc.LatestBlockNumber)
	if header == nil {
		return nil, nil
	}
	head := header.Number.Uint64()

	if f.begin == -1 {
		f.begin = int64(head)
	}
	end := uint64(f.end)
	if f.end == -1 {
		end = head
	}
	// Gather all indexed logs, and finish with non indexed ones
	var (
		logs []*types.Log
		err  error
	)
	size, sections := f.backend.BloomStatus()
	// indexed is the maximum value of the block in which the index was created.
	// the perform an index search
	if indexed := sections * size; indexed > uint64(f.begin) {
		if indexed > end {
			logs, err = f.indexedLogs(ctx, end)
		} else {
			logs, err = f.indexedLogs(ctx, indexed-1)
		}
		if err != nil {
			return logs, err
		}
	}
	// Perform a non-indexed search
	rest, err := f.unindexedLogs(ctx, end)
	logs = append(logs, rest...)
	return logs, err
}
```

Index search

```go
// indexedLogs returns the logs matching the filter criteria based on the bloom
// bits indexed available locally or via the network.
func (f *Filter) indexedLogs(ctx context.Context, end uint64) ([]*types.Log, error) {
	// Create a matcher session and request servicing from the backend
	matches := make(chan uint64, 64)
	// Start matcher
	session, err := f.matcher.Start(uint64(f.begin), end, matches)
	if err != nil {
		return nil, err
	}
	defer session.Close(time.Second)
	// Perform filtering services. These are all in the core.

	f.backend.ServiceFilter(ctx, session)

	// Iterate over the matches until exhausted or context closed
	var logs []*types.Log

	for {
		select {
		case number, ok := <-matches:
			// Abort if all matches have been fulfilled
			if !ok {  // Did not receive the value and the channel has been closed
				f.begin = int64(end) + 1  //Update begin, for the following non-indexed search
				return logs, nil
			}
			// Retrieve the suggested block and pull any truly matching logs
			header, err := f.backend.HeaderByNumber(ctx, rpc.BlockNumber(number))
			if header == nil || err != nil {
				return logs, err
			}
			found, err := f.checkMatches(ctx, header) //find matching values
			if err != nil {
				return logs, err
			}
			logs = append(logs, found...)

		case <-ctx.Done():
			return logs, ctx.Err()
		}
	}
}
```

checkMatches, get all the receipts and get all the logs from the receipt. Execute the filterLogs method.

```go
// checkMatches checks if the receipts belonging to the given header contain any log events that
// match the filter criteria. This function is called when the bloom filter signals a potential match.
func(f * Filter) checkMatches(ctx context.Context, header * types.Header)(logs[] * types.Log, err error) {
    // Get the logs of the block
    receipts, err: = f.backend.GetReceipts(ctx, header.Hash())
    if err != nil {
        return nil, err
    }
    var unfiltered[] * types.Log
    for\ _, receipt: = range receipts {
        unfiltered = append(unfiltered, ([]\ * types.Log)(receipt.Logs)...)
    }
    logs = filterLogs(unfiltered, nil, nil, f.addresses, f.topics)
    if len(logs) > 0 {
        return logs, nil
    }
    return nil, nil
}
```

filterLogs, this method finds the matching in the given logs. And return.

```go
// filterLogs creates a slice of logs matching the given criteria.
func filterLogs(logs []*types.Log, fromBlock, toBlock *big.Int, addresses []common.Address, topics [][]common.Hash) []*types.Log {
	var ret []*types.Log
Logs:
	for _, log := range logs {
		if fromBlock != nil && fromBlock.Int64() >= 0 && fromBlock.Uint64() > log.BlockNumber {
			continue
		}
		if toBlock != nil && toBlock.Int64() >= 0 && toBlock.Uint64() < log.BlockNumber {
			continue
		}

		if len(addresses) > 0 && !includes(addresses, log.Address) {
			continue
		}
		// If the to filtered topics is greater than the amount of topics in logs, skip.
		if len(topics) > len(log.Topics) {
			continue Logs
		}
		for i, topics := range topics {
			match := len(topics) == 0 // empty rule set == wildcard
			for _, topic := range topics {
				if log.Topics[i] == topic {
					match = true
					break
				}
			}
			if !match {
				continue Logs
			}
		}
		ret = append(ret, log)
	}
	return ret
}
```

unindexedLogs, a non-indexed query that loops through all the blocks. First use the header.Bloom inside the block to see if it is possible. If it exists, use checkMatches to retrieve all matches.

```go
// indexedLogs returns the logs matching the filter criteria based on raw block
// iteration and bloom matching.
func(f * Filter) unindexedLogs(ctx context.Context, end uint64)([] * types.Log, error) {
    var logs[]\ * types.Log
    for;
    f.begin <= int64(end);
    f.begin++{
        header, err: = f.backend.HeaderByNumber(ctx, rpc.BlockNumber(f.begin))
        if header == nil || err != nil {
            return logs, err
        }
        if bloomFilter(header.Bloom, f.addresses, f.topics) {
            found, err: = f.checkMatches(ctx, header)
            if err != nil {
                return logs, err
            }
            logs = append(logs, found...)
        }
    }
    return logs, nil
}
```

## Sum up

The filter source package mainly implements two functions.

- Provides a filter RPC that publishes subscription patterns. Used to provide real-time filtering of transactions, blocks, logs, etc. to the rpc client.
- A log filtering mode based on bloomIndexer is provided. In this mode, Bloom filtering can be performed on a large number of blocks quickly. Filtering operations for historical logs are also provided.
