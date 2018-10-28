Statesync is used to get the trie tree of all the states of the block specified by the pivot point, that is, the information of all accounts, including ordinary accounts and contract accounts.

## data structure

stateSync schedules the download of a request for a particular state trie defined by a given state root.

```go
// stateSync schedules requests for downloading a particular state trie defined
// by a given state root.
type stateSync struct {
	d *Downloader // Downloader instance to access and manage current peerset

	sched  *trie.TrieSync             // State trie sync scheduler defining the tasks
	keccak hash.Hash                  // Keccak256 hasher to verify deliveries with
	tasks  map[common.Hash]*stateTask // Set of tasks currently queued for retrieval

	numUncommitted   int
	bytesUncommitted int

	deliver    chan *stateReq // Delivery channel multiplexing peer responses
	cancel     chan struct{}  // Channel to signal a termination request
	cancelOnce sync.Once      // Ensures cancel only ever gets called once
	done       chan struct{}  // Channel to signal termination completion
	err        error          // Any error hit during sync (set before completion)
}
```

Constructor

```go
func newStateSync(d *Downloader, root common.Hash) *stateSync {
	return &stateSync{
		d:       d,
		sched:   state.NewStateSync(root, d.stateDB),
		keccak:  sha3.NewKeccak256(),
		tasks:   make(map[common.Hash]*stateTask),
		deliver: make(chan *stateReq),
		cancel:  make(chan struct{}),
		done:    make(chan struct{}),
	}
}
```

NewStateSync

```go
// NewStateSync create a new state trie download scheduler.
func NewStateSync(root common.Hash, database trie.DatabaseReader) *trie.TrieSync {
	var syncer *trie.TrieSync
	callback := func(leaf []byte, parent common.Hash) error {
		var obj Account
		if err := rlp.Decode(bytes.NewReader(leaf), &obj); err != nil {
			return err
		}
		syncer.AddSubTrie(obj.Root, 64, parent, nil)
		syncer.AddRawEntry(common.BytesToHash(obj.CodeHash), 64, parent)
		return nil
	}
	syncer = trie.NewTrieSync(root, database, callback)
	return syncer
}
```

syncState, this function is called by the downloader.

```go
// syncState starts downloading state with the given root hash.
func (d *Downloader) syncState(root common.Hash) *stateSync {
	s := newStateSync(d, root)
	select {
	case d.stateSyncStart <- s:
	case <-d.quitCh:
		s.err = errCancelStateFetch
		close(s.done)
	}
	return s
}
```

## Startup

A new goroutine is started in the downloader to run the stateFetcher function. This function first tries to get information from the stateSyncStart channel. The syncState function will send data to the stateSyncStart channel.

```go
// stateFetcher manages the active state sync and accepts requests
// on its behalf.
func (d *Downloader) stateFetcher() {
	for {
		select {
		case s := <-d.stateSyncStart:
			for next := s; next != nil; { // This for loop represents that the downloader can change the objects that need to be synchronized at any time by sending a signal.
				next = d.runStateSync(next)
			}
		case <-d.stateCh:
			// Ignore state responses while no sync is running.
		case <-d.quitCh:
			return
		}
	}
}
```

Let's see below where we will call the syncState() function. The processFastSyncContent function will start when the peer is first discovered.

```go
// processFastSyncContent takes fetch results from the queue and writes them to the
// database. It also controls the synchronisation of state nodes of the pivot block.
func (d *Downloader) processFastSyncContent(latest *types.Header) error {
	// Start syncing state of the reported head block. This should get us most of
	// the state of the pivot block.
	stateSync := d.syncState(latest.Root)
	defer stateSync.Cancel()
	go func() {
		if err := stateSync.Wait(); err != nil && err != errCancelStateFetch {
			d.queue.Close() // wake up Results
		}
	}()
	// Figure out the ideal pivot block. Note, that this goalpost may move if the
	// sync takes long enough for the chain head to move significantly.
	pivot := uint64(0)
	if height := latest.Number.Uint64(); height > uint64(fsMinFullBlocks) {
		pivot = height - uint64(fsMinFullBlocks)
	}
	// To cater for moving pivot points, track the pivot block and subsequently
	// accumulated download results separately.
	var (
		oldPivot *fetchResult   // Locked in pivot block, might change eventually
		oldTail  []*fetchResult // Downloaded content after the pivot
	)
	for {
		// Wait for the next batch of downloaded data to be available, and if the pivot
		// block became stale, move the goalpost
		results := d.queue.Results(oldPivot == nil) // Block if we're not monitoring pivot staleness
		if len(results) == 0 {
			// If pivot sync is done, stop
			if oldPivot == nil {
				return stateSync.Cancel()
			}
			// If sync failed, stop
			select {
			case <-d.cancelCh:
				return stateSync.Cancel()
			default:
			}
		}
		if d.chainInsertHook != nil {
			d.chainInsertHook(results)
		}
		if oldPivot != nil {
			results = append(append([]*fetchResult{oldPivot}, oldTail...), results...)
		}
		// Split around the pivot block and process the two sides via fast/full sync
		if atomic.LoadInt32(&d.committed) == 0 {
			latest = results[len(results)-1].Header
			if height := latest.Number.Uint64(); height > pivot+2*uint64(fsMinFullBlocks) {
				log.Warn("Pivot became stale, moving", "old", pivot, "new", height-uint64(fsMinFullBlocks))
				pivot = height - uint64(fsMinFullBlocks)
			}
		}
		P, beforeP, afterP := splitAroundPivot(pivot, results)
		if err := d.commitFastSyncData(beforeP, stateSync); err != nil {
			return err
		}
		if P != nil {
			// If new pivot block found, cancel old state retrieval and restart
			if oldPivot != P {
				stateSync.Cancel()

				stateSync = d.syncState(P.Header.Root)
				defer stateSync.Cancel()
				go func() {
					if err := stateSync.Wait(); err != nil && err != errCancelStateFetch {
						d.queue.Close() // wake up Results
					}
				}()
				oldPivot = P
			}
			// Wait for completion, occasionally checking for pivot staleness
			select {
			case <-stateSync.done:
				if stateSync.err != nil {
					return stateSync.err
				}
				if err := d.commitPivotBlock(P); err != nil {
					return err
				}
				oldPivot = nil

			case <-time.After(time.Second):
				oldTail = afterP
				continue
			}
		}
		// Fast sync done, pivot commit done, full import
		if err := d.importBlockResults(afterP); err != nil {
			return err
		}
	}
}
```

runStateSync, this method gets the downloaded state from stateCh, and then delivers it to the deliver channel for others to process.

```go
// runStateSync runs a state synchronisation until it completes or another root
// hash is requested to be switched over to.
func (d *Downloader) runStateSync(s *stateSync) *stateSync {
	var (
		active   = make(map[string]*stateReq) // Currently in-flight requests
		// Completed or failed requests
		finished []*stateReq
		timeout  = make(chan *stateReq)       // Timed out active requests
	)
	defer func() {
		// Cancel active request timers on exit. Also set peers to idle so they're
		// available for the next sync.
		for _, req := range active {
			req.timer.Stop()
			req.peer.SetNodeDataIdle(len(req.items))
		}
	}()
	// Run the state sync.
	go s.run()
	defer s.Cancel()

	// Listen for peer departure events to cancel assigned tasks
	peerDrop := make(chan *peerConnection, 1024)
	peerSub := s.d.peers.SubscribePeerDrops(peerDrop)
	defer peerSub.Unsubscribe()

	for {
		// Enable sending of the first buffered element if there is one.
		var (
			deliverReq   *stateReq
			deliverReqCh chan *stateReq
		)
		if len(finished) > 0 {
			deliverReq = finished[0]
			deliverReqCh = s.deliver
		}

		select {
		// The stateSync lifecycle:
		case next := <-d.stateSyncStart:
			return next

		case <-s.done:
			return nil

		// Send the next finished request to the current sync:
		case deliverReqCh <- deliverReq:
			finished = append(finished[:0], finished[1:]...)

		// Handle incoming state packs:
		// Process incoming packets. The data that the downloader receives the state will be sent to this channel.
		case pack := <-d.stateCh:
			// Discard any data not requested (or previsouly timed out)
			req := active[pack.PeerId()]
			if req == nil {
				log.Debug("Unrequested node data", "peer", pack.PeerId(), "len", pack.Items())
				continue
			}
			// Finalize the request and queue up for processing
			req.timer.Stop()
			req.response = pack.(*statePack).states

			finished = append(finished, req)
			delete(active, pack.PeerId())

			// Handle dropped peer connections:
		case p := <-peerDrop:
			// Skip if no request is currently pending
			req := active[p.id]
			if req == nil {
				continue
			}
			// Finalize the request and queue up for processing
			req.timer.Stop()
			req.dropped = true

			finished = append(finished, req)
			delete(active, p.id)

		// Handle timed-out requests:
		case req := <-timeout:
			// If the peer is already requesting something else, ignore the stale timeout.
			// This can happen when the timeout and the delivery happens simultaneously,
			// causing both pathways to trigger.
			if active[req.peer.id] != req {
				continue
			}
			// Move the timed out data back into the download queue
			finished = append(finished, req)
			delete(active, req.peer.id)

		// Track outgoing state requests:
		case req := <-d.trackStateReq:
			// If an active request already exists for this peer, we have a problem. In
			// theory the trie node schedule must never assign two requests to the same
			// peer. In practive however, a peer might receive a request, disconnect and
			// immediately reconnect before the previous times out. In this case the first
			// request is never honored, alas we must not silently overwrite it, as that
			// causes valid requests to go missing and sync to get stuck.
			if old := active[req.peer.id]; old != nil {
				log.Warn("Busy peer assigned new state fetch", "peer", old.peer.id)

				// Make sure the previous one doesn't get siletly lost
				old.timer.Stop()
				old.dropped = true

				finished = append(finished, old)
			}
			// Start a timer to notify the sync loop if the peer stalled.
			req.timer = time.AfterFunc(req.timeout, func() {
				select {
				case timeout <- req:
				case <-s.done:
					// Prevent leaking of timer goroutines in the unlikely case where a
					// timer is fired just before exiting runStateSync.
				}
			})
			active[req.peer.id] = req
		}
	}
}
```

Run and loop methods, get tasks, assign tasks, and get results.

```go
func (s *stateSync) run() {
	s.err = s.loop()
	close(s.done)
}

// loop is the main event loop of a state trie sync. It it responsible for the
// assignment of new tasks to peers (including sending it to them) as well as
// for the processing of inbound data. Note, that the loop does not directly
// receive data from peers, rather those are buffered up in the downloader and
// pushed here async. The reason is to decouple processing from data receipt
// and timeouts.
func (s *stateSync) loop() error {
	// Listen for new peer events to assign tasks to them
	newPeer := make(chan *peerConnection, 1024)
	peerSub := s.d.peers.SubscribeNewPeers(newPeer)
	defer peerSub.Unsubscribe()

	// Keep assigning new tasks until the sync completes or aborts
	for s.sched.Pending() > 0 {
		// Refresh the data from the cache to the persistent store. This is the size specified by the command line --cache.
		if err := s.commit(false); err != nil {
			return err
		}

		s.assignTasks()
		// Tasks assigned, wait for something to happen
		select {
		case <-newPeer:
			// New peer arrived, try to assign it download tasks

		case <-s.cancel:
			return errCancelStateFetch

		case req := <-s.deliver:
			// Received the return message sent by the runStateSync method. Note that the return message contains the successful request and also contains the unsuccessful request.
			// Response, disconnect or timeout triggered, drop the peer if stalling
			log.Trace("Received node data response", "peer", req.peer.id, "count", len(req.response), "dropped", req.dropped, "timeout", !req.dropped && req.timedOut())
			// at least more than 2 requests to process
			if len(req.items) <= 2 && !req.dropped && req.timedOut() {
				// 2 items are the minimum requested, if even that times out, we've no use of
				// this peer at the moment.
				log.Warn("Stalling state sync, dropping peer", "peer", req.peer.id)
				s.d.dropPeer(req.peer.id)
			}
			// Process all the received blobs and check for stale delivery
			stale, err := s.process(req)
			if err != nil {
				log.Warn("Node data write error", "err", err)
				return err
			}
			// The the delivery contains requested data, mark the node idle (otherwise it's a timed out delivery)
			if !stale {
				req.peer.SetNodeDataIdle(len(req.response))
			}
		}
	}
	return s.commit(true)
}
```
