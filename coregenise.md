
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Genesis is the meaning of the creation block. A blockchain is formed from the same creation block and is formed by rules. Different networks have different creation blocks, creation networks of the main network and test network. It is different.

This module sets the state of genesis based on the initial value of the incoming genesis and the database. If there is no creation block, create it in the database.

data structure

// Genesis specifies the header fields, state of a genesis block. It also defines hard
// fork switch-over blocks through the chain configuration.
// Genesis specifies the field of the header, the state of the starting block. It also configures to define hard fork switching blocks.
Type Genesis struct {
Config *params.ChainConfig `json:"config"`
Nonce uint64 `json:"nonce"`
Timestamp uint64 `json:"timestamp"`
ExtraData []byte `json:"extraData"`
GasLimit uint64 `json:"gasLimit" gencodec:"required"`
Difficulty *big.Int `json:"difficulty" gencodec:"required"`
Mixhash common.Hash `json:"mixHash"`
Coinbase common.Address `json:"coinbase"`
Alloc GenesisAlloc `json:"alloc" gencodec:"required"`

// These fields are used for consensus tests. Please don't use them
// in actual genesis blocks.
Number uint64 `json:"number"`
GasUsed uint64 `json:"gasUsed"`
ParentHash common.Hash `json:"parentHash"`
}

// GenesisAlloc specifies the initial state that is part of the genesis block.
// GenesisAlloc specifies the initial state of the first block.
Type GenesisAlloc map[common.Address]GenesisAccount


SetupGenesisBlock,

// SetupGenesisBlock writes or updates the genesis block in db.
//
// The block that will be used is:
//
// genesis == nil genesis != nil
// +------------------------------------------
// db has no genesis | main-net default | genesis
// db has genesis | from DB | genesis (if compatible)
//
// The stored chain configuration will be updated if it is compatible (i.e. does not
// specify a fork block below the local head block). In case of a conflict, the
// error is a *params.ConfigCompatError and the new, unwritten config is returned.
// If the stored blockchain configuration is not compatible then it will be updated (). In order to avoid conflicts, an error will be returned and the new configuration and the original configuration will be returned.
// The returned chain configuration is never nil.

// genesis is not nil if it is testnet dev or rinkeby mode. If it is mainnet or a private link. So empty
Func SetupGenesisBlock(db ethdb.Database, genesis *Genesis) (*params.ChainConfig, common.Hash, error) {
If genesis != nil &amp;&amp; genesis.Config == nil {
Return params.AllProtocolChanges, common.Hash{}, errGenesisNoConfig
}

// Just commit the new block if there is no stored genesis block.
Stored := GetCanonicalHash(db, 0) //Get the block corresponding to genesis
If (stored == common.Hash{}) { //If there is no block, the geth will start here.
If genesis == nil {
//If the genesis is nil and the stored is also nil then use the main network
// If it is test dev rinkeby then the genesis is not empty and will be set to the respective genesis
log.Info("Writing default main-net genesis block")
Genesis = DefaultGenesisBlock()
} else { // Otherwise use the configured block
log.Info("Writing custom genesis block")
}
// write to the database
Block, err := genesis.Commit(db)
Return genesis.Config, block.Hash(), err
}

// Check whether the genesis block is already written.
If genesis != nil { //If the genesis exists and the block exists, then compare the two blocks to see if they are the same
Block, _ := genesis.ToBlock()
Hash := block.Hash()
If hash != stored {
Return genesis.Config, block.Hash(), &amp;GenesisMismatchError{stored, hash}
}
}

// Get the existing chain configuration.
// Get the genesis configuration of the currently existing blockchain
Newcfg := genesis.configOrDefault(stored)
/ / Get the current blockchain configuration
Storedcfg, err := GetChainConfig(db, stored)
If err != nil {
If err == ErrChainConfigNotFound {
// This case happens if a genesis write was interrupted.
log.Warn("Found genesis block without chain config")
Err = WriteChainConfig(db, stored, newcfg)
}
Return newcfg, stored, err
}
// Special case: don't change the existing config of a non-mainnet chain if no new
// config is supplied. These chains would get AllProtocolChanges (and a compat error)
// if we just continued here.
// Special case: If you do not provide a new configuration, do not change the existing configuration of the non-primary network link.
// If we continue here, these chains will get AllProtocolChanges (and compat errors).
If genesis == nil &amp;&amp; stored != params.MainnetGenesisHash {
Return storedcfg, stored, nil // If it is a private link it will exit from here.
}

// Check config compatibility and write the config. Compatibility errors
// are returned to the caller unless we're already at block zero.
// Check the compatibility of the configuration, unless we are in block 0, otherwise return a compatibility error.
Height := GetBlockNumber(db, GetHeadHeaderHash(db))
If height == missingNumber {
Return newcfg, stored, fmt.Errorf("missing block number for head header hash")
}
compatErr := storedcfg.CheckCompatible(newcfg, height)
// If the block has already written data, then the genesis configuration cannot be changed.
If compatErr != nil &amp;&amp; height != 0 &amp;&amp; compatErr.RewindTo != 0 {
Return newcfg, stored, compatErr
}
// If the main network will exit from here.
Return newcfg, stored, WriteChainConfig(db, stored, newcfg)
}


ToBlock, this method uses the genesis data, uses a memory-based database, then creates a block and returns.


// ToBlock creates the block and state of a genesis specification.
Func (g *Genesis) ToBlock() (*types.Block, *state.StateDB) {
Db, _ := ethdb.NewMemDatabase()
Statedb, _ := state.New(common.Hash{}, state.NewDatabase(db))
For addr, account := range g.Alloc {
statedb.AddBalance(addr, account.Balance)
statedb.SetCode(addr, account.Code)
statedb.SetNonce(addr, account.Nonce)
For key, value := range account.Storage {
statedb.SetState(addr, key, value)
}
}
Root := statedb.IntermediateRoot(false)
Head := &amp;types.Header{
Number: new(big.Int).SetUint64(g.Number),
Nonce: types.EncodeNonce(g.Nonce),
Time: new(big.Int).SetUint64(g.Timestamp),
ParentHash: g.ParentHash,
Extra: g.ExtraData,
GasLimit: new(big.Int).SetUint64(g.GasLimit),
GasUsed: new(big.Int).SetUint64(g.GasUsed),
Difficulty: g.Difficulty,
MixDigest: g.Mixhash,
Coinbase: g.Coinbase,
Root: root,
}
If g.GasLimit == 0 {
head.GasLimit = params.GenesisGasLimit
}
If g.Difficulty == nil {
head.Difficulty = params.GenesisDifficulty
}
Return types.NewBlock(head, nil, nil, nil), statedb
}

The Commit method and the MustCommit method, the Commit method writes the block and state of a given genesis into the database. This block is considered to be the canonical blockchain.

// Commit writes the block and state of a genesis specification to the database.
// The block is committed as the canonical head block.
Func (g *Genesis) Commit(db ethdb.Database) (*types.Block, error) {
Block, statedb := g.ToBlock()
If block.Number().Sign() != 0 {
Return nil, fmt.Errorf("can't commit genesis block with number &gt; 0")
}
If _, err := statedb.CommitTo(db, false); err != nil {
Return nil, fmt.Errorf("cannot write state: %v", err)
}
// write the total difficulty
If err := WriteTd(db, block.Hash(), block.NumberU64(), g.Difficulty); err != nil {
Return nil, err
}
// write block
If err := WriteBlock(db, block); err != nil {
Return nil, err
}
// Write block receipt
If err := WriteBlockReceipts(db, block.Hash(), block.NumberU64(), nil); err != nil {
Return nil, err
}
// write headerPrefix + num (uint64 big endian) + numSuffix -&gt; hash
If err := WriteCanonicalHash(db, block.Hash(), block.NumberU64()); err != nil {
Return nil, err
}
// write "LastBlock" -&gt; hash
If err := WriteHeadBlockHash(db, block.Hash()); err != nil {
Return nil, err
}
// write "LastHeader" -&gt; hash
If err := WriteHeadHeaderHash(db, block.Hash()); err != nil {
Return nil, err
}
Config := g.Config
If config == nil {
Config = params.AllProtocolChanges
}
// write ethereum-config-hash -&gt; config
Return block, WriteChainConfig(db, block.Hash(), config)
}

// MustCommit writes the genesis block and state to db, panicking on error.
// The block is committed as the canonical head block.
Func (g *Genesis) MustCommit(db ethdb.Database) *types.Block {
Block, err := g.Commit(db)
If err != nil {
Panic(err)
}
Return block
}

Return to the default Genesis for various modes

// DefaultGenesisBlock returns the Ethereum main net genesis block.
Func DefaultGenesisBlock() *Genesis {
Return &amp;Genesis{
Config: params.MainnetChainConfig,
Nonce: 66,
ExtraData: hexutil.MustDecode("0x11bbe8db4e347b4e8c937c1c8370e4b5ed33adb3db69cbdb7a38e1e50b1b82fa"),
GasLimit: 5000,
Difficulty: big.NewInt(17179869184),
Alloc: decodePrealloc(mainnetAllocData),
}
}

// DefaultTestnetGenesisBlock returns the Ropsten network genesis block.
Func DefaultTestnetGenesisBlock() *Genesis {
Return &amp;Genesis{
Config: params.TestnetChainConfig,
Nonce: 66,
ExtraData: hexutil.MustDecode("0x3535353535353535353535353535353535353535353535353535353535353535"),
GasLimit: 16777216,
Difficulty: big.NewInt(1048576),
Alloc: decodePrealloc(testnetAllocData),
}
}

// DefaultRinkebyGenesisBlock returns the Rinkeby network genesis block.
Func DefaultRinkebyGenesisBlock() *Genesis {
Return &amp;Genesis{
Config: params.RinkebyChainConfig,
Timestamp: 1492009146,
ExtraData: hexutil.MustDecode("0x52657370656374206d7920617574686f7269746168207e452e436172746d616e42eb768f2244c8811c63729a21a3569731535f067ffc57839b00206d1ad20c69a1981b489f772031b279182d99e65703f0076e4812653aab85fca0f00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000),
GasLimit: 4700000,
Difficulty: big.NewInt(1),
Alloc: decodePrealloc(rinkebyAllocData),
}
}

// DevGenesisBlock returns the 'geth --dev' genesis block.
Func DevGenesisBlock() *Genesis {
Return &amp;Genesis{
Config: params.AllProtocolChanges,
Nonce: 42,
GasLimit: 4712388,
Difficulty: big.NewInt(131072),
Alloc: decodePrealloc(devAllocData),
}
}
</pre></body></html>