## agent

The agent is the object of specific mining. The process it performs is to accept the calculated block header, calculate the mixhash and nonce, and return the mined block header.

The CpuAgent is constructed. Generally, the CPU is not used for mining. Generally, mining is performed using a dedicated GPU for mining. The code for GPU mining will not be reflected here.

```go
type CpuAgent struct {
	mu sync.Mutex

	workCh        chan *Work       // Accepting the channel for mining tasks
	stop          chan struct{}
	quitCurrentOp chan struct{}
	returnCh      chan<- *Result   // Return channel after mining completion

	chain  consensus.ChainReader // Get blockchain information
	engine consensus.Engine      // Consensus engine, here refers to the Pow engine

	isMining int32 // isMining indicates whether the agent is currently mining
}

func NewCpuAgent(chain consensus.ChainReader, engine consensus.Engine) *CpuAgent {
	miner := &CpuAgent{
		chain:  chain,
		engine: engine,
		stop:   make(chan struct{}, 1),
		workCh: make(chan *Work, 1),
	}
	return miner
}
```

Set the return value channel and get the Work channel to facilitate the external value and get the return information.

```go
func (self *CpuAgent) Work() chan<- *Work            { return self.workCh }
func (self *CpuAgent) SetReturnCh(ch chan<- *Result) { self.returnCh = ch }
```

Start and message loop, if you have started mining, then exit directly, otherwise start update goroutine update accept tasks from workCh, mine, or accept exit information, exit.

```go
func (self *CpuAgent) Start() {
	if !atomic.CompareAndSwapInt32(&self.isMining, 0, 1) {
		return // agent already started
	}
	go self.update()
}

func (self *CpuAgent) update() {
out:
	for {
		select {
		case work := <-self.workCh:
			self.mu.Lock()
			if self.quitCurrentOp != nil {
				close(self.quitCurrentOp)
			}
			self.quitCurrentOp = make(chan struct{})
			go self.mine(work, self.quitCurrentOp)
			self.mu.Unlock()
		case <-self.stop:
			self.mu.Lock()
			if self.quitCurrentOp != nil {
				close(self.quitCurrentOp)
				self.quitCurrentOp = nil
			}
			self.mu.Unlock()
			break out
		}
	}
}
```

Mine, mining, call the consistency engine for mining, if the mining is successful, send the message to returnCh.

```go
func (self *CpuAgent) mine(work *Work, stop <-chan struct{}) {
	if result, err := self.engine.Seal(self.chain, work.Block, stop); result != nil {
		log.Info("Successfully sealed new block", "number", result.Number(), "hash", result.Hash())
		self.returnCh <- &Result{work, result}
	} else {
		if err != nil {
			log.Warn("Block sealing failed", "err", err)
		}
		self.returnCh <- nil
	}
}
```

GetHashRate, this function returns the current HashRate.

```go
func (self *CpuAgent) GetHashRate() int64 {
	if pow, ok := self.engine.(consensus.PoW); ok {
		return int64(pow.Hashrate())
	}
	return 0
}
```

## remote_agent

Remote_agent provides a set of RPC interfaces that enable remote miners to perform mining functions. For example, I have a mining machine. The inside of the mining machine does not run the Ethereum node. The mining machine first obtains the current task from remote_agent, and then performs mining calculation. When the mining is completed, the calculation result is submitted and the mining is completed.

Data structure and construction

```go
type RemoteAgent struct {
	mu sync.Mutex

	quitCh   chan struct{}
	workCh   chan *Work  		// accept the task
	returnCh chan<- *Result		// Result return

	chain       consensus.ChainReader
	engine      consensus.Engine
	currentWork *Work	// Current task
	work        map[common.Hash]*Work // Tasks that have not yet been submitted, are being calculated

	hashrateMu sync.RWMutex
	hashrate   map[common.Hash]hashrate  // Hashrate of the task being calculated

	running int32 // running indicates whether the agent is active. Call atomically
}

func NewRemoteAgent(chain consensus.ChainReader, engine consensus.Engine) *RemoteAgent {
	return &RemoteAgent{
		chain:    chain,
		engine:   engine,
		work:     make(map[common.Hash]*Work),
		hashrate: make(map[common.Hash]hashrate),
	}
}
```

Start and stop

```go
func (a *RemoteAgent) Start() {
	if !atomic.CompareAndSwapInt32(&a.running, 0, 1) {
		return
	}
	a.quitCh = make(chan struct{})
	a.workCh = make(chan *Work, 1)
	go a.loop(a.workCh, a.quitCh)
}

func (a *RemoteAgent) Stop() {
	if !atomic.CompareAndSwapInt32(&a.running, 1, 0) {
		return
	}
	close(a.quitCh)
	close(a.workCh)
}
```

Get the input and output channels, this is the same as agent.go.

```go
func (a *RemoteAgent) Work() chan<- *Work {
	return a.workCh
}

func (a *RemoteAgent) SetReturnCh(returnCh chan<- *Result) {
	a.returnCh = returnCh
}
```

The loop method is similar to the work done in agent.go. When the task is received, it is stored in the currentWork field. If you haven't completed a job in 84 seconds, then delete the job. If you haven't received the hashrate report for 10 seconds, delete the trace/.

```go
// loop monitors mining events on the work and quit channels, updating the internal
// state of the rmeote miner until a termination is requested.
//
// Note, the reason the work and quit channels are passed as parameters is because
// RemoteAgent.Start() constantly recreates these channels, so the loop code cannot
// assume data stability in these member fields.
func (a *RemoteAgent) loop(workCh chan *Work, quitCh chan struct{}) {
	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-quitCh:
			return
		case work := <-workCh:
			a.mu.Lock()
			a.currentWork = work
			a.mu.Unlock()
		case <-ticker.C:
			// cleanup
			a.mu.Lock()
			for hash, work := range a.work {
				if time.Since(work.createdAt) > 7*(12*time.Second) {
					delete(a.work, hash)
				}
			}
			a.mu.Unlock()

			a.hashrateMu.Lock()
			for id, hashrate := range a.hashrate {
				if time.Since(hashrate.ping) > 10*time.Second {
					delete(a.hashrate, id)
				}
			}
			a.hashrateMu.Unlock()
		}
	}
}
```

GetWork, this method is called by a remote miner to get the current mining task.

```go
func (a *RemoteAgent) GetWork() ([3]string, error) {
	a.mu.Lock()
	defer a.mu.Unlock()

	var res [3]string

	if a.currentWork != nil {
		block := a.currentWork.Block

		res[0] = block.HashNoNonce().Hex()
		seedHash := ethash.SeedHash(block.NumberU64())
		res[1] = common.BytesToHash(seedHash).Hex()
		// Calculate the "target" to be returned to the external miner
		n := big.NewInt(1)
		n.Lsh(n, 255)
		n.Div(n, block.Difficulty())
		n.Lsh(n, 1)
		res[2] = common.BytesToHash(n.Bytes()).Hex()

		a.work[block.HashNoNonce()] = a.currentWork
		return res, nil
	}
	return res, errors.New("No work available yet, don't panic.")
}
```

SubmitWork, the remote miners call this method to submit the results of the mining. Submit the result to returnCh after verifying the result

```go
// SubmitWork tries to inject a pow solution into the remote agent, returning
// whether the solution was accepted or not (not can be both a bad pow as well as
// any other error, like no work pending).
func (a *RemoteAgent) SubmitWork(nonce types.BlockNonce, mixDigest, hash common.Hash) bool {
	a.mu.Lock()
	defer a.mu.Unlock()

	// Make sure the work submitted is present
	work := a.work[hash]
	if work == nil {
		log.Info("Work submitted but none pending", "hash", hash)
		return false
	}
	// Make sure the Engine solutions is indeed valid
	result := work.Block.Header()
	result.Nonce = nonce
	result.MixDigest = mixDigest

	if err := a.engine.VerifySeal(a.chain, result); err != nil {
		log.Warn("Invalid proof-of-work submitted", "hash", hash, "err", err)
		return false
	}
	block := work.Block.WithSeal(result)

	// Solutions seems to be valid, return to the miner and notify acceptance
	a.returnCh <- &Result{work, block}
	delete(a.work, hash)

	return true
}
```

SubmitHashrate, submit hash power

```go
func (a *RemoteAgent) SubmitHashrate(id common.Hash, rate uint64) {
	a.hashrateMu.Lock()
	defer a.hashrateMu.Unlock()

	a.hashrate[id] = hashrate{time.Now(), rate}
}
```

## unconfirmed

Unconfirmed is a data structure used to track the user's local mining information, such as dug out a block, then wait for enough subsequent block confirmation (5), then check whether the local mining block is included in the specification. Inside the blockchain.

Data structure

```go
// headerRetriever is used by the unconfirmed block set to verify whether a previously
// mined block is part of the canonical chain or not.
type headerRetriever interface {
	// GetHeaderByNumber retrieves the canonical header associated with a block number.
	GetHeaderByNumber(number uint64) *types.Header
}

// unconfirmedBlock is a small collection of metadata about a locally mined block
// that is placed into a unconfirmed set for canonical chain inclusion tracking.
type unconfirmedBlock struct {
	index uint64
	hash  common.Hash
}

// unconfirmedBlocks implements a data structure to maintain locally mined blocks
// have have not yet reached enough maturity to guarantee chain inclusion. It is
// used by the miner to provide logs to the user when a previously mined block
// has a high enough guarantee to not be reorged out of te canonical chain.
type unconfirmedBlocks struct {
	chain  headerRetriever // Blockchain to verify canonical status through
	depth  uint            // Depth after which to discard previous blocks
	blocks *ring.Ring      // Block infos to allow canonical chain cross checks
	lock   sync.RWMutex    // Protects the fields from concurrent access
}

// newUnconfirmedBlocks returns new data structure to track currently unconfirmed blocks.
func newUnconfirmedBlocks(chain headerRetriever, depth uint) *unconfirmedBlocks {
	return &unconfirmedBlocks{
		chain: chain,
		depth: depth,
	}
}
```

Insert the tracking block, when the miner digs into a block, index is the height of the block, and hash is the hash value of the block.

```go
// Insert adds a new block to the set of unconfirmed ones.
func (set *unconfirmedBlocks) Insert(index uint64, hash common.Hash) {
	// If a new block was mined locally, shift out any old enough blocks
	set.Shift(index)

	// Create the new item as its own ring
	item := ring.New(1)
	item.Value = &unconfirmedBlock{
		index: index,
		hash:  hash,
	}
	// Set as the initial ring or append to the end
	set.lock.Lock()
	defer set.lock.Unlock()

	if set.blocks == nil {
		set.blocks = item
	} else {
		// Move to the last element of the loop queue to insert the item
		set.blocks.Move(-1).Link(item)
	}
	// Display a log for the user to notify of a new mined block unconfirmed
	log.Info("🔨 mined potential block", "number", index, "hash", hash)
}
```

The Shift method removes blocks whose index exceeds the passed index-depth and checks if they are in the canonical blockchain.

```go
// Shift drops all unconfirmed blocks from the set which exceed the unconfirmed sets depth
// allowance, checking them against the canonical chain for inclusion or staleness
// report.
func (set *unconfirmedBlocks) Shift(height uint64) {
	set.lock.Lock()
	defer set.lock.Unlock()

	for set.blocks != nil {
		// Retrieve the next unconfirmed block and abort if too fresh
		// Because the blocks in blocks are arranged in order. At the very beginning, it is definitely the oldest block.
		// So only need to check the last block at a time, if it is finished, it will be removed from the loop queue.
		next := set.blocks.Value.(*unconfirmedBlock)
		if next.index+uint64(set.depth) > height { // If it is old enough.
			break
		}
		// Block seems to exceed depth allowance, check for canonical status
		// Query the block header of that block height
		header := set.chain.GetHeaderByNumber(next.index)
		switch {
		case header == nil:
			log.Warn("Failed to retrieve header of mined block", "number", next.index, "hash", next.hash)
		case header.Hash() == next.hash: // If the block header is equal to ourselves,
			log.Info("🔗 block reached canonical chain", "number", next.index, "hash", next.hash)
		default: // Otherwise we are above the side chain.
			log.Info("⑂ block  became a side fork", "number", next.index, "hash", next.hash)
		}
		// Drop the block out of the ring
		// Delete from the loop queue
		if set.blocks.Value == set.blocks.Next().Value {
			// If the current value is equal to our own, indicating that only the loop queue has only one element, then the setting is not nil
			set.blocks = nil
		} else {
			// Otherwise move to the end, then delete one and move to the front.
			set.blocks = set.blocks.Move(-1)
			set.blocks.Unlink(1)
			set.blocks = set.blocks.Move(1)
		}
	}
}
```

## worker.go

The worker contains a lot of agents, including the agent and remote_agent mentioned earlier. The worker is also responsible for building blocks and objects. At the same time, the task is provided to the agent.

Data structure:

Agent interface

```go
// Agent can register themself with the worker
type Agent interface {
	Work() chan<- *Work
	SetReturnCh(chan<- *Result)
	Stop()
	Start()
	GetHashRate() int64
}
```

Work structure, Work stores the worker's current environment and holds all temporary status information.

```go
// Work is the workers current environment and holds
// all of the current state information
type Work struct {
	config *params.ChainConfig
	signer types.Signer			// Signer

	state     *state.StateDB // apply state changes here
	ancestors *set.Set       // ancestor set (used for checking uncle parent validity)
	family    *set.Set       // family set (used for checking uncle invalidity)
	uncles    *set.Set       // uncle set
	tcount    int            // tx count in cycle

	Block *types.Block // the new block

	header   *types.Header			// Block head
	txs      []*types.Transaction   // transaction
	receipts []*types.Receipt  		// receipt

	createdAt time.Time 			// creation time
}

type Result struct {
	Work  *Work
	Block *types.Block
}
```

worker

```go
// worker is the main object which takes care of applying messages to the new state
type worker struct {
	config *params.ChainConfig
	engine consensus.Engine
	mu sync.Mutex
	// update loop
	mux          *event.TypeMux
	txCh         chan core.TxPreEvent		// Channel used to accept transactions in txPool
	txSub        event.Subscription			// Subscriber for accepting transactions in txPool
	chainHeadCh  chan core.ChainHeadEvent	// Channel used to accept the block header
	chainHeadSub event.Subscription
	chainSideCh  chan core.ChainSideEvent	// Channel used to accept a blockchain removed from the canonical blockchain
	chainSideSub event.Subscription
	wg           sync.WaitGroup

	agents map[Agent]struct{}				// All agents
	recv   chan *Result						// Agent will send the result to this channel

	eth     Backend							// Eth consensus
	chain   *core.BlockChain
	proc    core.Validator					// blockchain validator
	chainDb ethdb.Database					// blockchain database

	coinbase common.Address					// Miner's address
	extra    []byte							//

	snapshotMu    sync.RWMutex				// Snapshot RWMutex (snapshot read and write lock)
	snapshotBlock *types.Block
	snapshotState *state.StateDB

	currentMu sync.Mutex
	current   *Work

	uncleMu        sync.Mutex
	possibleUncles map[common.Hash]*types.Block

	unconfirmed *unconfirmedBlocks // set of locally mined blocks pending canonicalness confirmations

	// atomic status counters
	mining int32
	atWork int32
}
```

structure

```go
func newWorker(config *params.ChainConfig, engine consensus.Engine, coinbase common.Address, eth Backend, mux *event.TypeMux) *worker {
	worker := &worker{
		config:         config,
		engine:         engine,
		eth:            eth,
		mux:            mux,
		txCh:           make(chan core.TxPreEvent, txChanSize), // 4096
		chainHeadCh:    make(chan core.ChainHeadEvent, chainHeadChanSize), // 10
		chainSideCh:    make(chan core.ChainSideEvent, chainSideChanSize), // 10
		chainDb:        eth.ChainDb(),
		recv:           make(chan *Result, resultQueueSize), // 10
		chain:          eth.BlockChain(),
		proc:           eth.BlockChain().Validator(),
		possibleUncles: make(map[common.Hash]*types.Block),
		coinbase:       coinbase,
		agents:         make(map[Agent]struct{}),
		unconfirmed:    newUnconfirmedBlocks(eth.BlockChain(), miningLogAtDepth),
	}
	// Subscribe TxPreEvent for tx pool
	worker.txSub = eth.TxPool().SubscribeTxPreEvent(worker.txCh)
	// Subscribe events for blockchain
	worker.chainHeadSub = eth.BlockChain().SubscribeChainHeadEvent(worker.chainHeadCh)
	worker.chainSideSub = eth.BlockChain().SubscribeChainSideEvent(worker.chainSideCh)
	go worker.update()

	go worker.wait()
	worker.commitNewWork()

	return worker
}
```

update

```go
func (self *worker) update() {
	defer self.txSub.Unsubscribe()
	defer self.chainHeadSub.Unsubscribe()
	defer self.chainSideSub.Unsubscribe()

	for {
		// A real event arrived, process interesting content
		select {
		// Handle ChainHeadEvent When receiving the information of a block header, the mining service is started immediately.
		case <-self.chainHeadCh:
			self.commitNewWork()

		// Handle ChainSideEvent Receive blocks that are not in the canonical blockchain and join the potential uncle collection
		case ev := <-self.chainSideCh:
			self.uncleMu.Lock()
			self.possibleUncles[ev.Block.Hash()] = ev.Block
			self.uncleMu.Unlock()

		// Handle TxPreEvent When receiving the transaction information in txPool.
		case ev := <-self.txCh:
			// Apply transaction to the pending state if we're not mining
			// If there is currently no mining, then apply the transaction to the current state so that the mining task can be started immediately.
			if atomic.LoadInt32(&self.mining) == 0 {
				self.currentMu.Lock()
				acc, _ := types.Sender(self.current.signer, ev.Tx)
				txs := map[common.Address]types.Transactions{acc: {ev.Tx}}
				txset := types.NewTransactionsByPriceAndNonce(self.current.signer, txs)

				self.current.commitTransactions(self.mux, txset, self.chain, self.coinbase)
				self.currentMu.Unlock()
			}

		// System stopped
		case <-self.txSub.Err():
			return
		case <-self.chainHeadSub.Err():
			return
		case <-self.chainSideSub.Err():
			return
		}
	}
}
```

commitNewWork submits a new task

```go
func (self *worker) commitNewWork() {
	self.mu.Lock()
	defer self.mu.Unlock()
	self.uncleMu.Lock()
	defer self.uncleMu.Unlock()
	self.currentMu.Lock()
	defer self.currentMu.Unlock()

	tstart := time.Now()
	parent := self.chain.CurrentBlock()

	tstamp := tstart.Unix()
	if parent.Time().Cmp(new(big.Int).SetInt64(tstamp)) >= 0 { // can't appear less than the time of the parent
		tstamp = parent.Time().Int64() + 1
	}
	// this will ensure we're not going off too far in the future
	// Our time should not be too far away from the present time, then wait for a while,
	// I feel that this function is completely for testing. If it is a real mining program, it should not wait.
	if now := time.Now().Unix(); tstamp > now+1 {
		wait := time.Duration(tstamp-now) * time.Second
		log.Info("Mining too far in the future", "wait", common.PrettyDuration(wait))
		time.Sleep(wait)
	}

	num := parent.Number()
	header := &types.Header{
		ParentHash: parent.Hash(),
		Number:     num.Add(num, common.Big1),
		GasLimit:   core.CalcGasLimit(parent),
		GasUsed:    new(big.Int),
		Extra:      self.extra,
		Time:       big.NewInt(tstamp),
	}
	// Only set the coinbase if we are mining (avoid spurious block rewards)
	if atomic.LoadInt32(&self.mining) == 1 {
		header.Coinbase = self.coinbase
	}
	if err := self.engine.Prepare(self.chain, header); err != nil {
		log.Error("Failed to prepare header for mining", "err", err)
		return
	}
	// If we are care about TheDAO hard-fork check whether to override the extra-data or not
	if daoBlock := self.config.DAOForkBlock; daoBlock != nil {
		// Check whether the block is among the fork extra-override range
		// Check if the block is within the range of DAO hard fork  [daoblock,daoblock+limit]
		limit := new(big.Int).Add(daoBlock, params.DAOForkExtraRange)
		if header.Number.Cmp(daoBlock) >= 0 && header.Number.Cmp(limit) < 0 {
			// Depending whether we support or oppose the fork, override differently
			if self.config.DAOForkSupport { // If we support DAO then set the reserved extra data
				header.Extra = common.CopyBytes(params.DAOForkBlockExtra)
			} else if bytes.Equal(header.Extra, params.DAOForkBlockExtra) {
				header.Extra = []byte{} // If miner opposes, don't let it use the reserved extra-data
			}
		}
	}
	// Could potentially happen if starting to mine in an odd state.
	err := self.makeCurrent(parent, header) // Use the new block header to set the current state
	if err != nil {
		log.Error("Failed to create mining context", "err", err)
		return
	}
	// Create the current work task and check any fork transitions needed
	work := self.current
	if self.config.DAOForkSupport && self.config.DAOForkBlock != nil && self.config.DAOForkBlock.Cmp(header.Number) == 0 {
		misc.ApplyDAOHardFork(work.state)  // Transfer funds from the DAO to the designated account.
	}
	pending, err := self.eth.TxPool().Pending() // Obstructed funds
	if err != nil {
		log.Error("Failed to fetch pending transactions", "err", err)
		return
	}
	// Create a transaction. Follow-up of this method
	txs := types.NewTransactionsByPriceAndNonce(self.current.signer, pending)
	// Submit a transaction
	work.commitTransactions(self.mux, txs, self.chain, self.coinbase)

	// compute uncles for the new block.
	var (
		uncles    []*types.Header
		badUncles []common.Hash
	)
	for hash, uncle := range self.possibleUncles {
		if len(uncles) == 2 {
			break
		}
		if err := self.commitUncle(work, uncle.Header()); err != nil {
			log.Trace("Bad uncle found and will be removed", "hash", hash)
			log.Trace(fmt.Sprint(uncle))

			badUncles = append(badUncles, hash)
		} else {
			log.Debug("Committing new uncle to block", "hash", hash)
			uncles = append(uncles, uncle.Header())
		}
	}
	for _, hash := range badUncles {
		delete(self.possibleUncles, hash)
	}
	// Create the new block to seal with the consensus engine
	// Use the given state to create a new block, Finalize will perform block rewards, etc.
	if work.Block, err = self.engine.Finalize(self.chain, header, work.state, work.txs, uncles, work.receipts); err != nil {
		log.Error("Failed to finalize block for sealing", "err", err)
		return
	}
	// We only care about logging if we're actually mining.
	//
	if atomic.LoadInt32(&self.mining) == 1 {
		log.Info("Commit new mining work", "number", work.Block.Number(), "txs", work.tcount, "uncles", len(uncles), "elapsed", common.PrettyDuration(time.Since(tstart)))
		self.unconfirmed.Shift(work.Block.NumberU64() - 1)
	}
	self.push(work)
}
```

Push method, if we are not mining, then return directly, otherwise give the task to each agent

```go
// push sends a new work task to currently live miner agents.
func (self *worker) push(work *Work) {
	if atomic.LoadInt32(&self.mining) != 1 {
		return
	}
	for agent := range self.agents {
		atomic.AddInt32(&self.atWork, 1)
		if ch := agent.Work(); ch != nil {
			ch <- work
		}
	}
}
```

makeCurrent, creating a new environment without the current cycle.

```go
// makeCurrent creates a new environment for the current cycle.
//
func (self *worker) makeCurrent(parent *types.Block, header *types.Header) error {
	state, err := self.chain.StateAt(parent.Root())
	if err != nil {
		return err
	}
	work := &Work{
		config:    self.config,
		signer:    types.NewEIP155Signer(self.config.ChainId),
		state:     state,
		ancestors: set.New(),
		family:    set.New(),
		uncles:    set.New(),
		header:    header,
		createdAt: time.Now(),
	}

	// when 08 is processed ancestors contain 07 (quick block)
	for _, ancestor := range self.chain.GetBlocksFromHash(parent.Hash(), 7) {
		for _, uncle := range ancestor.Uncles() {
			work.family.Add(uncle.Hash())
		}
		work.family.Add(ancestor.Hash())
		work.ancestors.Add(ancestor.Hash())
	}

	// Keep track of transactions which return errors so they can be removed
	work.tcount = 0
	self.current = work
	return nil
}
```

commitTransactions

```go
func (env *Work) commitTransactions(mux *event.TypeMux, txs *types.TransactionsByPriceAndNonce, bc *core.BlockChain, coinbase common.Address) {
	// Initialize the total gasPool to env.header.GasLimit because it is a new block in the package
	if env.gasPool == nil {
		env.gasPool = new(core.GasPool).AddGas(env.header.GasLimit)
	}

	var coalescedLogs []*types.Log

	for {
		// If we don't have enough gas for any further transactions then we're done
		// Exit the packaged transaction if all Gas consumption in the current block has been used up
		if env.gasPool.Gas() < params.TxGas {
			log.Trace("Not enough gas for further transactions", "have", env.gasPool, "want", params.TxGas)
			break
		}

		// Retrieve the next transaction and abort if all done
		tx := txs.Peek()
		if tx == nil {
			break
		}
		// Error may be ignored here. The error has already been checked
		// during transaction acceptance is the transaction pool.
		//
		// We use the eip155 signer regardless of the current hf.
		from, _ := types.Sender(env.signer, tx)
		// Check whether the tx is replay protected. If we're not in the EIP155 hf
		// phase, start ignoring the sender until we do.
		// Please refer to https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md
		// After the DAO event, Ethereum splits into ETH and ETC, because the things on the two chains are the same, so at ETC
		// The transaction that occurred above can be retrieved on ETH and vice versa. So Vitalik proposed EIP155 to avoid this situation.
		if tx.Protected() && !env.config.IsEIP155(env.header.Number) {
			log.Trace("Ignoring reply protected transaction", "hash", tx.Hash(), "eip155", env.config.EIP155Block)

			txs.Pop()
			continue
		}
		// Start executing the transaction
		env.state.Prepare(tx.Hash(), common.Hash{}, env.tcount)
		// execute the transaction
		err, logs := env.commitTransaction(tx, bc, coinbase, gp)
		switch err {
		case core.ErrGasLimitReached:
			// Pop the current out-of-gas transaction without shifting in the next from the account
			log.Trace("Gas limit exceeded for current block", "sender", from)
			txs.Pop()

		case core.ErrNonceTooLow:
			// New head notification data race between the transaction pool and miner, shift
			log.Trace("Skipping transaction with low nonce", "sender", from, "nonce", tx.Nonce())
			txs.Shift()

		case core.ErrNonceTooHigh:
			// Reorg notification data race between the transaction pool and miner, skip account =
			log.Trace("Skipping account with hight nonce", "sender", from, "nonce", tx.Nonce())
			txs.Pop()

		case nil:
			// Everything ok, collect the logs and shift in the next transaction from the same account
			coalescedLogs = append(coalescedLogs, logs...)
			env.tcount++
			txs.Shift()

		default:
			// Strange error, discard the transaction and get the next in line (note, the
			// nonce-too-high clause will prevent us from executing in vain).
			log.Debug("Transaction failed, account skipped", "hash", tx.Hash(), "err", err)
			txs.Shift()
		}
	}

	if len(coalescedLogs) > 0 || env.tcount > 0 {
		// make a copy, the state caches the logs and these logs get "upgraded" from pending to mined
		// logs by filling in the block hash when the block was mined by the local miner. This can
		// cause a race condition if a log was "upgraded" before the PendingLogsEvent is processed.
		cpy := make([]*types.Log, len(coalescedLogs))
		for i, l := range coalescedLogs {
			cpy[i] = new(types.Log)
			*cpy[i] = *l
		}
		go func(logs []*types.Log, tcount int) {
			if len(logs) > 0 {
				mux.Post(core.PendingLogsEvent{Logs: logs})
			}
			if tcount > 0 {
				mux.Post(core.PendingStateEvent{})
			}
		}(cpy, env.tcount)
	}
}
```

commitTransaction execute ApplyTransaction

```go
func (env *Work) commitTransaction(tx *types.Transaction, bc *core.BlockChain, coinbase common.Address, gp *core.GasPool) (error, []*types.Log) {
	snap := env.state.Snapshot()

	receipt, _, err := core.ApplyTransaction(env.config, bc, &coinbase, gp, env.state, env.header, tx, env.header.GasUsed, vm.Config{})
	if err != nil {
		env.state.RevertToSnapshot(snap)
		return err, nil
	}
	env.txs = append(env.txs, tx)
	env.receipts = append(env.receipts, receipt)

	return nil, receipt.Logs
}
```

The wait function is used to accept the results of the mining and then write to the local blockchain and broadcast it through the eth protocol.

```go
func (self *worker) wait() {
	for {
		mustCommitNewWork := true
		for result := range self.recv {
			atomic.AddInt32(&self.atWork, -1)

			if result == nil {
				continue
			}
			block := result.Block
			work := result.Work

			// Update the block hash in all logs since it is now available and not when the
			// receipt/log of individual transactions were created.
			for _, r := range work.receipts {
				for _, l := range r.Logs {
					l.BlockHash = block.Hash()
				}
			}
			for _, log := range work.state.Logs() {
				log.BlockHash = block.Hash()
			}
			stat, err := self.chain.WriteBlockAndState(block, work.receipts, work.state)
			if err != nil {
				log.Error("Failed writing block to chain", "err", err)
				continue
			}
			// check if canon block and write transactions
			if stat == core.CanonStatTy { // Description Blockchain that has been inserted into the specification
				// implicit by posting ChainHeadEvent
				// Because in this state, will send ChainHeadEvent, will trigger the code inside the update, this part of the code will commitNewWork, so there is no need to commit here.
				mustCommitNewWork = false
			}
			// Broadcast the block and announce chain insertion event
			self.mux.Post(core.NewMinedBlockEvent{Block: block})
			var (
				events []interface{}
				logs   = work.state.Logs()
			)
			events = append(events, core.ChainEvent{Block: block, Hash: block.Hash(), Logs: logs})
			if stat == core.CanonStatTy {
				events = append(events, core.ChainHeadEvent{Block: block})
			}
			self.chain.PostChainEvents(events, logs)

			// Insert the block into the set of pending ones to wait for confirmations
			self.unconfirmed.Insert(block.NumberU64(), block.Hash())

			if mustCommitNewWork {
				self.commitNewWork()
			}
		}
	}
}
```

## miner

Miner is used to manage workers, subscribe to external events, and control the start and stop of workers.

Data structure

```go
// Backend wraps all methods required for mining.
type Backend interface {
	AccountManager() *accounts.Manager
	BlockChain() *core.BlockChain
	TxPool() *core.TxPool
	ChainDb() ethdb.Database
}

// Miner creates blocks and searches for proof-of-work values.
type Miner struct {
	mux *event.TypeMux

	worker *worker

	coinbase common.Address
	mining   int32
	eth      Backend
	engine   consensus.Engine

	canStart    int32 // can start indicates whether we can start the mining operation
	shouldStart int32 // should start indicates whether we should start after sync
}
```

Constructed, created a CPU agent started miner update goroutine

```go
func New(eth Backend, config *params.ChainConfig, mux *event.TypeMux, engine consensus.Engine) *Miner {
	miner := &Miner{
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
```

Update subscribes to the downloader event. Note that this goroutine is a one-time loop. Once you receive a downloader's downloader.DoneEvent or downloader.FailedEvent event, it will set canStart to 1. and exit the loop. This is to avoid hackers. DOS attack, keep you in an abnormal state

```go
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
			atomic.StoreInt32(&self.canStart, 0)
			if self.Mining() {
				self.Stop()
				atomic.StoreInt32(&self.shouldStart, 1)
				log.Info("Mining aborted due to sync")
			}
		case downloader.DoneEvent, downloader.FailedEvent:
			shouldStart := atomic.LoadInt32(&self.shouldStart) == 1

			atomic.StoreInt32(&self.canStart, 1)
			atomic.StoreInt32(&self.shouldStart, 0)
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
```

Start

```go
func (self *Miner) Start(coinbase common.Address) {
	atomic.StoreInt32(&self.shouldStart, 1)  // shouldStart should be started
	self.worker.setEtherbase(coinbase)
	self.coinbase = coinbase

	if atomic.LoadInt32(&self.canStart) == 0 {  // canStart can be started,
		log.Info("Network syncing, will start miner afterwards")
		return
	}
	atomic.StoreInt32(&self.mining, 1)

	log.Info("Starting mining operation")
	self.worker.start()  // Start the worker to start mining
	self.worker.commitNewWork()  // submit a new work
}
```
