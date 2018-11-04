##eth POW analysis

### Consensus engine description

In the CPU mining part, the CpuAgent mine function calls the self.engine.Seal function when performing the mining operation. The engine here is the consensus engine. Seal is one of the most important interfaces. It implements the search for nonce values ​​and the calculation of hashes. And this function is an important function that guarantees consensus and cannot be forged. In the PoW consensus algorithm, the Seal function implements proof of work. This part of the source code is under consensus/ethhash.

### Consensus engine interface

```go
type Engine interface {
	// Get the block digger, ie coinbase
	Author(header *types.Header) (common.Address, error)


	// VerifyHeader is used to check the block header and check it by consensus rules. The verification block can be used here. Ketong passes the VerifySeal method.
	VerifyHeader(chain ChainReader, header *types.Header, seal bool) error


	// VerifyHeaders is similar to VerifyHeader, and this is used to batch checkpoints. This method returns an exit signal
	// Used to terminate the operation for asynchronous verification.
	VerifyHeaders(chain ChainReader, headers []*types.Header, seals []bool) (chan<- struct{}, <-chan error)

	// VerifyUncles Rules for verifying a bad block to conform to the consensus engine
	VerifyUncles(chain ChainReader, block *types.Block) error

	// VerifySeal Check the block header according to the rules of the consensus algorithm
	VerifySeal(chain ChainReader, header *types.Header) error

	// Prepare The consensus field used to initialize the block header is based on the consensus engine. These changes are all implemented inline.
	Prepare(chain ChainReader, header *types.Header) error

	// Finalize Complete all state modifications and eventually assemble them into blocks.
	// The block header and state database can be updated to conform to the consensus rules at the time of final validation.
	Finalize(chain ChainReader, header *types.Header, state *state.StateDB, txs []*types.Transaction,
		uncles []*types.Header, receipts []*types.Receipt) (*types.Block, error)

	// Seal Production of a new block based on the input block
	Seal(chain ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error)

	// CalcDifficulty It is the difficulty adjustment algorithm that returns the difficulty value of the new block.
	CalcDifficulty(chain ChainReader, time uint64, parent *types.Header) *big.Int

	// APIs Return the RPC provided by the consensus engine APIs
	APIs(chain ChainReader) []rpc.API
}
```

### ethhash implementation

#### ethhash structure

```go
type Ethash struct {
	config Config

	caches   *lru // In memory caches to avoid regenerating too often
	datasets *lru // In memory datasets to avoid regenerating too often

	// Mining related fields
	rand     *rand.Rand    // Properly seeded random source for nonces
	threads  int           // Number of threads to mine on if mining
	// channel Used to update mining notices
	update   chan struct{} // Notification channel to update mining parameters
	hashrate metrics.Meter // Meter tracking the average hashrate

	// Test network related parameters
	// The fields below are hooks for testing
	shared    *Ethash       // Shared PoW verifier to avoid cache regeneration
	fakeFail  uint64        // Block number which fails PoW check even in fake mode
	fakeDelay time.Duration // Time delay to sleep for before returning from verify

	lock sync.Mutex // Ensures thread safety for the in-memory caches and mining fields
}
```

Ethhash is a concrete implementation of PoW. Since there are a large number of datasets to be used, there are two pointers to lru. And control the number of mining threads through threads. And in test mode or fake mode, simple and fast processing, so that it can get results quickly.

The Athor method obtained the miner's address from which the block was dug.

```go
func (ethash *Ethash) Author(header *types.Header) (common.Address, error) {
	return header.Coinbase, nil
}
```

VerifyHeader is used to verify that the block header information conforms to the ethash consensus engine rules.

```go
// VerifyHeader checks whether a header conforms to the consensus rules of the
// stock Ethereum ethash engine.
func (ethash *Ethash) VerifyHeader(chain consensus.ChainReader, header *types.Header, seal bool) error {
	// When in ModeFullFake mode, any header information is accepted
	if ethash.config.PowMode == ModeFullFake {
		return nil
	}
	// If the header is known, it does not need to be verified and returns directly.
	number := header.Number.Uint64()
	if chain.GetHeader(header.Hash(), number) != nil {
		return nil
	}
	parent := chain.GetHeader(header.ParentHash, number-1)
	if parent == nil {  // Failed to get parent node
		return consensus.ErrUnknownAncestor
	}
	// Further header verification
	return ethash.verifyHeader(chain, header, parent, false, seal)
}
```

Then look at the implementation of verifyHeader,

```go
func (ethash *Ethash) verifyHeader(chain consensus.ChainReader, header, parent *types.Header, uncle bool, seal bool) error {
	// Ensure that the extra data segment has a reasonable length
	if uint64(len(header.Extra)) > params.MaximumExtraDataSize {
		return fmt.Errorf("extra-data too long: %d > %d", len(header.Extra), params.MaximumExtraDataSize)
	}
	// Check timestamp
	if uncle {
		if header.Time.Cmp(math.MaxBig256) > 0 {
			return errLargeBlockTime
		}
	} else {
		if header.Time.Cmp(big.NewInt(time.Now().Add(allowedFutureBlockTime).Unix())) > 0 {
			return consensus.ErrFutureBlock
		}
	}
	if header.Time.Cmp(parent.Time) <= 0 {
		return errZeroBlockTime
	}
	// The difficulty of the block is checked based on the timestamp and the difficulty of the parent block.
	expected := ethash.CalcDifficulty(chain, header.Time.Uint64(), parent)

	if expected.Cmp(header.Difficulty) != 0 {
		return fmt.Errorf("invalid difficulty: have %v, want %v", header.Difficulty, expected)
	}
	// check gas limit <= 2^63-1
	cap := uint64(0x7fffffffffffffff)
	if header.GasLimit > cap {
		return fmt.Errorf("invalid gasLimit: have %v, max %v", header.GasLimit, cap)
	}
	// check gasUsed <= gasLimit
	if header.GasUsed > header.GasLimit {
		return fmt.Errorf("invalid gasUsed: have %d, gasLimit %d", header.GasUsed, header.GasLimit)
	}

	// gas limit Is it within the allowable range?
	diff := int64(parent.GasLimit) - int64(header.GasLimit)
	if diff < 0 {
		diff *= -1
	}
	limit := parent.GasLimit / params.GasLimitBoundDivisor

	if uint64(diff) >= limit || header.GasLimit < params.MinGasLimit {
		return fmt.Errorf("invalid gas limit: have %d, want %d += %d", header.GasLimit, parent.GasLimit, limit)
	}
	// The check block number should be the parent block block number +1
	if diff := new(big.Int).Sub(header.Number, parent.Number); diff.Cmp(big.NewInt(1)) != 0 {
		return consensus.ErrInvalidNumber
	}
	// Verify that a particular block meets the requirements
	if seal {
		if err := ethash.VerifySeal(chain, header); err != nil {
			return err
		}
	}
	// If all checks pass, verify the special field of the hard fork.
	if err := misc.VerifyDAOHeaderExtraData(chain.Config(), header); err != nil {
		return err
	}
	if err := misc.VerifyForkHashes(chain.Config(), header, uncle); err != nil {
		return err
	}
	return nil
}
```

Ethash calculates the difficulty of the next block through the CalcDifficulty function, and creates different difficulty calculation methods for the difficulty of different stages.

```go
func (ethash *Ethash) CalcDifficulty(chain consensus.ChainReader, time uint64, parent *types.Header) *big.Int {
	return CalcDifficulty(chain.Config(), time, parent)
}

func CalcDifficulty(config *params.ChainConfig, time uint64, parent *types.Header) *big.Int {
	next := new(big.Int).Add(parent.Number, big1)
	switch {
	case config.IsByzantium(next):
		return calcDifficultyByzantium(time, parent)
	case config.IsHomestead(next):
		return calcDifficultyHomestead(time, parent)
	default:
		return calcDifficultyFrontier(time, parent)
	}
}
```

VerifyHeaders is similar to VerifyHeader except that VerifyHeaders performs bulk check operations. Create multiple goroutines to perform validation operations, and then create a goroutine for assignment control task assignment and result acquisition. Finally return a result channel

```go
func (ethash *Ethash) VerifyHeaders(chain consensus.ChainReader, headers []*types.Header, seals []bool) (chan<- struct{}, <-chan error) {
	// If we're running a full engine faking, accept any input as valid
	if ethash.config.PowMode == ModeFullFake || len(headers) == 0 {
		abort, results := make(chan struct{}), make(chan error, len(headers))
		for i := 0; i < len(headers); i++ {
			results <- nil
		}
		return abort, results
	}

	// Spawn as many workers as allowed threads
	workers := runtime.GOMAXPROCS(0)
	if len(headers) < workers {
		workers = len(headers)
	}

	// Create a task channel and spawn the verifiers
	var (
		inputs = make(chan int)
		done   = make(chan int, workers)
		errors = make([]error, len(headers))
		abort  = make(chan struct{})
	)
	// Generate workers goroutine for check head
	for i := 0; i < workers; i++ {
		go func() {
			for index := range inputs {
				errors[index] = ethash.verifyHeaderWorker(chain, headers, seals, index)
				done <- index
			}
		}()
	}

	errorsOut := make(chan error, len(headers))
	// goroutine Used to send tasks to workers goroutine
	go func() {
		defer close(inputs)
		var (
			in, out = 0, 0
			checked = make([]bool, len(headers))
			inputs  = inputs
		)
		for {
			select {
			case inputs <- in:
				if in++; in == len(headers) {
					// Reached end of headers. Stop sending to workers.
					inputs = nil
				}
			// Statistics and send an error message to errorsOut
			case index := <-done:
				for checked[index] = true; checked[out]; out++ {
					errorsOut <- errors[out]
					if out == len(headers)-1 {
						return
					}
				}
			case <-abort:
				return
			}
		}
	}()
	return abort, errorsOut
}
```

VerifyHeaders uses verifyHeaderWorker when checking a single block header. After the function gets the parent block, it calls verifyHeader to check it.

```go
func (ethash *Ethash) verifyHeaderWorker(chain consensus.ChainReader, headers []*types.Header, seals []bool, index int) error {
	var parent *types.Header
	if index == 0 {
		parent = chain.GetHeader(headers[0].ParentHash, headers[0].Number.Uint64()-1)
	} else if headers[index-1].Hash() == headers[index].ParentHash {
		parent = headers[index-1]
	}
	if parent == nil {
		return consensus.ErrUnknownAncestor
	}
	if chain.GetHeader(headers[index].Hash(), headers[index].Number.Uint64()) != nil {
		return nil // known block
	}
	return ethash.verifyHeader(chain, headers[index], parent, false, seals[index])
}
```

VerifyUncles is used for verification of the unblock. Similar to the check block header, the unchecked block check returns directly in the ModeFullFake mode. Get all the uncle blocks, then traverse the checksum, the checksum will terminate, or the checksum will return.

```go
func (ethash *Ethash) VerifyUncles(chain consensus.ChainReader, block *types.Block) error {
	// If we're running a full engine faking, accept any input as valid
	if ethash.config.PowMode == ModeFullFake {
		return nil
	}
	// Verify that there are at most 2 uncles included in this block
	if len(block.Uncles()) > maxUncles {
		return errTooManyUncles
	}
	// Collecting uncles and their ancestors
	uncles, ancestors := set.New(), make(map[common.Hash]*types.Header)

	number, parent := block.NumberU64()-1, block.ParentHash()
	for i := 0; i < 7; i++ {
		ancestor := chain.GetBlock(parent, number)
		if ancestor == nil {
			break
		}
		ancestors[ancestor.Hash()] = ancestor.Header()
		for _, uncle := range ancestor.Uncles() {
			uncles.Add(uncle.Hash())
		}
		parent, number = ancestor.ParentHash(), number-1
	}
	ancestors[block.Hash()] = block.Header()
	uncles.Add(block.Hash())

	// Verify each of the unblocks
	for _, uncle := range block.Uncles() {
		// Make sure every uncle is rewarded only once
		hash := uncle.Hash()
		if uncles.Has(hash) {
			return errDuplicateUncle
		}
		uncles.Add(hash)

		// Make sure the uncle has a valid ancestry
		if ancestors[hash] != nil {
			return errUncleIsAncestor
		}
		if ancestors[uncle.ParentHash] == nil || uncle.ParentHash == block.ParentHash() {
			return errDanglingUncle
		}
		if err := ethash.verifyHeader(chain, uncle, ancestors[uncle.ParentHash], true, true); err != nil {
			return err
		}
	}
	return nil
}
```

Prepare implements the consensus engine's Prepare interface, which is used to fill the difficulty field of the block header to conform to the ethash protocol. This change is online.

```go
func (ethash *Ethash) Prepare(chain consensus.ChainReader, header *types.Header) error {
	parent := chain.GetHeader(header.ParentHash, header.Number.Uint64()-1)
	if parent == nil {
		return consensus.ErrUnknownAncestor
	}
	header.Difficulty = ethash.CalcDifficulty(chain, header.Time.Uint64(), parent)
	return nil
}
```

Finalize implements the Finalize interface of the consensus engine, rewards digging into block accounts and unblocked accounts, and populates the value of the root of the state tree. And return to the new block.

```go
func (ethash *Ethash) Finalize(chain consensus.ChainReader, header *types.Header, state *state.StateDB, txs []*types.Transaction, uncles []*types.Header, receipts []*types.Receipt) (*types.Block, error) {
	// Accumulate any block and uncle rewards and commit the final state root
	accumulateRewards(chain.Config(), state, header, uncles)
	header.Root = state.IntermediateRoot(chain.Config().IsEIP158(header.Number))

	// Header seems complete, assemble into a block and return
	return types.NewBlock(header, txs, uncles, receipts), nil
}

func accumulateRewards(config *params.ChainConfig, state *state.StateDB, header *types.Header, uncles []*types.Header) {
	// Select the correct block reward based on chain progression
	blockReward := FrontierBlockReward
	if config.IsByzantium(header.Number) {
		blockReward = ByzantiumBlockReward
	}
	// Accumulate the rewards for the miner and any included uncles
	reward := new(big.Int).Set(blockReward)
	r := new(big.Int)
	// Reward a bad block account块账户
	for _, uncle := range uncles {
		r.Add(uncle.Number, big8)
		r.Sub(r, header.Number)
		r.Mul(r, blockReward)
		r.Div(r, big8)
		state.AddBalance(uncle.Coinbase, r)

		r.Div(blockReward, big32)
		reward.Add(reward, r)
	}
	// Reward the coinbase account
	state.AddBalance(header.Coinbase, reward)
}
```

#### Seal function implementation

In the CPU mining part, the core function of CpuAgent calls the Seal function when performing the mining operation. The Seal function attempts to find a nonce value that satisfies the block difficulty. In ModeFake and ModeFullFake mode, it returns quickly and takes the nonce value directly to 0. In shared PoW mode, use the shared Seal function. Turn on threads goroutine for mining (find the qualified nonce value).

```go
// Seal implements consensus.Engine, attempting to find a nonce that satisfies
// the block's difficulty requirements.
func (ethash *Ethash) Seal(chain consensus.ChainReader, block *types.Block, stop <-chan struct{}) (*types.Block, error) {
	// In ModeFake and ModeFullFake mode, it returns quickly and takes the nonce value directly to 0.
	if ethash.config.PowMode == ModeFake || ethash.config.PowMode == ModeFullFake {
		header := block.Header()
		header.Nonce, header.MixDigest = types.BlockNonce{}, common.Hash{}
		return block.WithSeal(header), nil
	}
	// In shared PoW mode, use the shared Seal function.
	if ethash.shared != nil {
		return ethash.shared.Seal(chain, block, stop)
	}
	// Create a runner and the multiple search threads it directs
	abort := make(chan struct{})
	found := make(chan *types.Block)

	ethash.lock.Lock()
	threads := ethash.threads
	if ethash.rand == nil {
		seed, err := crand.Int(crand.Reader, big.NewInt(math.MaxInt64))
		if err != nil {
			ethash.lock.Unlock()
			return nil, err
		}
		ethash.rand = rand.New(rand.NewSource(seed.Int64()))
	}
	ethash.lock.Unlock()
	if threads == 0 {
		threads = runtime.NumCPU()
	}
	if threads < 0 {
		threads = 0 // Allows disabling local mining without extra logic around local/remote
	}
	var pend sync.WaitGroup
	for i := 0; i < threads; i++ {
		pend.Add(1)
		go func(id int, nonce uint64) {
			defer pend.Done()
			ethash.mine(block, id, nonce, abort, found)
		}(i, uint64(ethash.rand.Int63()))
	}
	// Wait until sealing is terminated or a nonce is found
	var result *types.Block
	select {
	case <-stop:
		// Outside abort, stop all miner threads
		close(abort)
	case result = <-found:
		// One of the threads found a block, abort all others
		close(abort)
	case <-ethash.update:
		// Thread count was changed on user request, restart
		close(abort)
		pend.Wait()
		return ethash.Seal(chain, block, stop)
	}
	// Wait for all mining goroutine returns
	pend.Wait()
	return result, nil
}
```

Mine is a true function for finding the nonce value, it traverses to find the nonce value and compares the PoW value to the target value. The principle can be briefly described as follows:

```go
								RAND(h, n)  <=  M / d
```

Here M represents a very large number, here is 2^256-1; d represents the Header member Difficulty. RAND() is a concept function that represents a series of complex operations and ultimately produces a random number. This function consists of two basic parameters: h is the hash of the Header (Header.HashNoNonce()), and n is the Header member Nonce. The whole relation can be roughly understood as trying to find a number in a way that does not exceed M in the maximum. If the number meets the condition (<=M/d), then Seal() is considered successful. It can be known from the above formula that M is constant, and the larger d is, the smaller the range is. So as the difficulty increases, the difficulty of dig out the block is also increasing.

```go
func (ethash *Ethash) mine(block *types.Block, id int, seed uint64, abort chan struct{}, found chan *types.Block) {
	// Get some data from the block header
	var (
		header  = block.Header()
		hash    = header.HashNoNonce().Bytes()
		// target is the upper limit of the PoW found target = maxUint256/Difficulty
		// where maxUint256 = 2^256-1 Difficulty is the difficulty value
		target  = new(big.Int).Div(maxUint256, header.Difficulty)
		number  = header.Number.Uint64()
		dataset = ethash.dataset(number)
	)
	// Try to find a nonce value until it terminates or finds the target value
	var (
		attempts = int64(0)
		nonce    = seed
	)
	logger := log.New("miner", id)
	logger.Trace("Started ethash search for new nonces", "seed", seed)
search:
	for {
		select {
		case <-abort:
			// stop mining
			logger.Trace("Ethash nonce search aborted", "attempts", nonce-seed)
			ethash.hashrate.Mark(attempts)
			break search

		default:
			// It is not necessary to update the hash rate at every nonce value, and update the hash rate every 2^x nonce values.
			attempts++
			if (attempts % (1 << 15)) == 0 {
				ethash.hashrate.Mark(attempts)
				attempts = 0
			}
			// Calculate the PoW value with this nonce
			digest, result := hashimotoFull(dataset.dataset, hash, nonce)
			// The calculated result is compared with the target value, and if it is less than the target value, the search is successful.
			if new(big.Int).SetBytes(result).Cmp(target) <= 0 {
				// Find the nonce value, update the block header
				header = types.CopyHeader(header)
				header.Nonce = types.EncodeNonce(nonce)
				header.MixDigest = common.BytesToHash(digest)

				// Pack the block header and return
				select {
				// WithSeal Replace the new block header with the old block header
				case found <- block.WithSeal(header):
					logger.Trace("Ethash nonce found and reported", "attempts", nonce-seed, "nonce", nonce)
				case <-abort:
					logger.Trace("Ethash nonce found but discarded", "attempts", nonce-seed, "nonce", nonce)
				}
				break search
			}
			nonce++
		}
	}
	// Datasets are unmapped in a finalizer. Ensure that the dataset stays live
	// during sealing so it's not unmapped while being read.
	runtime.KeepAlive(dataset)
}
```

The appeal function calls the hashimotoFull function to calculate the value of the PoW.

```go
func hashimotoFull(dataset []uint32, hash []byte, nonce uint64) ([]byte, []byte) {
	lookup := func(index uint32) []uint32 {
		offset := index * hashWords
		return dataset[offset : offset+hashWords]
	}
	return hashimoto(hash, nonce, uint64(len(dataset))*4, lookup)
}
```

Hashimoto is used to aggregate data to produce specific back-end hash and nonce values.

![images：https://blog.csdn.net/metal1/article/details/79682636](picture/pow_hashimoto.png)
Briefly describe the part of the process:

- First, Hashimoto () function into the reference @Hash and @Nonce combined into one array of 40 bytes long, it takes a SHA-512 hash value SEED name, a length of 64 bytes.。
- Then, convert seed[] into an array mix[] with uint32 as the element, note that a uint32 number is equal to 4 bytes, so seed[] can only be converted to 16 uint32 numbers, and the mix[] array is 32 in length, so this time The mix[] array is equal to each other.
- Next, the lookup() function comes up. With a loop, constantly call lookup() to extract the uint32 element type array from the external dataset, and mix the unknown data into the mix[] array. The number of loops can be adjusted with parameters and is currently set to 64 times. In each loop, the change generates the parameter index, so that each time the array that is called by the lookup() function is different. The way to mix data here is a vector-like XOR operation from the FNV algorithm. After the data to be confusing is completed, you get a basically unrecognizable mix[], a length of 32 uint32 array. At this time, it is folded (compressed) into a uint32 array whose length is reduced to 1/4 of the original length, and the folding operation method is still from the FNV algorithm.
- Finally, the collapsed mix[] is directly converted into a byte array of length 8 by a uint32 array of length 8. This is the return value @digest; the previous seed[] array is merged with the digest and the SHA- is taken again. A 256 hash value, resulting in a byte array of length 32, which is the return value @result . (Transferred from https://blog.csdn.net/metal1/article/details/79682636 ))

```go
func hashimoto(hash []byte, nonce uint64, size uint64, lookup func(index uint32) []uint32) ([]byte, []byte) {
	// Calculate the number of theoretical rows
	rows := uint32(size / mixBytes)

	// Replace header+nonce into a 64-byte seed
	seed := make([]byte, 40)
	copy(seed, hash)
	binary.LittleEndian.PutUint64(seed[32:], nonce)

	seed = crypto.Keccak512(seed)
	seedHead := binary.LittleEndian.Uint32(seed)

	// Convert seed[] to an array with uint32 as an element mix[]
	mix := make([]uint32, mixBytes/4)
	for i := 0; i < len(mix); i++ {
		mix[i] = binary.LittleEndian.Uint32(seed[i%16*4:])
	}
	// Mixing unknown data into the mix[] array
	temp := make([]uint32, len(mix))

	for i := 0; i < loopAccesses; i++ {
		parent := fnv(uint32(i)^seedHead, mix[i%len(mix)]) % rows
		for j := uint32(0); j < mixBytes/hashBytes; j++ {
			copy(temp[j*hashWords:], lookup(2*parent+j))
		}
		fnvHash(mix, temp)
	}
	// Compressed into a uint32 array whose length is reduced to 1/4 of the original length
	for i := 0; i < len(mix); i += 4 {
		mix[i/4] = fnv(fnv(fnv(mix[i], mix[i+1]), mix[i+2]), mix[i+3])
	}
	mix = mix[:len(mix)/4]

	digest := make([]byte, common.HashLength)
	for i, val := range mix {
		binary.LittleEndian.PutUint32(digest[i*4:], val)
	}
	return digest, crypto.Keccak256(append(seed, digest...))
}
```

#### VerifySeal function analysis

VerifySeal is used to verify that the nonce value of the block meets the PoW difficulty requirements.

```go
func (ethash *Ethash) VerifySeal(chain consensus.ChainReader, header *types.Header) error {
	// ModeFake and ModeFullFake modes are not verified and are directly verified.
	if ethash.config.PowMode == ModeFake || ethash.config.PowMode == ModeFullFake {
		time.Sleep(ethash.fakeDelay)
		if ethash.fakeFail == header.Number.Uint64() {
			return errInvalidPoW
		}
		return nil
	}
	// shared: Under PoW, use the shared verification method
	if ethash.shared != nil {
		return ethash.shared.VerifySeal(chain, header)
	}
	// Ensure that we have a valid difficulty for the block
	if header.Difficulty.Sign() <= 0 {
		return errInvalidDifficulty
	}
	// Calculate the digest and PoW values and verify the block header
	number := header.Number.Uint64()

	cache := ethash.cache(number)
	size := datasetSize(number)
	if ethash.config.PowMode == ModeTest {
		size = 32 * 1024
	}
	digest, result := hashimotoLight(size, cache.cache, header.HashNoNonce().Bytes(), header.Nonce.Uint64())
	// Caches are unmapped in a finalizer. Ensure that the cache stays live
	// until after the call to hashimotoLight so it's not unmapped while being used.
	runtime.KeepAlive(cache)

	if !bytes.Equal(header.MixDigest[:], digest) {
		return errInvalidMixDigest
	}
	target := new(big.Int).Div(maxUint256, header.Difficulty)
	// Compare whether the target difficulty requirements are met
	if new(big.Int).SetBytes(result).Cmp(target) > 0 {
		return errInvalidPoW
	}
	return nil
}
```

hashimotoLight and hashimotoFull function similarly, except that hashimotoLight uses a smaller memory that occupies less memory.

```go
func hashimotoLight(size uint64, cache []uint32, hash []byte, nonce uint64) ([]byte, []byte) {
	keccak512 := makeHasher(sha3.NewKeccak512())

	lookup := func(index uint32) []uint32 {
		rawData := generateDatasetItem(cache, index, keccak512)

		data := make([]uint32, len(rawData)/4)
		for i := 0; i < len(data); i++ {
			data[i] = binary.LittleEndian.Uint32(rawData[i*4:])
		}
		return data
	}
	return hashimoto(hash, nonce, size, lookup)
}
```
