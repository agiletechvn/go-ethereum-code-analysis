The downloader is mainly responsible for the initial synchronization of the blockchain. The current synchronization has two modes. One is the traditional fullmode. This mode builds the blockchain by downloading the block header and the block body. The synchronization process is The normal block insertion process is the same, including block header verification, transaction verification, transaction execution, account status change, etc. This is actually a process that consumes CPU and disk. The other mode is the fast sync mode of fast sync, which has a special document to describe. Please refer to the documentation for fast sync. Simply put, the fast sync mode downloads the block header, block body and receipt. The insertion process does not execute the transaction, and then synchronizes all account statuses at a block height (the highest block height - 1024), followed by The 1024 blocks will be built using fullmode. This mode will increase the insertion time of the block without generating a large amount of historical account information. It will save disk relatively, but it will consume more network. Because you need to download the receipt and status.

## downloader data structure

```go
type Downloader struct {
	mode SyncMode       // Synchronisation mode defining the strategy used (per sync cycle)
	mux  *event.TypeMux // Event multiplexer to announce sync operation events
	// queue use to schedule block headers, transactions, receipts, then assembly them after download
	queue   *queue   // Scheduler for selecting the hashes to download
	// peer collection
	peers   *peerSet // Set of active peers from which download can proceed
	stateDB ethdb.Database
	// fast sync head of the Pivot point
	fsPivotLock  *types.Header // Pivot header on critical section entry (cannot change between retries)
	fsPivotFails uint32        // Number of subsequent fast sync failures in the critical section
	// download round trip delay
	rttEstimate   uint64 // Round trip time to target for download requests
	rttConfidence uint64 // Confidence in the estimated RTT (unit: millionths to allow atomic ops)  estimate the confidence of RTT - round trip delay time

	// Statistics
	syncStatsChainOrigin uint64 // Origin block number where syncing started at
	syncStatsChainHeight uint64 // Highest block number known when syncing started
	syncStatsState       stateSyncStats
	syncStatsLock        sync.RWMutex // Lock protecting the sync stats fields

	lightchain LightChain
	blockchain BlockChain

	// Callbacks
	dropPeer peerDropFn // Drops a peer for misbehaving

	// Status
	synchroniseMock func(id string, hash common.Hash) error // Replacement for synchronise during testing
	synchronising   int32
	notified        int32

	// Channels
	headerCh      chan dataPack        // [eth/62] Channel receiving inbound block headers  header
	bodyCh        chan dataPack        // [eth/62] Channel receiving inbound block bodies   bodies
	receiptCh     chan dataPack        // [eth/63] Channel receiving inbound receipts       receipts
	bodyWakeCh    chan bool            // [eth/62] Channel to signal the block body fetcher of new tasks
	receiptWakeCh chan bool            // [eth/63] Channel to signal the receipt fetcher of new tasks
	headerProcCh  chan []*types.Header // [eth/62] Channel to feed the header processor new tasks

	// for stateFetcher
	stateSyncStart chan *stateSync   // state fetcher
	trackStateReq  chan *stateReq	 // TODO
	stateCh        chan dataPack // [eth/63] Channel receiving inbound node state data	   state
	// Cancellation and termination
	cancelPeer string        // Identifier of the peer currently being used as the master (cancel on drop)
	cancelCh   chan struct{} // Channel to cancel mid-flight syncs
	cancelLock sync.RWMutex  // Lock to protect the cancel channel and peer in delivers

	quitCh   chan struct{} // Quit channel to signal termination
	quitLock sync.RWMutex  // Lock to prevent double closes

	// Testing hooks
	syncInitHook     func(uint64, uint64)  // Method to call upon initiating a new sync run
	bodyFetchHook    func([]*types.Header) // Method to call upon starting a block body fetch
	receiptFetchHook func([]*types.Header) // Method to call upon starting a receipt fetch
	chainInsertHook  func([]*fetchResult)  // Method to call upon inserting a chain of blocks (possibly in multiple invocations)
}
```

Construction method

```go
// New creates a new downloader to fetch hashes and blocks from remote peers.
func New(mode SyncMode, stateDb ethdb.Database, mux *event.TypeMux, chain BlockChain, lightchain LightChain, dropPeer peerDropFn) *Downloader {
	if lightchain == nil {
		lightchain = chain
	}
	dl := &Downloader{
		mode:           mode,
		stateDB:        stateDb,
		mux:            mux,
		queue:          newQueue(),
		peers:          newPeerSet(),
		rttEstimate:    uint64(rttMaxEstimate),
		rttConfidence:  uint64(1000000),
		blockchain:     chain,
		lightchain:     lightchain,
		dropPeer:       dropPeer,
		headerCh:       make(chan dataPack, 1),
		bodyCh:         make(chan dataPack, 1),
		receiptCh:      make(chan dataPack, 1),
		bodyWakeCh:     make(chan bool, 1),
		receiptWakeCh:  make(chan bool, 1),
		headerProcCh:   make(chan []*types.Header, 1),
		quitCh:         make(chan struct{}),
		stateCh:        make(chan dataPack),
		stateSyncStart: make(chan *stateSync),
		trackStateReq:  make(chan *stateReq),
	}
	go dl.qosTuner()  // Simple mainly used to calculate rttEstimate and rttConfidence
	go dl.stateFetcher() // Start the task listener of stateFetcher, but this time there is no task to generate state fetcher.

	return dl
}
```

## Synchronous download

Synchronise: tries to synchronize with a peer. If there are some errors during the synchronization, Peer will be deleted. Then It will be tried again.

```go
// Synchronise tries to sync up our local block chain with a remote peer, both
// adding various sanity checks as well as wrapping it with various log entries.
func (d *Downloader) Synchronise(id string, head common.Hash, td *big.Int, mode SyncMode) error {
	err := d.synchronise(id, head, td, mode)
	switch err {
	case nil:
	case errBusy:

	case errTimeout, errBadPeer, errStallingPeer,
		errEmptyHeaderSet, errPeersUnavailable, errTooOld,
		errInvalidAncestor, errInvalidChain:
		log.Warn("Synchronisation failed, dropping peer", "peer", id, "err", err)
		d.dropPeer(id)

	default:
		log.Warn("Synchronisation failed, retrying", "err", err)
	}
	return err
}
```

synchronise

```go
// synchronise will select the peer and use it for synchronising. If an empty string is given
// it will use the best peer possible and synchronize if it's TD is higher than our own. If any of the
// checks fail an error will be returned. This method is synchronous
func (d *Downloader) synchronise(id string, hash common.Hash, td *big.Int, mode SyncMode) error {
	// Mock out the synchronisation if testing
	if d.synchroniseMock != nil {
		return d.synchroniseMock(id, hash)
	}
	// Make sure only one goroutine is ever allowed past this point at once
	if !atomic.CompareAndSwapInt32(&d.synchronising, 0, 1) {
		return errBusy
	}
	// allow concurrent
	defer atomic.StoreInt32(&d.synchronising, 0)

	// Post a user notification of the sync (only once per session)
	if atomic.CompareAndSwapInt32(&d.notified, 0, 1) {
		log.Info("Block synchronisation started")
	}
	// Reset the queue, peer set and wake channels to clean any internal leftover state
	d.queue.Reset()
	d.peers.Reset()
	// process both d.bodyWakeCh, d.receiptWakeCh
	for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
		select {
		case <-ch:
		default:
		}
	}
	// wait for d.headerCh, d.bodyCh, d.receiptCh
	for _, ch := range []chan dataPack{d.headerCh, d.bodyCh, d.receiptCh} {
		for empty := false; !empty; {
			select {
			case <-ch:
			default:
				empty = true
			}
		}
	}

	// headerProcCh
	for empty := false; !empty; {
		select {
		case <-d.headerProcCh:
		default:
			empty = true
		}
	}
	// Create cancel channel for aborting mid-flight and mark the master peer
	d.cancelLock.Lock()
	d.cancelCh = make(chan struct{})
	d.cancelPeer = id
	d.cancelLock.Unlock()

	defer d.Cancel() // No matter what, we can't leave the cancel channel open

	// Set the requested sync mode, unless it's forbidden
	d.mode = mode
	if d.mode == FastSync && atomic.LoadUint32(&d.fsPivotFails) >= fsCriticalTrials {
		d.mode = FullSync
	}
	// Retrieve the origin peer and initiate the downloading process
	p := d.peers.Peer(id)
	if p == nil {
		return errUnknownPeer
	}
	return d.syncWithPeer(p, hash, td)
}
```

syncWithPeer

```go
// syncWithPeer starts a block synchronization based on the hash chain from the
// specified peer and head hash.
func (d *Downloader) syncWithPeer(p *peerConnection, hash common.Hash, td *big.Int) (err error) {
	...
	// Look up the sync boundaries: the common ancestor and the target block
	latest, err := d.fetchHeight(p)
	if err != nil {
		return err
	}
	height := latest.Number.Uint64()
	// findAncestor
	origin, err := d.findAncestor(p, height)
	if err != nil {
		return err
	}
	// update chain height
	d.syncStatsLock.Lock()
	if d.syncStatsChainHeight <= origin || d.syncStatsChainOrigin > origin {
		d.syncStatsChainOrigin = origin
	}
	d.syncStatsChainHeight = height
	d.syncStatsLock.Unlock()

	// Initiate the sync using a concurrent header and content retrieval algorithm
	pivot := uint64(0)
	switch d.mode {
	case LightSync:
		pivot = height
	case FastSync:
		// Calculate the new fast/slow sync pivot point
		if d.fsPivotLock == nil {
			pivotOffset, err := rand.Int(rand.Reader, big.NewInt(int64(fsPivotInterval)))
			if err != nil {
				panic(fmt.Sprintf("Failed to access crypto random source: %v", err))
			}
			if height > uint64(fsMinFullBlocks)+pivotOffset.Uint64() {
				pivot = height - uint64(fsMinFullBlocks) - pivotOffset.Uint64()
			}
		} else {
			// Pivot point locked in, use this and do not pick a new one!
			pivot = d.fsPivotLock.Number.Uint64()
		}
		// If the point is below the origin, move origin back to ensure state download
		if pivot < origin {
			if pivot > 0 {
				origin = pivot - 1
			} else {
				origin = 0
			}
		}
		log.Debug("Fast syncing until pivot block", "pivot", pivot)
	}
	d.queue.Prepare(origin+1, d.mode, pivot, latest)
	if d.syncInitHook != nil {
		d.syncInitHook(origin, height)
	}
	// fetcher (header,bodies,receipts,headers)
	fetchers := []func() error{
		func() error { return d.fetchHeaders(p, origin+1) }, // Headers are always retrieved
		func() error { return d.fetchBodies(origin + 1) },   // Bodies are retrieved during normal and fast sync
		func() error { return d.fetchReceipts(origin + 1) }, // Receipts are retrieved during fast sync
		func() error { return d.processHeaders(origin+1, td) },
	}
	if d.mode == FastSync { // sync from latest block
		fetchers = append(fetchers, func() error { return d.processFastSyncContent(latest) })
	} else if d.mode == FullSync {
		fetchers = append(fetchers, d.processFullSyncContent)
	}
	err = d.spawnSync(fetchers)
	if err != nil && d.mode == FastSync && d.fsPivotLock != nil {
		// If sync failed in the critical section, bump the fail counter.
		atomic.AddUint32(&d.fsPivotFails, 1)
	}
	return err
}
```

spawnSync starts a goroutine for each fetcher and then blocks the fetcher error.

```go
// spawnSync runs d.process and all given fetcher functions to completion in
// separate goroutines, returning the first error that appears.
func (d *Downloader) spawnSync(fetchers []func() error) error {
	var wg sync.WaitGroup
	errc := make(chan error, len(fetchers))
	wg.Add(len(fetchers))
	for _, fn := range fetchers {
		fn := fn
		go func() { defer wg.Done(); errc <- fn() }()
	}
	// Wait for the first error, then terminate the others.
	var err error
	for i := 0; i < len(fetchers); i++ {
		if i == len(fetchers)-1 {
			// Close the queue when all fetchers have exited.
			// This will cause the block processor to end when
			// it has processed the queue.
			d.queue.Close()
		}
		if err = <-errc; err != nil {
			break
		}
	}
	d.queue.Close()
	d.Cancel()
	wg.Wait()
	return err
}
```

## headers processing

The fetchHeaders method is used to get the header. Then according to the obtained header to get information such as body and receipt.

```go
// fetchHeaders keeps retrieving headers concurrently from the number
// requested, until no more are returned, potentially throttling on the way. To
// facilitate concurrency but still protect against malicious nodes sending bad
// headers, we construct a header chain skeleton using the "origin" peer we are
// syncing with, and fill in the missing headers using anyone else. Headers from
// other peers are only accepted if they map cleanly to the skeleton. If no one
// can fill in the skeleton - not even the origin peer - it's assumed invalid and
// the origin is dropped.
func (d *Downloader) fetchHeaders(p *peerConnection, from uint64) error {
	p.log.Debug("Directing header downloads", "origin", from)
	defer p.log.Debug("Header download terminated")

	// Create a timeout timer, and the associated header fetcher
	skeleton := true            // Skeleton assembly phase or finishing up
	request := time.Now()       // time of the last skeleton fetch request
	timeout := time.NewTimer(0) // timer to dump a non-responsive active peer
	<-timeout.C                 // timeout channel should be initially empty
	defer timeout.Stop()

	var ttl time.Duration
	getHeaders := func(from uint64) {
		request = time.Now()

		ttl = d.requestTTL()
		timeout.Reset(ttl)

		if skeleton { // fill skeleton
			p.log.Trace("Fetching skeleton headers", "count", MaxHeaderFetch, "from", from)
			go p.peer.RequestHeadersByNumber(from+uint64(MaxHeaderFetch)-1, MaxSkeletonSize, MaxHeaderFetch-1, false)
		} else { // direct request
			p.log.Trace("Fetching full headers", "count", MaxHeaderFetch, "from", from)
			go p.peer.RequestHeadersByNumber(from, MaxHeaderFetch, 0, false)
		}
	}
	// Start pulling the header chain skeleton until all is done
	getHeaders(from)

	for {
		select {
		case <-d.cancelCh:
			return errCancelHeaderFetch

		case packet := <-d.headerCh: // The header returned on the network will be delivered to the headerCh channel.
			// Make sure the active peer is giving us the skeleton headers
			if packet.PeerId() != p.id {
				log.Debug("Received skeleton from incorrect peer", "peer", packet.PeerId())
				break
			}
			headerReqTimer.UpdateSince(request)
			timeout.Stop()

			// If the skeleton's finished, pull any remaining head headers directly from the origin
			if packet.Items() == 0 && skeleton {
				skeleton = false
				getHeaders(from)
				continue
			}
			// If no more headers are inbound, notify the content fetchers and return
			// tell headerProcCh channel
			if packet.Items() == 0 {
				p.log.Debug("No more headers available")
				select {
				case d.headerProcCh <- nil:
					return nil
				case <-d.cancelCh:
					return errCancelHeaderFetch
				}
			}
			headers := packet.(*headerPack).headers

			// If we received a skeleton batch, resolve internals concurrently
			if skeleton { // fill the skeleton in this method
				filled, proced, err := d.fillHeaderSkeleton(from, headers)
				if err != nil {
					p.log.Debug("Skeleton chain invalid", "err", err)
					return errInvalidChain
				}
				headers = filled[proced:]
				// how many have been processed
				from += uint64(proced)
			}
			// Insert all the new headers and fetch the next batch
			if len(headers) > 0 {
				p.log.Trace("Scheduling new headers", "count", len(headers), "from", from)
				select {
				case d.headerProcCh <- headers:
				case <-d.cancelCh:
					return errCancelHeaderFetch
				}
				from += uint64(len(headers))
			}
			getHeaders(from)

		case <-timeout.C:
			// Header retrieval timed out, consider the peer bad and drop
			p.log.Debug("Header request timed out", "elapsed", ttl)
			headerTimeoutMeter.Mark(1)
			d.dropPeer(p.id)

			// Finish the sync gracefully instead of dumping the gathered data though
			for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
				select {
				case ch <- false:
				case <-d.cancelCh:
				}
			}
			select {
			case d.headerProcCh <- nil:
			case <-d.cancelCh:
			}
			return errBadPeer
		}
	}
}
```

The processHeaders method, which takes the header from the headerProcCh channel. And the obtained header is thrown into the queue for scheduling, so that the body fetcher or the receipt fetcher can receive the fetch task.

```go
// processHeaders takes batches of retrieved headers from an input channel and
// keeps processing and scheduling them into the header chain and downloader's
// queue until the stream ends or a failure occurs.
func (d *Downloader) processHeaders(origin uint64, td *big.Int) error {
	// Calculate the pivoting point for switching from fast to slow sync
	pivot := d.queue.FastSyncPivot()

	// Keep a count of uncertain headers to roll back
	// rollback: Used to handle this logic if a point fails. Then the 2048 nodes previously inserted must be rolled back. Because the security does not meet the requirements, you can refer to the fast sync documentation for details.
	rollback := []*types.Header{}
	defer func() { // this function is used to rollback when the error exists
		if len(rollback) > 0 {
			// Flatten the headers and roll them back
			hashes := make([]common.Hash, len(rollback))
			for i, header := range rollback {
				hashes[i] = header.Hash()
			}
			lastHeader, lastFastBlock, lastBlock := d.lightchain.CurrentHeader().Number, common.Big0, common.Big0
			if d.mode != LightSync {
				lastFastBlock = d.blockchain.CurrentFastBlock().Number()
				lastBlock = d.blockchain.CurrentBlock().Number()
			}
			d.lightchain.Rollback(hashes)
			curFastBlock, curBlock := common.Big0, common.Big0
			if d.mode != LightSync {
				curFastBlock = d.blockchain.CurrentFastBlock().Number()
				curBlock = d.blockchain.CurrentBlock().Number()
			}
			log.Warn("Rolled back headers", "count", len(hashes),
				"header", fmt.Sprintf("%d->%d", lastHeader, d.lightchain.CurrentHeader().Number),
				"fast", fmt.Sprintf("%d->%d", lastFastBlock, curFastBlock),
				"block", fmt.Sprintf("%d->%d", lastBlock, curBlock))

			// If we're already past the pivot point, this could be an attack, thread carefully
			if rollback[len(rollback)-1].Number.Uint64() > pivot {
				// If we didn't ever fail, lock in the pivot header (must! not! change!)
				if atomic.LoadUint32(&d.fsPivotFails) == 0 {
					for _, header := range rollback {
						if header.Number.Uint64() == pivot {
							log.Warn("Fast-sync pivot locked in", "number", pivot, "hash", header.Hash())
							d.fsPivotLock = header
						}
					}
				}
			}
		}
	}()

	// Wait for batches of headers to process
	gotHeaders := false

	for {
		select {
		case <-d.cancelCh:
			return errCancelHeaderProcessing

		case headers := <-d.headerProcCh:
			// Terminate header processing if we synced up
			if len(headers) == 0 { // processing completed
				// Notify everyone that headers are fully processed
				for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
					select {
					case ch <- false:
					case <-d.cancelCh:
					}
				}
				// If no headers were retrieved at all, the peer violated it's TD promise that it had a
				// better chain compared to ours. The only exception is if it's promised blocks were
				// already imported by other means (e.g. fecher):
				//
				// R <remote peer>, L <local node>: Both at block 10
				// R: Mine block 11, and propagate it to L
				// L: Queue block 11 for import
				// L: Notice that R's head and TD increased compared to ours, start sync
				// L: Import of block 11 finishes
				// L: Sync begins, and finds common ancestor at 11
				// L: Request new headers up from 11 (R's TD was higher, it must have something)
				// R: Nothing to give
				if d.mode != LightSync { // The opponent's TD is bigger than us, but she didn't get anything. Then think that the other party is the wrong partner. Will disconnect from the other party
					if !gotHeaders && td.Cmp(d.blockchain.GetTdByHash(d.blockchain.CurrentBlock().Hash())) > 0 {
						return errStallingPeer
					}
				}
				// If fast or light syncing, ensure promised headers are indeed delivered. This is
				// needed to detect scenarios where an attacker feeds a bad pivot and then bails out
				// of delivering the post-pivot blocks that would flag the invalid content.
				//
				// This check cannot be executed "as is" for full imports, since blocks may still be
				// queued for processing when the header download completes. However, as long as the
				// peer gave us something useful, we're already happy/progressed (above check).
				if d.mode == FastSync || d.mode == LightSync {
					if td.Cmp(d.lightchain.GetTdByHash(d.lightchain.CurrentHeader().Hash())) > 0 {
						return errStallingPeer
					}
				}
				// Disable any rollback and return
				rollback = nil
				return nil
			}
			// Otherwise split the chunk of headers into batches and process them
			gotHeaders = true

			for len(headers) > 0 {
				// Terminate if something failed in between processing chunks
				select {
				case <-d.cancelCh:
					return errCancelHeaderProcessing
				default:
				}
				// Select the next chunk of headers to import
				limit := maxHeadersProcess
				if limit > len(headers) {
					limit = len(headers)
				}
				chunk := headers[:limit]

				// In case of header only syncing, validate the chunk immediately
				if d.mode == FastSync || d.mode == LightSync { // download block only
					// Collect the yet unknown headers to mark them as uncertain
					unknown := make([]*types.Header, 0, len(headers))
					for _, header := range chunk {
						if !d.lightchain.HasHeader(header.Hash(), header.Number.Uint64()) {
							unknown = append(unknown, header)
						}
					}
					// If we're importing pure headers, verify based on their recentness
					// verify every few blocks
					frequency := fsHeaderCheckFrequency
					if chunk[len(chunk)-1].Number.Uint64()+uint64(fsHeaderForceVerify) > pivot {
						frequency = 1
					}
					// The lightchain is equal to the chain by default. Insert the block header. If it fails then you need to roll back.
					if n, err := d.lightchain.InsertHeaderChain(chunk, frequency); err != nil {
						// If some headers were inserted, add them too to the rollback list
						if n > 0 {
							rollback = append(rollback, chunk[:n]...)
						}
						log.Debug("Invalid header encountered", "number", chunk[n].Number, "hash", chunk[n].Hash(), "err", err)
						return errInvalidChain
					}
					// All verifications passed, store newly found uncertain headers
					rollback = append(rollback, unknown...)
					if len(rollback) > fsHeaderSafetyNet {
						rollback = append(rollback[:0], rollback[len(rollback)-fsHeaderSafetyNet:]...)
					}
				}
				// If we're fast syncing and just pulled in the pivot, make sure it's the one locked in
				if d.mode == FastSync && d.fsPivotLock != nil && chunk[0].Number.Uint64() <= pivot && chunk[len(chunk)-1].Number.Uint64() >= pivot { // if PivotLock, check the same hash
					if pivot := chunk[int(pivot-chunk[0].Number.Uint64())]; pivot.Hash() != d.fsPivotLock.Hash() {
						log.Warn("Pivot doesn't match locked in one", "remoteNumber", pivot.Number, "remoteHash", pivot.Hash(), "localNumber", d.fsPivotLock.Number, "localHash", d.fsPivotLock.Hash())
						return errInvalidChain
					}
				}
				// Unless we're doing light chains, schedule the headers for associated content retrieval
				if d.mode == FullSync || d.mode == FastSync {
					// If we've reached the allowed number of pending headers, stall a bit
					for d.queue.PendingBlocks() >= maxQueuedHeaders || d.queue.PendingReceipts() >= maxQueuedHeaders {
						select {
						case <-d.cancelCh:
							return errCancelHeaderProcessing
						case <-time.After(time.Second):
						}
					}
					// Otherwise insert the headers for content retrieval
					inserts := d.queue.Schedule(chunk, origin)
					if len(inserts) != len(chunk) {
						log.Debug("Stale headers")
						return errBadPeer
					}
				}
				headers = headers[limit:]
				origin += uint64(limit)
			}
			// Signal the content downloaders of the availablility of new tasks
			for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
				select {
				case ch <- true:
				default:
				}
			}
		}
	}
}
```

## body processing

The fetchBodies function defines some closure functions and then calls the fetchParts function.

```go
// fetchBodies iteratively downloads the scheduled block bodies, taking any
// available peers, reserving a chunk of blocks for each, waiting for delivery
// and also periodically checking for timeouts.
// fetchBodies: The download block is continuously downloaded, and any available links are used in the middle, and a part of the block is reserved for each link, waiting for the block to be delivered, and periodically checking whether it times out.
func (d *Downloader) fetchBodies(from uint64) error {
	log.Debug("Downloading block bodies", "origin", from)

	var (
		deliver = func(packet dataPack) (int, error) {
			pack := packet.(*bodyPack)
			return d.queue.DeliverBodies(pack.peerId, pack.transactions, pack.uncles)
		}
		expire   = func() map[string]int { return d.queue.ExpireBodies(d.requestTTL()) }  // time to live
		fetch    = func(p *peerConnection, req *fetchRequest) error { return p.FetchBodies(req) }
		capacity = func(p *peerConnection) int { return p.BlockCapacity(d.requestRTT()) } // relay time
		setIdle  = func(p *peerConnection, accepted int) { p.SetBodiesIdle(accepted) } // peer idle
	)
	err := d.fetchParts(errCancelBodyFetch, d.bodyCh, deliver, d.bodyWakeCh, expire,
		d.queue.PendingBlocks, d.queue.InFlightBlocks, d.queue.ShouldThrottleBlocks, d.queue.ReserveBodies,
		d.bodyFetchHook, fetch, d.queue.CancelBodies, capacity, d.peers.BodyIdlePeers, setIdle, "bodies")

	log.Debug("Block body download terminated", "err", err)
	return err
}
```

fetchParts

```go
// fetchParts iteratively downloads scheduled block parts, taking any available
// peers, reserving a chunk of fetch requests for each, waiting for delivery and
// also periodically checking for timeouts.
// As the scheduling/timeout logic mostly is the same for all downloaded data
// types, this method is used by each for data gathering and is instrumented with
// various callbacks to handle the slight differences between processing them.
// The instrumentation parameters:
//  - errCancel:   error type to return if the fetch operation is cancelled (mostly makes logging nicer)
//  - deliveryCh:  channel from which to retrieve downloaded data packets (merged from all concurrent peers)
//  - deliver:     processing callback to deliver data packets into type specific download queues (usually within `queue`)
//  - wakeCh:      notification channel for waking the fetcher when new tasks are available (or sync completed) use to notify fetcher of new task
//  - expire:      task callback method to abort requests that took too long and return the faulty peers (traffic shaping)
//  - pending:     task callback for the number of requests still needing download (detect completion/non-completability)
//  - inFlight:    task callback for the number of in-progress requests (wait for all active downloads to finish)
//  - throttle:    task callback to check if the processing queue is full and activate throttling (bound memory use)
//  - reserve:     task callback to reserve new download tasks to a particular peer (also signals partial completions)
//  - fetchHook:   tester callback to notify of new tasks being initiated (allows testing the scheduling logic)
//  - fetch:       network callback to actually send a particular download request to a physical remote peer
//  - cancel:      task callback to abort an in-flight download request and allow rescheduling it (in case of lost peer)
//  - capacity:    network callback to retrieve the estimated type-specific bandwidth capacity of a peer (traffic shaping)
//  - idle:        network callback to retrieve the currently (type specific) idle peers that can be assigned tasks
//  - setIdle:     network callback to set a peer back to idle and update its estimated capacity (traffic shaping)
//  - kind:        textual label of the type being downloaded to display in log mesages
func (d *Downloader) fetchParts(errCancel error, deliveryCh chan dataPack, deliver func(dataPack) (int, error), wakeCh chan bool,
	expire func() map[string]int, pending func() int, inFlight func() bool, throttle func() bool, reserve func(*peerConnection, int) (*fetchRequest, bool, error),
	fetchHook func([]*types.Header), fetch func(*peerConnection, *fetchRequest) error, cancel func(*fetchRequest), capacity func(*peerConnection) int,
	idle func() ([]*peerConnection, int), setIdle func(*peerConnection, int), kind string) error {

	// Create a ticker to detect expired retrieval tasks
	ticker := time.NewTicker(100 * time.Millisecond)
	defer ticker.Stop()

	update := make(chan struct{}, 1)

	// Prepare the queue and fetch block parts until the block header fetcher's done
	finished := false
	for {
		select {
		case <-d.cancelCh:
			return errCancel

		case packet := <-deliveryCh:
			// If the peer was previously banned and failed to deliver it's pack
			// in a reasonable time frame, ignore it's message.
			if peer := d.peers.Peer(packet.PeerId()); peer != nil {
				// Deliver the received chunk of data and check chain validity
				accepted, err := deliver(packet)
				if err == errInvalidChain {
					return err
				}
				// Unless a peer delivered something completely else than requested (usually
				// caused by a timed out request which came through in the end), set it to
				// idle. If the delivery's stale, the peer should have already been idled.
				if err != errStaleDelivery {
					setIdle(peer, accepted)
				}
				// Issue a log to the user to see what's going on
				switch {
				case err == nil && packet.Items() == 0:
					peer.log.Trace("Requested data not delivered", "type", kind)
				case err == nil:
					peer.log.Trace("Delivered new batch of data", "type", kind, "count", packet.Stats())
				default:
					peer.log.Trace("Failed to deliver retrieved data", "type", kind, "err", err)
				}
			}
			// Blocks assembled, try to update the progress
			select {
			case update <- struct{}{}:
			default:
			}

		case cont := <-wakeCh:
			// The header fetcher sent a continuation flag, check if it's done
			if !cont {
				finished = true
			}
			// Headers arrive, try to update the progress
			select {
			case update <- struct{}{}:
			default:
			}

		case <-ticker.C:
			// Sanity check update the progress
			select {
			case update <- struct{}{}:
			default:
			}

		case <-update:
			// Short circuit if we lost all our peers
			if d.peers.Len() == 0 {
				return errNoPeers
			}
			// Check for fetch request timeouts and demote the responsible peers
			for pid, fails := range expire() {
				if peer := d.peers.Peer(pid); peer != nil {
					// If a lot of retrieval elements expired, we might have overestimated the remote peer or perhaps
					// ourselves. Only reset to minimal throughput but don't drop just yet. If even the minimal times
					// out that sync wise we need to get rid of the peer.

					// The reason the minimum threshold is 2 is because the downloader tries to estimate the bandwidth
					// and latency of a peer separately, which requires pushing the measures capacity a bit and seeing
					// how response times reacts, to it always requests one more than the minimum (i.e. min 2).

					if fails > 2 {
						peer.log.Trace("Data delivery timed out", "type", kind)
						setIdle(peer, 0)
					} else {
						peer.log.Debug("Stalling delivery, dropping", "type", kind)
						d.dropPeer(pid)
					}
				}
			}
			// If there's nothing more to fetch, wait or terminate
			if pending() == 0 { // no task
				if !inFlight() && finished {
					log.Debug("Data fetching completed", "type", kind)
					return nil
				}
				break
			}
			// Send a download request to all idle peers, until throttled
			progressed, throttled, running := false, false, inFlight()
			idles, total := idle()

			for _, peer := range idles {
				// Short circuit if throttling activated
				if throttle() {
					throttled = true
					break
				}
				// Short circuit if there is no more available task.
				if pending() == 0 {
					break
				}
				// Reserve a chunk of fetches for a peer. A nil can mean either that
				// no more headers are available, or that the peer is known not to
				// have them.
				request, progress, err := reserve(peer, capacity(peer))
				if err != nil {
					return err
				}
				if progress {
					progressed = true
				}
				if request == nil {
					continue
				}
				if request.From > 0 {
					peer.log.Trace("Requesting new batch of data", "type", kind, "from", request.From)
				} else if len(request.Headers) > 0 {
					peer.log.Trace("Requesting new batch of data", "type", kind, "count", len(request.Headers), "from", request.Headers[0].Number)
				} else {
					peer.log.Trace("Requesting new batch of data", "type", kind, "count", len(request.Hashes))
				}
				// Fetch the chunk and make sure any errors return the hashes to the queue
				if fetchHook != nil {
					fetchHook(request.Headers)
				}
				if err := fetch(peer, request); err != nil {
					// Although we could try and make an attempt to fix this, this error really
					// means that we've double allocated a fetch task to a peer. If that is the
					// case, the internal state of the downloader and the queue is very wrong so
					// better hard crash and note the error instead of silently accumulating into
					// a much bigger issue.
					panic(fmt.Sprintf("%v: %s fetch assignment failed", peer, kind))
				}
				running = true
			}
			// Make sure that we have peers available for fetching. If all peers have been tried
			// and all failed throw an error
			if !progressed && !throttled && !running && len(idles) == total && pending() > 0 {
				return errPeersUnavailable
			}
		}
	}
}
```

## receipt processing

The processing of the receipt is similar to the body.

```go
// fetchReceipts iteratively downloads the scheduled block receipts, taking any
// available peers, reserving a chunk of receipts for each, waiting for delivery
// and also periodically checking for timeouts.
func (d *Downloader) fetchReceipts(from uint64) error {
	log.Debug("Downloading transaction receipts", "origin", from)

	var (
		deliver = func(packet dataPack) (int, error) {
			pack := packet.(*receiptPack)
			return d.queue.DeliverReceipts(pack.peerId, pack.receipts)
		}
		expire   = func() map[string]int { return d.queue.ExpireReceipts(d.requestTTL()) }
		fetch    = func(p *peerConnection, req *fetchRequest) error { return p.FetchReceipts(req) }
		capacity = func(p *peerConnection) int { return p.ReceiptCapacity(d.requestRTT()) }
		setIdle  = func(p *peerConnection, accepted int) { p.SetReceiptsIdle(accepted) }
	)
	err := d.fetchParts(errCancelReceiptFetch, d.receiptCh, deliver, d.receiptWakeCh, expire,
		d.queue.PendingReceipts, d.queue.InFlightReceipts, d.queue.ShouldThrottleReceipts, d.queue.ReserveReceipts,
		d.receiptFetchHook, fetch, d.queue.CancelReceipts, capacity, d.peers.ReceiptIdlePeers, setIdle, "receipts")

	log.Debug("Transaction receipt download terminated", "err", err)
	return err
}
```

## processFastSyncContent and processFullSyncContent

```go
// processFastSyncContent takes fetch results from the queue and writes them to the
// database. It also controls the synchronisation of state nodes of the pivot block.
func (d *Downloader) processFastSyncContent(latest *types.Header) error {
	// Start syncing state of the reported head block.
	// This should get us most of the state of the pivot block.
	stateSync := d.syncState(latest.Root)
	defer stateSync.Cancel()
	go func() {
		if err := stateSync.Wait(); err != nil {
			d.queue.Close() // wake up WaitResults
		}
	}()

	pivot := d.queue.FastSyncPivot()
	for {
		results := d.queue.WaitResults() // waiting for queue output
		if len(results) == 0 {
			return stateSync.Cancel()
		}
		if d.chainInsertHook != nil {
			d.chainInsertHook(results)
		}
		P, beforeP, afterP := splitAroundPivot(pivot, results)
		if err := d.commitFastSyncData(beforeP, stateSync); err != nil {
			return err
		}
		if P != nil {
			stateSync.Cancel()
			if err := d.commitPivotBlock(P); err != nil {
				return err
			}
		}
		// For all nodes after the pivot point, you need to follow the full processing.
		if err := d.importBlockResults(afterP); err != nil {
			return err
		}
	}
}
```

processFullSyncContent, relatively simple. Get the block from the queue and insert it.

```go
// processFullSyncContent takes fetch results from the queue and imports them into the chain.
func (d *Downloader) processFullSyncContent() error {
	for {
		results := d.queue.WaitResults()
		if len(results) == 0 {
			return nil
		}
		if d.chainInsertHook != nil {
			d.chainInsertHook(results)
		}
		if err := d.importBlockResults(results); err != nil {
			return err
		}
	}
}
```
