
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>##eth POW Analysis
### Consensus Engine Description
In the CPU mining part, the CpuAgent mine function calls the self.engine.Seal function when performing the mining operation. The engine here is the consensus engine. Seal is one of the most important interfaces. It implements the search for nonce values ​​and the calculation of hashes. And this function is an important function that guarantees consensus and cannot be forged.
In the PoW consensus algorithm, the Seal function implements the working proof. This part of the source code is under consensus/ethhash.
### Consensus Engine Interface
Type engine interface {
/ / Get the block digger, that is, coinbase
Author(header *types.Header) (common.Address, error)


// VerifyHeader is used to check the block header and verify it by consensus rules. The verification block can be used here. Ketong passes the VerifySeal method.
VerifyHeader(chain ChainReader, header *types.Header, seal bool) error


// VerifyHeaders is similar to VerifyHeader, and this is used to batch checkpoints. This method returns an exit signal
// Used to terminate the operation for asynchronous verification.
VerifyHeaders(chain ChainReader, headers []*types.Header, seals []bool) (chan&lt;- struct{}, &lt;-chan error)

// VerifyUncles is used to verify the unblock to conform to the rules of the consensus engine
VerifyUncles(chain ChainReader, block *types.Block) error

// VerifySeal checks the block header according to the rules of the consensus algorithm
VerifySeal(chain ChainReader, header *types.Header) error

// Prepare the consensus field used to initialize the block header based on the consensus engine. These changes are all implemented inline.
Prepare(chain ChainReader, header *types.Header) error

// Finalize completes all state changes and eventually assembles into blocks.
// The block header and state database can be updated to conform to the consensus rules at the time of final validation.
Finalize(chain ChainReader, header *types.Header, state *state.StateDB, txs []*types.Transaction,
Uncles []*types.Header, receipts []*types.Receipt) (*types.Block, error)

// Seal packs a new block based on the input block
Seal(chain ChainReader, block *types.Block, stop &lt;-chan struct{}) (*types.Block, error)

// CalcDifficulty is the difficulty adjustment algorithm that returns the difficulty value of the new block.
CalcDifficulty(chain ChainReader, time uint64, parent *types.Header) *big.Int

// APIs return RPC APIs provided by the consensus engine
APIs(chain ChainReader) []rpc.API
}
### ethhash Implementation Analysis
#### ethhash Structure
```
Type Ethash struct {
Config Config

// cache
Caches *lru // In memory caches to avoid regenerating too often
// memory data set
Datasets *lru // In memory datasets to avoid regenerating too often

// Mining related fields
Rand *rand.Rand // Properly seeded random source for nonces
// number of mining threads
Threads int // Number of threads to mine on if mining
// channel is used to update the mining notice
Update chan struct{} // Notification channel to update mining parameters
Hashrate metrics.Meter // Meter tracking the average hashrate

// Test network related parameters
// The fields below are hooks for testing
Shared *Ethash // Shared PoW verifier to avoid cache regeneration
fakeFail uint64 // Block number which fails PoW check even in fake mode
fakeDelay time.Duration // Time delay to sleep for before returning from verify

Lock sync.Mutex // Ensures thread safety for the in-memory caches and mining fields
}
```
Ethhash is a concrete implementation of PoW. Since there are a large number of data sets to be used, all have two pointers to lru. And through the threads control the number of mining threads. And in test mode or fake mode, simple and fast processing, so that it can get results quickly.

The Athor method obtained the address of the miner who dug out the block.
```
Func (ethash *Ethash) Author(header *types.Header) (common.Address, error) {
Return header.Coinbase, nil
}
```
VerifyHeader is used to verify that the block header information conforms to the ethash consensus engine rules.
```
// VerifyHeader checks whether a header conforms to the consensus rules of the
// stock Ethereum ethash engine.
Func (ethash *Ethash) VerifyHeader(chain consensus.ChainReader, header *types.Header, seal bool) error {
// When in ModeFullFake mode, any header information is accepted
If ethash.config.PowMode == ModeFullFake {
Return nil
}
// If the header is known, return without checking.
Number := header.Number.Uint64()
If chain.GetHeader(header.Hash(), number) != nil {
Return nil
}
Parent := chain.GetHeader(header.ParentHash, number-1)
If parent == nil { // Get the parent node failed
Return consensus.ErrUnknownAncestor
}
// further header verification
Return ethash.verifyHeader(chain, header, parent, false, seal)
}
```
Then look at the implementation of verifyHeader,
```
Func (ethash *Ethash) verifyHeader(chain consensus.ChainReader, header, parent *types.Header, uncle bool, seal bool) error {
// Make sure the extra data segment has a reasonable length
If uint64(len(header.Extra)) &gt; params.MaximumExtraDataSize {
Return fmt.Errorf("extra-data too long: %d &gt; %d", len(header.Extra), params.MaximumExtraDataSize)
}
// check the timestamp
If uncle {
If header.Time.Cmp(math.MaxBig256) &gt; 0 {
Return errLargeBlockTime
}
} else {
If header.Time.Cmp(big.NewInt(time.Now().Add(allowedFutureBlockTime).Unix())) &gt; 0 {
Return consensus.ErrFutureBlock
}
}
If header.Time.Cmp(parent.Time) &lt;= 0 {
Return errZeroBlockTime
}
// Check the difficulty of the block based on the timestamp and the difficulty of the parent block.
Expected := ethash.CalcDifficulty(chain, header.Time.Uint64(), parent)

If expected.Cmp(header.Difficulty) != 0 {
Return fmt.Errorf("invalid difficulty: have %v, want %v", header.Difficulty, expected)
}
// check gas limit &lt;= 2^63-1
Cap := uint64(0x7fffffffffffffff)
If header.GasLimit &gt; cap {
Return fmt.Errorf("invalid gasLimit: have %v, max %v", header.GasLimit, cap)
}
// check gasUsed &lt;= gasLimit
If header.GasUsed &gt; header.GasLimit {
Return fmt.Errorf("invalid gasUsed: have %d, gasLimit %d", header.GasUsed, header.GasLimit)
}

// Is the gas limit within the allowable range?
Diff := int64(parent.GasLimit) - int64(header.GasLimit)
If diff &lt; 0 {
Diff *= -1
}
Limit := parent.GasLimit / params.GasLimitBoundDivisor

If uint64(diff) &gt;= limit || header.GasLimit &lt; params.MinGasLimit {
Return fmt.Errorf("invalid gas limit: have %d, want %d += %d", header.GasLimit, parent.GasLimit, limit)
}
// The check block number should be the parent block block number +1
If diff := new(big.Int).Sub(header.Number, parent.Number); diff.Cmp(big.NewInt(1)) != 0 {
Return consensus.ErrInvalidNumber
}
// Verify that a particular block meets the requirements
If seal {
If err := ethash.VerifySeal(chain, header); err != nil {
Return err
}
}
// Validate the special fields of the hard fork if all checks pass.
If err := misc.VerifyDAOHeaderExtraData(chain.Config(), header); err != nil {
Return err
}
If err := misc.VerifyForkHashes(chain.Config(), header, uncle); err != nil {
Return err
}
Return nil
}
```
Ethash calculates the difficulty of the next block through the CalcDifficulty function, and creates different difficulty calculation methods for the difficulty of different stages.
```
Func (ethash *Ethash) CalcDifficulty(chain consensus.ChainReader, time uint64, parent *types.Header) *big.Int {
Return CalcDifficulty(chain.Config(), time, parent)
}

Func CalcDifficulty(config *params.ChainConfig, time uint64, parent *types.Header) *big.Int {
Next := new(big.Int).Add(parent.Number, big1)
Switch {
Case config.IsByzantium(next):
Return calcDifficultyByzantium(time, parent)
Case config.IsHomestead(next):
Return calcDifficultyHomestead(time, parent)
Default:
Return calcDifficultyFrontier(time, parent)
}
}
```
VerifyHeaders is similar to VerifyHeader except that VerifyHeaders performs bulk check operations. Create multiple goroutines to perform validation operations, and then create a goroutine for assignment control task assignment and result acquisition. Finally return a result channel
```
Func (ethash *Ethash) VerifyHeaders(chain consensus.ChainReader, headers []*types.Header, seals []bool) (chan&lt;- struct{}, &lt;-chan error) {
// If we're running a full engine faking, accept any input as valid
If ethash.config.PowMode == ModeFullFake || len(headers) == 0 {
Abort, results := make(chan struct{}), make(chan error, len(headers))
For i := 0; i &lt; len(headers); i++ {
Results &lt;- nil
}
Return abort, results
}

// Spawn as many workers as allowed threads
Workers := runtime.GOMAXPROCS(0)
If len(headers) &lt; workers {
Workers = len(headers)
}

// Create a task channel and spawn the verifiers
Var (
Inputs = make(chan int)
Done = make(chan int, workers)
Errors = make([]error, len(headers))
Abort = make(chan struct{})
)
/ / Generate workers goroutine for check head
For i := 0; i &lt; workers; i++ {
Go func() {
For index := range inputs {
Errors[index] = ethash.verifyHeaderWorker(chain, headers, seals, index)
Done &lt;- index
}
}()
}

errorsOut := make(chan error, len(headers))
// goroutine is used to send tasks to workers goroutine
Go func() {
Defer close(inputs)
Var (
In, out = 0, 0
Checked = make([]bool, len(headers))
Inputs inputs
)
For {
Select {
Case inputs &lt;- in:
If in++; in == len(headers) {
// Reached end of headers. Stop sending to workers.
Inputs = nil
}
// Statistics and send an error message to errorsOut
Case index := &lt;-done:
For checked[index] = true; checked[out]; out++ {
errorsOut &lt;- errors[out]
If out == len(headers)-1 {
Return
}
}
Case &lt;-abort:
Return
}
}
}()
Return abort, errorsOut
}
```
VerifyHeaders uses verifyHeaderWorker when checking a single block header. After the function gets the parent block, it calls verifyHeader to check it.
```
Func (ethash *Ethash) verifyHeaderWorker(chain consensus.ChainReader, headers []*types.Header, seals []bool, index int) error {
Var parent *types.Header
If index == 0 {
Parent = chain.GetHeader(headers[0].ParentHash, headers[0].Number.Uint64()-1)
} else if headers[index-1].Hash() == headers[index].ParentHash {
Parent = headers[index-1]
}
If parent == nil {
Return consensus.ErrUnknownAncestor
}
If chain.GetHeader(headers[index].Hash(), headers[index].Number.Uint64()) != nil {
Return nil // known block
}
Return ethash.verifyHeader(chain, headers[index], parent, false, seals[index])
}

```
VerifyUncles is used for verification of the unblock. Similar to the check block header, the unchecked block check returns directly in the ModeFullFake mode. Get all the uncle blocks, then traverse the checksum, the checksum will terminate, or the checksum will return.
```
Func (ethash *Ethash) VerifyUncles(chain consensus.ChainReader, block *types.Block) error {
// If we're running a full engine faking, accept any input as valid
If ethash.config.PowMode == ModeFullFake {
Return nil
}
// Verify that there are at at 2 uncles included in this block
If len(block.Uncles()) &gt; maxUncles {
Return errTooManyUncles
}
// Collect the uncle and its ancestors
Uncles, ancestors := set.New(), make(map[common.Hash]*types.Header)

Number, parent := block.NumberU64()-1, block.ParentHash()
For i := 0; i &lt; 7; i++ {
Ancestor := chain.GetBlock(parent, number)
If ancestor == nil {
Break
}
Ancestors[ancestor.Hash()] = ancestor.Header()
For _, uncle := range ancestor.Uncles() {
uncles.Add(uncle.Hash())
}
Parent, number = ancestor.ParentHash(), number-1
}
Ancestors[block.Hash()] = block.Header()
uncles.Add(block.Hash())

// Verify each of the unblocks
For _, uncle := range block.Uncles() {
// Make sure every uncle is rewarded only once
Hash := uncle.Hash()
If uncles.Has(hash) {
Return errDuplicateUncle
}
uncles.Add(hash)

// Make sure the uncle has a valid ancestry
If ancestors[hash] != nil {
Return errUncleIsAncestor
}
If ancestors[uncle.ParentHash] == nil || uncle.ParentHash == block.ParentHash() {
Return errDanglingUncle
}
If err := ethash.verifyHeader(chain, uncle, ancestors[uncle.ParentHash], true, true); err != nil {
Return err
}
}
Return nil
}
```
Prepare implements the consensus interface of the consensus engine, which is used to fill the difficulty field of the block header to make it conform to the ethash protocol. This change is online.
```
Func (ethash *Ethash) Prepare(chain consensus.ChainReader, header *types.Header) error {
Parent := chain.GetHeader(header.ParentHash, header.Number.Uint64()-1)
If parent == nil {
Return consensus.ErrUnknownAncestor
}
header.Difficulty = ethash.CalcDifficulty(chain, header.Time.Uint64(), parent)
Return nil
}
```
Finalize implements the Finalize interface of the consensus engine, rewards digging into block accounts and unblocked accounts, and populates the value of the root of the state tree. And return to the new block.
```
Func (ethash *Ethash) Finalize(chain consensus.ChainReader, header *types.Header, state *state.StateDB, txs []*types.Transaction, uncles []*types.Header, receipts []*types.Receipt) ( *types.Block, error) {
// Accumulate any block and uncle rewards and commit the final state root
accumulateRewards(chain.Config(), state, header, uncles)
header.Root = state.IntermediateRoot(chain.Config().IsEIP158(header.Number))

// Header seems complete, assemble into a block and return
Return types.NewBlock(header, txs, uncles, receipts), nil
}

Func accumulateRewards(config *params.ChainConfig, state *state.StateDB, header *types.Header, uncles []*types.Header) {
// Select the correct block reward based on chain progression
blockReward := FrontierBlockReward
If config.IsByzantium(header.Number) {
blockReward = ByzantiumBlockReward
}
// Accumulate the rewards for the miner and any included uncles
Reward := new(big.Int).Set(blockReward)
r := new(big.Int)
// Reward a bad block account
For _, uncle := range uncles {
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
#### Seal function implementation analysis
In the CPU mining part, the CpuAgent mine function calls the Seal function when performing the mining operation. The Seal function attempts to find a nonce value that satisfies the block difficulty.
In ModeFake and ModeFullFake mode, it returns quickly and takes the nonce value directly to 0.
In shared PoW mode, use the shared Seal function.
Open threads goroutine for mining (find the qualified nonce value).
```
// Seal implements consensus.Engine, attempting to find a nonce that satisfies
// the block's difficulty requirements.
Func (ethash *Ethash) Seal(chain consensus.ChainReader, block *types.Block, stop &lt;-chan struct{}) (*types.Block, error) {
// In ModeFake and ModeFullFake mode, return quickly and take the nonce value directly to 0.
If ethash.config.PowMode == ModeFake || ethash.config.PowMode == ModeFullFake {
Header := block.Header()
header.Nonce, header.MixDigest = types.BlockNonce{}, common.Hash{}
Return block.WithSeal(header), nil
}
// In shared PoW mode, use the shared Seal function.
If ethash.shared != nil {
Return ethash.shared.Seal(chain, block, stop)
}
// Create a runner and the multiple search threads it directs
Abort := make(chan struct{})
Found := make(chan *types.Block)

ethash.lock.Lock()
Threads := ethash.threads
If ethash.rand == nil {
Seed, err := crand.Int(crand.Reader, big.NewInt(math.MaxInt64))
If err != nil {
ethash.lock.Unlock()
Return nil, err
}
Ethash.rand = rand.New(rand.NewSource(seed.Int64()))
}
ethash.lock.Unlock()
If threads == 0 {
Threads = runtime.NumCPU()
}
If threads &lt; 0 {
Threads = 0 // Allows disabling local mining without extra logic around local/remote
}
Var pend sync.WaitGroup
For i := 0; i &lt; threads; i++ {
pend.Add(1)
Go func(id int, nonce uint64) {
Defer pend.Done()
Ethash.mine(block, id, nonce, abort, found)
}(i, uint64(ethash.rand.Int63()))
}
// Wait until sealing is terminated or a nonce is found
Var result *types.Block
Select {
Case &lt;-stop:
// Outside abort, stop all miner threads
Close(abort)
Case result = &lt;-found:
// One of the threads found a block, abort all others
Close(abort)
Case &lt;-ethash.update:
// Thread count was changed on user request, restart
Close(abort)
pend.Wait()
Return ethash.Seal(chain, block, stop)
}
// Wait for all mining goroutine returns
pend.Wait()
Return result, nil
}
```
Mine is a true function for finding the nonce value. It traverses to find the nonce value and compares the PoW value with the target value.
The principle can be briefly described as follows:
```
RAND(h, n) &lt;= M / d
```
Here M represents a very large number, here is 2^256-1; d represents the Header member Difficulty. RAND() is a concept function that represents a series of complex operations and ultimately produces a random number. This function consists of two basic parameters: h is the hash of the Header (Header.HashNoNonce()), and n is the leader of the Header. The whole relation can be roughly understood as trying to find a number in a way that does not exceed M in the maximum. If the number meets the condition (&lt;=M/d), then Seal() is considered successful.
It can be known from the above formula that M is constant, and the larger d is, the smaller the range is. So as the difficulty increases, the difficulty of dig out the block is also increasing.
```
Func (ethash *Ethash) mine(block *types.Block, id int, seed uint64, abort chan struct{}, found chan *types.Block) {
// Get some data from the block header
Var (
Header = block.Header()
Hash = header.HashNoNonce().Bytes()
// target is the upper limit of the PoW found target = maxUint256/Difficulty
// where maxUint256 = 2^256-1 Difficulty is the difficulty value
Target = new(big.Int).Div(maxUint256, header.Difficulty)
Number = header.Number.Uint64()
Dataset = ethash.dataset(number)
)
// Try to find a nonce value until it terminates or finds the target value
Var (
Attempts = int64(0)
Nonce = seed
)
Logger := log.New("miner", id)
logger.Trace("Started ethash search for new nonces", "seed", seed)
Search:
For {
Select {
Case &lt;-abort:
// stop mining
logger.Trace("Ethash nonce search aborted", "attempts", nonce-seed)
ethash.hashrate.Mark(attempts)
Break search

Default:
// Don't update the hash rate in every nonce value, update the hash rate every 2^x nonce values
Attempts++
If (attempts % (1 &lt;&lt; 15)) == 0 {
ethash.hashrate.Mark(attempts)
Attempts = 0
}
// Calculate the PoW value with this nonce
Digest, result := hashimotoFull(dataset.dataset, hash, nonce)
// Compare the calculated result with the target value. If it is less than the target value, the search is successful.
If new(big.Int).SetBytes(result).Cmp(target) &lt;= 0 {
/ / Find the nonce value, update the block header
Header = types.CopyHeader(header)
header.Nonce = types.EncodeNonce(nonce)
header.MixDigest = common.BytesToHash(digest)

// Pack the block header and return
Select {
// WithSeal replaces the new block header with the old block header
Case found &lt;- block.WithSeal(header):
logger.Trace("Ethash nonce found and reported", "attempts", nonce-seed, "nonce", nonce)
Case &lt;-abort:
logger.Trace("Ethash nonce found but discarded", "attempts", nonce-seed, "nonce", nonce)
}
Break search
}
Nonce++
}
}
// Datasets are unmapped in a finalizer. Ensure that the dataset stays live
// during sealing so it's not unmapped while being read.
runtime.KeepAlive(dataset)
}
```
The appeal function calls the hashimotoFull function to calculate the value of the PoW.
```
Func hashimotoFull(dataset []uint32, hash []byte, nonce uint64) ([]byte, []byte) {
Lookup := func(index uint32) []uint32 {
Offset := index * hashWords
Return dataset[offset : offset+hashWords]
}
Return hashimoto(hash, nonce, uint64(len(dataset))*4, lookup)
}
```
Hashimoto is used to aggregate data to produce specific back-end hash and nonce values.
[Image source: https://blog.csdn.net/metal1/article/details/79682636](picture/pow_hashimoto.png)
Briefly describe the part of the process:
- First, the hashimoto() function merges the @hash and @nonce arguments into a 40-byte array, taking its SHA-512 hash value as a seed with a length of 64 bytes.
- Then, convert seed[] into an array mix[] with uint32 as the element. Note that a uint32 number is equal to 4 bytes, so seed[] can only be converted to 16 uint32 numbers, and the mix[] array is 32 in length, so this When the mix[] array is before and after the half is equivalent.
- Next, the lookup() function comes up. Use a loop, constantly call lookup() to extract the uint32 element type array from the external dataset, and mix the unknown data into the mix[] array. The number of cycles can be adjusted with parameters and is currently set to 64 times. In each loop, the change generates the parameter index, so that each time the array that is called by the lookup() function is different. The way to mix data here is a vector-like XOR operation from the FNV algorithm.
After the data to be confusing is completed, a substantially unrecognizable mix[] and a uint32 array of length 32 are obtained. At this time, it is folded (compressed) into a uint32 array whose length is reduced to 1/4 of the original length, and the folding operation method is still from the FNV algorithm.
- Finally, the collapsed mix[] is directly converted into a byte array of length 32 by a uint32 array of length 8. This is the return value @digest; the previous seed[] array is merged with the digest and the SHA is taken once again. The -256 hash value yields a byte array of length 32, which is the return value @result. (Transferred from https://blog.csdn.net/metal1/article/details/79682636)
```
Func hashimoto(hash []byte, nonce uint64, size uint64, lookup func(index uint32) []uint32) ([]byte, []byte) {
/ / Calculate the number of theoretical lines
Rows := uint32(size / mixBytes)

// Replace header+nonce into a 64-byte seed
Seed := make([]byte, 40)
Copy(seed, hash)
binary.LittleEndian.PutUint64(seed[32:], nonce)

Seed = crypto.Keccak512(seed)
seedHead := binary.LittleEndian.Uint32(seed)

// Convert seed[] to an array with uint32 as an element mix[]
Mix := make([]uint32, mixBytes/4)
For i := 0; i &lt; len(mix); i++ {
Mix[i] = binary.LittleEndian.Uint32(seed[i%16*4:])
}
// Mix unknown data into the mix[] array
Temp := make([]uint32, len(mix))

For i := 0; i &lt; loopAccesses; i++ {
Parent := fnv(uint32(i)^seedHead, mix[i%len(mix)]) % rows
For j := uint32(0); j &lt; mixBytes/hashBytes; j++ {
Copy(temp[j*hashWords:], lookup(2*parent+j))
}
fnvHash(mix, temp)
}
// Compressed into a uint32 array whose length is reduced to 1/4 of the original length
For i := 0; i &lt; len(mix); i += 4 {
Mix[i/4] = fnv(fnv(fnv(mix[i], mix[i+1]), mix[i+2]), mix[i+3])
}
Mix = mix[:len(mix)/4]

Digest := make([]byte, common.HashLength)
For i, val := range mix {
binary.LittleEndian.PutUint32(digest[i*4:], val)
}
Return digest, crypto.Keccak256(append(seed, digest...))
}
```
#### VerifySeal function implementation analysis
VerifySeal is used to verify that the nonce value of the block meets the PoW difficulty requirements.
```
Func (ethash *Ethash) VerifySeal(chain consensus.ChainReader, header *types.Header) error {
// ModeFake, ModeFullFake mode is not verified, and the verification is passed directly.
If ethash.config.PowMode == ModeFake || ethash.config.PowMode == ModeFullFake {
time.Sleep(ethash.fakeDelay)
If ethash.fakeFail == header.Number.Uint64() {
Return errInvalidPoW
}
Return nil
}
// shared PoW, using the shared verification method
If ethash.shared != nil {
Return ethash.shared.VerifySeal(chain, header)
}
// Ensure that we have a valid difficulty for the block
If header.Difficulty.Sign() &lt;= 0 {
Return errInvalidDifficulty
}
// Calculate the digest and PoW values ​​and check the block header
Number := header.Number.Uint64()

Cache := ethash.cache(number)
Size := datasetSize(number)
If ethash.config.PowMode == ModeTest {
Size = 32 * 1024
}
Digest, result := hashimotoLight(size, cache.cache, header.HashNoNonce().Bytes(), header.Nonce.Uint64())
// Caches are unmapped in a finalizer. Ensure that the cache stays live
// until after the call to hashimotoLight so it's not unmapped while being used.
runtime.KeepAlive(cache)

If !bytes.Equal(header.MixDigest[:], digest) {
Return errInvalidMixDigest
}
Target := new(big.Int).Div(maxUint256, header.Difficulty)
/ / Compare whether the target difficulty requirements are met
If new(big.Int).SetBytes(result).Cmp(target) &gt; 0 {
Return errInvalidPoW
}
Return nil
}
```
hashimotoLight and hashimotoFull function similarly, except that hashimotoLight uses a smaller cache that takes up less memory.
```
Func hashimotoLight(size uint64, cache []uint32, hash []byte, nonce uint64) ([]byte, []byte) {
Keccak512 := makeHasher(sha3.NewKeccak512())

Lookup := func(index uint32) []uint32 {
rawData := generateDatasetItem(cache, index, keccak512)

Data := make([]uint32, len(rawData)/4)
For i := 0; i &lt; len(data); i++ {
Data[i] = binary.LittleEndian.Uint32(rawData[i*4:])
}
Return data
}
Return hashimoto(hash, nonce, size, lookup)
}
```</pre></body></html>