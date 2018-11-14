
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>The definition of the service in the node, eth is actually a service.

Type Service interface {
// Protocols retrieves the P2P protocols the service wishes to start.
Protocols() []p2p.Protocol

// APIs retrieves the list of RPC descriptors the service provides
APIs() []rpc.API

// Start is called after all services have been constructed and the networking
// layer was also initialized to spawn any goroutines required by the service.
Start(server *p2p.Server) error

// Stop terminates all goroutines belonging to the service, blocking until they
// are all terminated.
Stop() error
}

Go ethereum's eth directory is an implementation of the Ethereum service. The Ethereum protocol is injected through the Register method of node.


// RegisterEthService adds an Ethereum client to the stack.
Func RegisterEthService(stack *node.Node, cfg *eth.Config) {
Var err error
If cfg.SyncMode == downloader.LightSync {
Err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
Return les.New(ctx, cfg)
})
} else {
Err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
fullNode, err := eth.New(ctx, cfg)
If fullNode != nil &amp;&amp; cfg.LightServ &gt; 0 {
Ls, _ := les.NewLesServer(fullNode, cfg)
fullNode.AddLesServer(ls)
}
Return fullNode, err
})
}
If err != nil {
Fatalf("Failed to register the Ethereum service: %v", err)
}
}

Data structure of the Ethereum agreement

// Ethereum implements the Ethereum full node service.
Type Ethereum struct {
Config *Config configuration
chainConfig *params.ChainConfig chain configuration

// Channel for shutting down the service
shutdownChan chan bool // Channel for shutting down the ethereum
stopDbUpgrade func() error // stop chain db sequential key upgrade

// Handlers
txPool *core.TxPool trading pool
Blockchain *core.BlockChain blockchain
protocolManager *ProtocolManager protocol management
lesServer LesServer lightweight client server

// DB interfaces
chainDb ethdb.Database // Block chain database blockchain database

eventMux *event.TypeMux
Engine consensus.Engine consistency engine. Should be part of Pow
accountManager *accounts.Manager Account Management

bloomRequests chan chan *bloombits.Retrieval // Channel receiving bloom data retrieval requests Channel for receiving bloom filter data requests
bloomIndexer *core.ChainIndexer // Bloom indexer operating during block imports //Executing the Bloom indexer operation when the block is imported Temporarily unclear what it is

ApiBackend *EthApiBackend //Provided to the API backend used by the RPC service

Miner *miner.Miner // miners
gasPrice *big.Int // The minimum value of the gasPrice received by the node. Trades smaller than this value will be rejected by this node
Exchangebase common.Address //miner address

networkId uint64 //Network ID testnet is 0 mainnet is 1
netRPCService *ethapi.PublicNetAPI //RPC service

Lock sync.RWMutex // Protects the variadic fields (e.g. gas price and etherbase)
}

The creation of the Ethereum agreement New. Temporarily does not involve the content of the core. Just a general introduction. The contents of the core will be analyzed later.

// New creates a new Ethereum object (including the
// initialisation of the common Ethereum object)
Func New(ctx *node.ServiceContext, config *Config) (*Ethereum, error) {
If config.SyncMode == downloader.LightSync {
Return nil, errors.New("can't run eth.Ethereum in light sync mode, use les.LightEthereum")
}
If !config.SyncMode.IsValid() {
Return nil, fmt.Errorf("invalid sync mode %d", config.SyncMode)
}
// Create leveldb. Open or create a new chaindata directory
chainDb, err := CreateDB(ctx, config, "chaindata")
If err != nil {
Return nil, err
}
/ / Database format upgrade
stopDbUpgrade := upgradeDeduplicateData(chainDb)
// Set the creation block. If there is already a creation block in the database, then take it out of the database (private chain). Or get the default value from the code.
chainConfig, genesisHash, genesisErr := core.SetupGenesisBlock(chainDb, config.Genesis)
If _, ok := genesisErr.(*params.ConfigCompatError); genesisErr != nil &amp;&amp; !ok {
Return nil, genesisErr
}
log.Info("Initialised chain configuration", "config", chainConfig)

Eth := &amp;Ethereum{
Config: config,
chainDb: chainDb,
chainConfig: chainConfig,
eventMux: ctx.EventMux,
accountManager: ctx.AccountManager,
Engine: CreateConsensusEngine(ctx, config, chainConfig, chainDb), // Consistency Engine. Here I understand that it is Pow.
shutdownChan: make(chan bool),
stopDbUpgrade: stopDbUpgrade,
networkId: config.NetworkId, // The network ID is used to distinguish the network. The test network is 0. The main network is 1
gasPrice: config.GasPrice, // The gasprice minimum of the transaction that can be accepted by the --gasprice client. If it is less than this value, it will be discarded by the node.
Etherbase: config.Etherbase, // beneficiary of mining
bloomRequests: make(chan chan *bloombits.Retrieval), //bloom request
bloomIndexer: NewBloomIndexer(chainDb, params.BloomBitsBlocks),
}

log.Info("Initialising Ethereum protocol", "versions", ProtocolVersions, "network", config.NetworkId)

If !config.SkipBcVersionCheck { // Check if the version of BlockChainVersion stored in the database is the same as the version of BlockChainVersion of the client.
bcVersion := core.GetBlockChainVersion(chainDb)
If bcVersion != core.BlockChainVersion &amp;&amp; bcVersion != 0 {
Return nil, fmt.Errorf("Blockchain DB version mismatch (%d / %d). Run geth upgradedb.\n", bcVersion, core.BlockChainVersion)
}
core.WriteBlockChainVersion(chainDb, core.BlockChainVersion)
}

vmConfig := vm.Config{EnablePreimageRecording: config.EnablePreimageRecording}
// Created a blockchain using the database
Eth.blockchain, err = core.NewBlockChain(chainDb, eth.chainConfig, eth.engine, vmConfig)
If err != nil {
Return nil, err
}
// Rewind the chain in case of an incompatible config upgrade.
If compat, ok := genesisErr.(*params.ConfigCompatError); ok {
log.Warn("Rewinding chain to upgrade configuration", "err", compat)
eth.blockchain.SetHead(compat.RewindTo)
core.WriteChainConfig(chainDb, genesisHash, chainConfig)
}
// bloomIndexer I don’t know what it is. It’s not a lot involved. Leave it for the time being
eth.bloomIndexer.Start(eth.blockchain.CurrentHeader(), eth.blockchain.SubscribeChainEvent)

If config.TxPool.Journal != "" {
config.TxPool.Journal = ctx.ResolvePath(config.TxPool.Journal)
}
// Create a trading pool. Used to store transactions received locally or on the network.
eth.txPool = core.NewTxPool(config.TxPool, eth.chainConfig, eth.blockchain)
// Create a protocol manager
If eth.protocolManager, err = NewProtocolManager(eth.chainConfig, config.SyncMode, config.NetworkId, eth.eventMux, eth.txPool, eth.engine, eth.blockchain, chainDb); err != nil {
Return nil, err
}
// Create miners
Eth.miner = miner.New(eth, eth.chainConfig, eth.EventMux(), eth.engine)
eth.miner.SetExtra(makeExtraData(config.ExtraData))
// ApiBackend is used to provide backend support for RPC calls
eth.ApiBackend = &amp;EthApiBackend{eth, nil}
// gpoParams GPO Gas Price Abbreviation for Oracle. GasPrice forecast. The current value of GasPrice is predicted by the most recent transaction. This value can be used as a reference for the cost of sending a transaction later.
gpoParams := config.GPO
If gpoParams.Default == nil {
gpoParams.Default = config.GasPrice
}
eth.ApiBackend.gpo = gasprice.NewOracle(eth.ApiBackend, gpoParams)

Return eth, nil
}

ApiBackend is defined in the api_backend.go file. Some functions are encapsulated.

// EthApiBackend implements ethapi.Backend for full nodes
Type EthApiBackend struct {
Eth *Ethereum
Gpo *gasprice.Oracle
}
Func (b *EthApiBackend) SetHead(number uint64) {
b.eth.protocolManager.downloader.Cancel()
b.eth.blockchain.SetHead(number)
}

In addition to some methods in the core method, the Object method has a ProtocolManager object that is more important in the Ethereum protocol. Ethereum is originally an agreement. The ProtocolManager can manage multiple sub-protocols of Ethereum.


// NewProtocolManager returns a new ethereum sub protocol manager. The Ethereum sub protocol manages peers capable
// with the ethereum network.
Func NewProtocolManager(config *params.ChainConfig, mode downloader.SyncMode, networkId uint64, mux *event.TypeMux, txpool txPool, engine consensus.Engine, blockchain *core.BlockChain, chaindb ethdb.Database) (*ProtocolManager, error) {
// Create the protocol manager with the base fields
Manager := &amp;ProtocolManager{
networkId: networkId,
eventMux: mux,
Txpool: txpool,
Blockchain: blockchain,
Chaindb: chaindb,
Chainconfig: config,
Peers: newPeerSet(),
newPeerCh: make(chan *peer),
noMorePeers: make(chan struct{}),
txsyncCh: make(chan *txsync),
quitSync: make(chan struct{}),
}
// Figure out whether to allow fast sync or not
If mode == downloader.FastSync &amp;&amp; blockchain.CurrentBlock().NumberU64() &gt; 0 {
log.Warn("Blockchain not empty, fast sync disabled")
Mode = downloader.FullSync
}
If mode == downloader.FastSync {
manager.fastSync = uint32(1)
}
// Initiate a sub-protocol for every implemented version we can handle
manager.SubProtocols = make([]p2p.Protocol, 0, len(ProtocolVersions))
For i, version := range ProtocolVersions {
// Skip protocol version if incompatible with the mode of operation
If mode == downloader.FastSync &amp;&amp; version &lt; eth63 {
Continue
}
// Compatible; initialise the sub-protocol
Version := version // Closure for the run
manager.SubProtocols = append(manager.SubProtocols, p2p.Protocol{
Name: ProtocolName,
Version: version,
Length: ProtocolLengths[i],
// Remember the protocol in p2p. After the p2p peer connection succeeds, the Run method will be called.
Run: func(p *p2p.Peer, rw p2p.MsgReadWriter) error {
Peer := manager.newPeer(int(version), p, rw)
Select {
Case manager.newPeerCh &lt;- peer:
manager.wg.Add(1)
Defer manager.wg.Done()
Return manager.handle(peer)
Case &lt;-manager.quitSync:
Return p2p.DiscQuitting
}
},
NodeInfo: func() interface{} {
Return manager.NodeInfo()
},
PeerInfo: func(id discover.NodeID) interface{} {
If p := manager.peers.Peer(fmt.Sprintf("%x", id[:8])); p != nil {
Return p.Info()
}
Return nil
},
})
}
If len(manager.SubProtocols) == 0 {
Return nil, errIncompatibleConfig
}
// Construct the different synchronisation mechanisms
// The downloader is responsible for synchronizing its own data from other peers.
// downloader is a full chain synchronization tool
Manager.downloader = downloader.New(mode, chaindb, manager.eventMux, blockchain, nil, manager.removePeer)
// validator is a function that uses the consistency engine to verify the block header
Validator := func(header *types.Header) error {
Return engine.VerifyHeader(blockchain, header, true)
}
// function to return the height of the block
Heighter := func() uint64 {
Return blockchain.CurrentBlock().NumberU64()
}
// If fast sync is on. Then the inserter will not be called.
Inserter := func(blocks types.Blocks) (int, error) {
// If fast sync is running, deny importing weird blocks
If atomic.LoadUint32(&amp;manager.fastSync) == 1 {
log.Warn("Discarded bad propagated block", "number", blocks[0].Number(), "hash", blocks[0].Hash())
Return 0, nil
}
// set to start receiving transactions
atomic.StoreUint32(&amp;manager.acceptTxs, 1) // Mark initial sync done on any fetcher import
// insert block
Return manager.blockchain.InsertChain(blocks)
}
// Generate a fetcher
// Fetcher is responsible for accumulating block notifications from each peer and scheduling the retrieval.
Manager.fetcher = fetcher.New(blockchain.GetBlockByHash, validator, manager.BroadcastBlock, heighter, inserter, manager.removePeer)

Return manager, nil
}


The APIs() method of the service returns the RPC method exposed by the service.

// APIs returns the collection of RPC services the ethereum package offers.
// NOTE, some of these services probably need to be moved to somewhere else.
Func (s *Ethereum) APIs() []rpc.API {
Apis := ethapi.GetAPIs(s.ApiBackend)

// Append any APIs exposed explicitly by the consensus engine
Apis = append(apis, s.engine.APIs(s.BlockChain())...)

// Append all the local APIs and return
Return append(apis, []rpc.API{
{
Namespace: "eth",
Version: "1.0",
Service: NewPublicEthereumAPI(s),
Public: true,
},
...
, {
Namespace: "net",
Version: "1.0",
Service: s.netRPCService,
Public: true,
},
}...)
}

The Protocols method of the service will return the services provided by those p2p protocols. Return all SubProtocols in the Protocol Manager. If there is lesServer then provide the protocol of lesServer. can be seen. All network functions are provided through the protocol.

// Protocols implements node.Service, returning all the currently configured
// network protocols to start.
Func (s *Ethereum) Protocols() []p2p.Protocol {
If s.lesServer == nil {
Return s.protocolManager.SubProtocols
}
Return append(s.protocolManager.SubProtocols, s.lesServer.Protocols()...)
}


After the Ethereum service is created, the service's Start method is called. Let's take a look at the Start method.

// Start implements node.Service, starting all internal goroutines needed by the
// Ethereum protocol implementation.
Func (s *Ethereum) Start(srvr *p2p.Server) error {
// Start the bloom bits servicing goroutines
// Start the goroutine TODO of the Bloom filter request processing
s.startBloomHandlers()

// Start the RPC service
// Create a network API net
s.netRPCService = ethapi.NewPublicNetAPI(srvr, s.NetVersion())

// Figure out a max peers count based on the server limits
maxPeers := srvr.MaxPeers
If s.config.LightServ &gt; 0 {
maxPeers -= s.config.LightPeers
If maxPeers &lt; srvr.MaxPeers/2 {
maxPeers = srvr.MaxPeers / 2
}
}
// Start the networking layer and the light server if requested
/ / Start the protocol manager
s.protocolManager.Start(maxPeers)
If s.lesServer != nil {
// If lesServer does not start it for nil.
s.lesServer.Start(srvr)
}
Return nil
}

Protocol Manager data structure

Type ProtocolManager struct {
networkId uint64

fastSync uint32 // Flag whether fast sync is enabled (gets disabled if we already have blocks)
acceptTxs uint32 // Flag whether we're considered synchronised (enables transaction processing)

Txpool txPool
Blockchain *core.BlockChain
Chaindb ethdb.Database
Chainconfig *params.ChainConfig
maxPeers int

Downloader *downloader.Downloader
Fetcher *fetcher.Fetcher
Peers *peerSet

SubProtocols []p2p.Protocol

eventMux *event.TypeMux
txCh chan core.TxPreEvent
txSub event.Subscription
minedBlockSub *event.TypeMuxSubscription

// channels for fetcher, syncer, txsyncLoop
newPeerCh chan *peer
txsyncCh chan *txsync
quitSync chan struct{}
noMorePeers chan struct{}

// wait group is used for graceful shutdowns during downloading
// and processing
Wg sync.WaitGroup
}

The Start method of the protocol manager. This method starts a large number of goroutines to handle various transactions. It can be speculated that this class should be the main implementation class of the Ethereum service.

Func (pm *ProtocolManager) Start(maxPeers int) {
pm.maxPeers = maxPeers

// broadcast transactions
// The channel for the broadcast transaction. txCh will serve as the TxPreEvent subscription channel for txpool. Txpool will notify this txCh with this message. The goroutine of the broadcast transaction will broadcast this message.
Pm.txCh = make(chan core.TxPreEvent, txChanSize)
// Subscribed receipt
pm.txSub = pm.txpool.SubscribeTxPreEvent(pm.txCh)
// Start the broadcast goroutine
Go pm.txBroadcastLoop()

// broadcast mined blocks
// Subscribe to mining news. A message is generated when the new block is dug up. This subscription and the subscription above use two different modes, this is the subscription method labeled Deprecated.
pm.minedBlockSub = pm.eventMux.Subscribe(core.NewMinedBlockEvent{})
// Mining broadcast goroutine When you dig out, you need to broadcast it to the network as soon as possible.
Go pm.minedBroadcastLoop()

// start sync handlers
// The synchronizer is responsible for periodically synchronizing with the network, downloading hashes and blocks, and processing notification handlers.
Go pm.syncer()
// txsyncLoop is responsible for the initial transaction synchronization for each new connection. When a new peer appears, we forward all pending transactions. In order to minimize the use of export bandwidth, we only send one packet at a time.
Go pm.txsyncLoop()
}


When the server of p2p is started, it will actively find the node to connect, or be connected by other nodes. The process of connecting is to first perform a handshake of the encrypted channel and then perform a handshake of the protocol. Finally, the goroutine is executed for each protocol to execute the Run method to give control to the final protocol. This run method first creates a peer object, then calls the handle method to handle the peer.

Run: func(p *p2p.Peer, rw p2p.MsgReadWriter) error {
Peer := manager.newPeer(int(version), p, rw)
Select {
Case manager.newPeerCh &lt;- peer: //Send the peer to the newPeerCh channel
manager.wg.Add(1)
Defer manager.wg.Done()
Return manager.handle(peer) // call the handle method
Case &lt;-manager.quitSync:
Return p2p.DiscQuitting
}
},


Handle method,


// handle is the callback invoked to manage the life cycle of an eth peer. When
// this function terminates, the peer is disconnected.
// handle is a callback method for managing the lifecycle management of eth peers. When this method exits, the peer connection will also be broken.
Func (pm *ProtocolManager) handle(p *peer) error {
If pm.peers.Len() &gt;= pm.maxPeers {
Return p2p.DiscTooManyPeers
}
p.Log().Debug("Ethereum peer connected", "name", p.Name())

// Execute the Ethereum handshake
Td, head, genesis := pm.blockchain.Status()
// td is total difficult, head is the current block header, and genesis is the information of the creation block. Only the creation block is the same to be able to shake hands successfully.
If err := p.Handshake(pm.networkId, td, head, genesis); err != nil {
p.Log().Debug("Ethereum handshake failed", "err", err)
Return err
}
If rw, ok := p.rw.(*meteredMsgReadWriter); ok {
rw.Init(p.version)
}
// Register the peer locally
// register the peer to the local
If err := pm.peers.Register(p); err != nil {
p.Log().Error("Ethereum peer registration failed", "err", err)
Return err
}
Defer pm.removePeer(p.id)

// Register the peer in the downloader. If the downloader considers it banned, we disconnect
// Register the peer to the downloader. If the downloader thinks the peer is banned, then disconnect.
If err := pm.downloader.RegisterPeer(p.id, p.version, p); err != nil {
Return err
}
// Propagate existing transactions. new transactions appearing
// after this will be sent via broadcasts.
// Send the current pending transaction to the other party, this only happens when the connection is just established.
pm.syncTransactions(p)

// If we're DAO hard-fork aware, validate any remote peer with regard to the hard-fork
/ / Verify the peer's DAO hard fork
If daoBlock := pm.chainconfig.DAOForkBlock; daoBlock != nil {
// Request the peer's DAO fork header for extra-data validation
If err := p.RequestHeadersByNumber(daoBlock.Uint64(), 1, 0, false); err != nil {
Return err
}
// Start a timer to disconnect if the peer doesn't reply in time
// If no response is received within 15 seconds. Then disconnect.
p.forkDrop = time.AfterFunc(daoChallengeTimeout, func() {
p.Log().Debug("Timed out DAO fork-check, dropping")
pm.removePeer(p.id)
})
// Make sure it's cleaned up if the peer dies off
Defer func() {
If p.forkDrop != nil {
p.forkDrop.Stop()
p.forkDrop = nil
}
}()
}
// main loop. handle incoming messages.
// Main loop. Process incoming messages.
For {
If err := pm.handleMsg(p); err != nil {
p.Log().Debug("Ethereum message handling failed", "err", err)
Return err
}
}
}


Handshake

// Handshake executes the eth protocol handshake, negotiating version number,
// network IDs, difficulties, head and genesis blocks.
Func (p *peer) Handshake(network uint64, td *big.Int, head common.Hash, genesis common.Hash) error {
// Send out own handshake in a new thread
// The size of the error channel is 2, in order to process the following two goroutine methods at once.
Errc := make(chan error, 2)
Var status statusData // safe to read after two values ​​have been received from errc

Go func() {
Errc &lt;- p2p.Send(p.rw, StatusMsg, &amp;statusData{
ProtocolVersion: uint32(p.version),
NetworkId: network,
TD: td,
CurrentBlock: head,
GenesisBlock: genesis,
})
}()
Go func() {
Errc &lt;- p.readStatus(network, &amp;status, genesis)
}()
Timeout := time.NewTimer(handshakeTimeout)
Defer timeout.Stop()
// If you receive any error (send, receive), or if it times out, then disconnect.
For i := 0; i &lt; 2; i++ {
Select {
Case err := &lt;-errc:
If err != nil {
Return err
}
Case &lt;-timeout.C:
Return p2p.DiscReadTimeout
}
}
P.td, p.head = status.TD, status.CurrentBlock
Return nil
}

readStatus, check the various situations returned by the peer,

Func (p *peer) readStatus(network uint64, status *statusData, genesis common.Hash) (err error) {
Msg, err := p.rw.ReadMsg()
If err != nil {
Return err
}
If msg.Code != StatusMsg {
Return errResp(ErrNoStatusMsg, "first msg has code %x (!= %x)", msg.Code, StatusMsg)
}
If msg.Size &gt; ProtocolMaxMsgSize {
Return errResp(ErrMsgTooLarge, "%v &gt; %v", msg.Size, ProtocolMaxMsgSize)
}
// Decode the handshake and make sure everything matches
If err := msg.Decode(&amp;status); err != nil {
Return errResp(ErrDecode, "msg %v: %v", msg, err)
}
If status.GenesisBlock != genesis {
Return errResp(ErrGenesisBlockMismatch, "%x (!= %x)", status.GenesisBlock[:8], genesis[:8])
}
If status.NetworkId != network {
Return errResp(ErrNetworkIdMismatch, "%d (!= %d)", status.NetworkId, network)
}
If int(status.ProtocolVersion) != p.version {
Return errResp(ErrProtocolVersionMismatch, "%d (!= %d)", status.ProtocolVersion, p.version)
}
Return nil
}

Register simply add the peer to your peer's map

// Register injects a new peer into the working set, or returns an error if the
// peer is already known.
Func (ps *peerSet) Register(p *peer) error {
ps.lock.Lock()
Defer ps.lock.Unlock()

If ps.closed {
Return errClosed
}
If _, ok := ps.peers[p.id]; ok {
Return errAlreadyRegistered
}
Ps.peers[p.id] = p
Return nil
}


After a series of checks and handshakes, the loop calls the handleMsg method to handle the event loop. This method is very long, mainly to deal with the response after receiving various messages.

// handleMsg is invoked whenever an inbound message is received from a remote
// peer. The remote connection is turn down upon returning any error.
Func (pm *ProtocolManager) handleMsg(p *peer) error {
// Read the next message from the remote peer, and ensure it's fully consumed
Msg, err := p.rw.ReadMsg()
If err != nil {
Return err
}
If msg.Size &gt; ProtocolMaxMsgSize {
Return errResp(ErrMsgTooLarge, "%v &gt; %v", msg.Size, ProtocolMaxMsgSize)
}
Defer msg.Discard()

// Handle the message depending on its contents
Switch {
Case msg.Code == StatusMsg:
// Status messages should never arrive after the handshake
// StatusMsg should be received during the HandleShake phase. After HandleShake, you should not receive such a message.
Return errResp(ErrExtraStatusMsg, "uncontrolled status message")

// Block header query, collect the requested headers and reply
// Upon receiving the message requesting the header of the block, it will return the block header information according to the request.
Case msg.Code == GetBlockHeadersMsg:
// Decode the complex header query
Var query getBlockHeadersData
If err := msg.Decode(&amp;query); err != nil {
Return errResp(ErrDecode, "%v: %v", msg, err)
}
hashMode := query.Origin.Hash != (common.Hash{})

// Gather headers until the fetch or network limits is reached
Var (
Bytes common.StorageSize
Headers []*types.Header
Unknown bool
)
For !unknown &amp;&amp; len(headers) &lt; int(query.Amount) &amp;&amp; bytes &lt; softResponseLimit &amp;&amp; len(headers) &lt; downloader.MaxHeaderFetch {
// Retrieve the next header satisfying the query
Var origin *types.Header
If hashMode {
Origin = pm.blockchain.GetHeaderByHash(query.Origin.Hash)
} else {
Origin = pm.blockchain.GetHeaderByNumber(query.Origin.Number)
}
If origin == nil {
Break
}
Number := origin.Number.Uint64()
Headers = append(headers, origin)
Bytes += estHeaderRlpSize

// Advance to the next header of the query
Switch {
Case query.Origin.Hash != (common.Hash{}) &amp;&amp; query.Reverse:
// Hash based traversal towards the genesis block
// Move to the creation block from the beginning of the Hash designation. That is the reverse movement. Find by hash
For i := 0; i &lt; int(query.Skip)+1; i++ {
If header := pm.blockchain.GetHeader(query.Origin.Hash, number); header != nil {// Get the previous block header by hash and number

query.Origin.Hash = header.ParentHash
Number--
} else {
Unknown = true
Break //break is to jump out of the switch. Unknow is used to jump out of the loop.
}
}
Case query.Origin.Hash != (common.Hash{}) &amp;&amp; !query.Reverse:
// Hash based traversal towards the leaf block
// find by hash
Var (
Current = origin.Number.Uint64()
Next = current + query.Skip + 1
)
If next &lt;= current { // forward, but next is smaller than the current, against integer overflow attacks.
Infos, _ := json.MarshalIndent(p.Peer.Info(), "", " ")
p.Log().Warn("GetBlockHeaders skip overflow attack", "current", current, "skip", query.Skip, "next", next, "attacker", infos)
Unknown = true
} else {
If header := pm.blockchain.GetHeaderByNumber(next); header != nil {
If pm.blockchain.GetBlockHashesFromHash(header.Hash(), query.Skip+1)[query.Skip] == query.Origin.Hash {
// If this header can be found, and this header and origin are on the same chain.
query.Origin.Hash = header.Hash()
} else {
Unknown = true
}
} else {
Unknown = true
}
}
Case query.Reverse: // find by number
// Number based traversal towards the genesis block
// query.Origin.Hash == (common.Hash{})
If query.Origin.Number &gt;= query.Skip+1 {
query.Origin.Number -= (query.Skip + 1)
} else {
Unknown = true
}

Case !query.Reverse: // find by number
// Number based traversal towards the leaf block
query.Origin.Number += (query.Skip + 1)
}
}
Return p.SendBlockHeaders(headers)

Case msg.Code == BlockHeadersMsg: //Received the answer to GetBlockHeadersMsg.
// A batch of headers arrived to one of our previous requests
Var headers []*types.Header
If err := msg.Decode(&amp;headers); err != nil {
Return errResp(ErrDecode, "msg %v: %v", msg, err)
}
// If no headers were received, but we're expending a DAO fork check, maybe it's that
// If the peer does not return any headers, and forkDrop is not empty, then it should be our DAO check request, we have previously sent a DAO header request in HandShake.
If len(headers) == 0 &amp;&amp; p.forkDrop != nil {
// Possibly an empty reply to the fork header checks, sanity check TDs
verifyDAO := true

// If we already have a DAO header, we can check the peer's TD against it. If
// the peer's ahead of this, it too must have a reply to the DAO check
If daoHeader := pm.blockchain.GetHeaderByNumber(pm.chainconfig.DAOForkBlock.Uint64()); daoHeader != nil {
If _, td := p.Head(); td.Cmp(pm.blockchain.GetTd(daoHeader.Hash(), daoHeader.Number.Uint64())) &gt;= 0 {
/ / This time check whether the peer's total difficult has exceeded the td value of the DAO fork block, if it exceeds, it means that the peer should have this block header, but the returned blank, then the verification fails. Nothing is done here. If the peer does not send it, it will be timed out.
verifyDAO = false
}
}
// If we're seemingly on the same chain, disable the drop timer
If verifyDAO { // If the verification is successful, delete the timer and return.
p.Log().Debug("Seems to be on the same side of the DAO fork")
p.forkDrop.Stop()
p.forkDrop = nil
Return nil
}
}
// Filter out any explicitly requested headers, deliver the rest to the downloader
// Filter out any very specific requests, then post the rest to the downloader
// If the length is 1, then filter is true
Filter := len(headers) == 1
If filter {
// If it's a potential DAO fork check, validate against the rules
If p.forkDrop != nil &amp;&amp; pm.chainconfig.DAOForkBlock.Cmp(headers[0].Number) == 0 { //DAO check
// Disable the fork drop timer
p.forkDrop.Stop()
p.forkDrop = nil

// Validate the header and either drop the peer or continue
If err := misc.VerifyDAOHeaderExtraData(pm.chainconfig, headers[0]); err != nil {
p.Log().Debug("Verified to be on the other side of the DAO fork, dropping")
Return err
}
p.Log().Debug("Verified to be on the same side of the DAO fork")
Return nil
}
// Irrelevant of the fork checks, send the header to the fetcher just in case
// If it is not a DAO request, pass it to the filter for filtering. The filter will return the headers that need to be processed, and these headers will be handed over to the downloader for distribution.
Headers = pm.fetcher.FilterHeaders(p.id, headers, time.Now())
}
If len(headers) &gt; 0 || !filter {
Err := pm.downloader.DeliverHeaders(p.id, headers)
If err != nil {
log.Debug("Failed to deliver headers", "err", err)
}
}

Case msg.Code == GetBlockBodiesMsg:
// Block Body request This is relatively simple. Get the body return from the blockchain.
// Decode the retrieval message
msgStream := rlp.NewStream(msg.Payload, uint64(msg.Size))
If _, err := msgStream.List(); err != nil {
Return err
}
// Gather blocks until the fetch or network limits is reached
Var (
Hash common.Hash
Bytes int
Bodies []rlp.RawValue
)
For bytes &lt; softResponseLimit &amp;&amp; len(bodies) &lt; downloader.MaxBlockFetch {
// Retrieve the hash of the next block
If err := msgStream.Decode(&amp;hash); err == rlp.EOL {
Break
} else if err != nil {
Return errResp(ErrDecode, "msg %v: %v", msg, err)
}
//Retrieve the requested block body, stopping from enough found found
If data := pm.blockchain.GetBodyRLP(hash); len(data) != 0 {
Bodies = append(bodies, data)
Bytes += len(data)
}
}
Return p.SendBlockBodiesRLP(bodies)

Case msg.Code == BlockBodiesMsg:
// A batch of block bodies arrived to one of our previous requests
Var request blockBodiesData
If err := msg.Decode(&amp;request); err != nil {
Return errResp(ErrDecode, "msg %v: %v", msg, err)
}
// Deliver them all to the downloader for queuing
Trasactions := make([][]*types.Transaction, len(request))
Uncles := make([][]*types.Header, len(request))

For i, body := range request {
Trasactions[i] = body.Transactions
Uncles[i] = body.Uncles
}
// Filter out any explicitly requested bodies, deliver the rest to the downloader
// Filter out any displayed requests, leaving the rest to the downloader
Filter := len(trasactions) &gt; 0 || len(uncles) &gt; 0
If filter {
Trasactions, uncles = pm.fetcher.FilterBodies(p.id, trasactions, uncles, time.Now())
}
If len(trasactions) &gt; 0 || len(uncles) &gt; 0 || !filter {
Err := pm.downloader.DeliverBodies(p.id, trasactions, uncles)
If err != nil {
log.Debug("Failed to deliver bodies", "err", err)
}
}

Case p.version &gt;= eth63 &amp;&amp; msg.Code == GetNodeDataMsg:
// The version of the peer is eth63 and is requesting NodeData
// Decode the retrieval message
msgStream := rlp.NewStream(msg.Payload, uint64(msg.Size))
If _, err := msgStream.List(); err != nil {
Return err
}
// Gather state data until the fetch or network limits is reached
Var (
Hash common.Hash
Bytes int
Data [][]byte
)
For bytes &lt; softResponseLimit &amp;&amp; len(data) &lt; downloader.MaxStateFetch {
// Retrieve the hash of the next state entry
If err := msgStream.Decode(&amp;hash); err == rlp.EOL {
Break
} else if err != nil {
Return errResp(ErrDecode, "msg %v: %v", msg, err)
}
//Retrieve the requested state entry, stopping from enough found
				// 请求的任何hash值都会返回给对方。 
				if entry, err := pm.chaindb.Get(hash.Bytes()); err == nil {
					data = append(data, entry)
					bytes += len(entry)
}
}
			return p.SendNodeData(data)

		case p.version &gt;= eth63 &amp;&amp; msg.Code == NodeDataMsg:
			// A batch of node state data arrived to one of our previous requests
			var data [][]byte
			if err := msg.Decode(&amp;data); err != nil {
				return errResp(ErrDecode, "msg %v: %v", msg, err)
}
			// Deliver all to the downloader
			// 数据交给downloader
			if err := pm.downloader.DeliverNodeData(p.id, data); err != nil {
				log.Debug("Failed to deliver node state data", "err", err)
}

		case p.version &gt;= eth63 &amp;&amp; msg.Code == GetReceiptsMsg:
			// 请求收据
			// Decode the retrieval message
			msgStream := rlp.NewStream(msg.Payload, uint64(msg.Size))
			if _, err := msgStream.List(); err != nil {
Return err
}
			// Gather state data until the fetch or network limits is reached
Var (
				hash     common.Hash
				bytes    int
				receipts []rlp.RawValue
)
			for bytes &lt; softResponseLimit &amp;&amp; len(receipts) &lt; downloader.MaxReceiptFetch {
				// Retrieve the hash of the next block
				if err := msgStream.Decode(&amp;hash); err == rlp.EOL {
Break
				} else if err != nil {
					return errResp(ErrDecode, "msg %v: %v", msg, err)
}
				// Retrieve the requested block's receipts, skipping if unknown to us
				results := core.GetBlockReceipts(pm.chaindb, hash, core.GetBlockNumber(pm.chaindb, hash))
				if results == nil {
					if header := pm.blockchain.GetHeaderByHash(hash); header == nil || header.ReceiptHash != types.EmptyRootHash {
Continue
}
}
				// If known, encode and queue for response packet
				if encoded, err := rlp.EncodeToBytes(results); err != nil {
					log.Error("Failed to encode receipt", "err", err)
				} else {
					receipts = append(receipts, encoded)
					bytes += len(encoded)
}
}
			return p.SendReceiptsRLP(receipts)

		case p.version &gt;= eth63 &amp;&amp; msg.Code == ReceiptsMsg:
			// A batch of receipts arrived to one of our previous requests
			var receipts [][]*types.Receipt
			if err := msg.Decode(&amp;receipts); err != nil {
				return errResp(ErrDecode, "msg %v: %v", msg, err)
}
			// Deliver all to the downloader
			if err := pm.downloader.DeliverReceipts(p.id, receipts); err != nil {
				log.Debug("Failed to deliver receipts", "err", err)
}

		case msg.Code == NewBlockHashesMsg:
			// 接收到BlockHashesMsg消息
			var announces newBlockHashesData
			if err := msg.Decode(&amp;announces); err != nil {
				return errResp(ErrDecode, "%v: %v", msg, err)
}
			// Mark the hashes as present at the remote node
			for _, block := range announces {
				p.MarkBlock(block.Hash)
}
			// Schedule all the unknown hashes for retrieval
			unknown := make(newBlockHashesData, 0, len(announces))
			for _, block := range announces {
				if !pm.blockchain.HasBlock(block.Hash, block.Number) {
					unknown = append(unknown, block)
}
}
			for _, block := range unknown {
				// 通知fetcher有一个潜在的block需要下载
				pm.fetcher.Notify(p.id, block.Hash, block.Number, time.Now(), p.RequestOneHeader, p.RequestBodies)
}

		case msg.Code == NewBlockMsg:
			// Retrieve and decode the propagated block
			var request newBlockData
			if err := msg.Decode(&amp;request); err != nil {
				return errResp(ErrDecode, "%v: %v", msg, err)
}
			request.Block.ReceivedAt = msg.ReceivedAt
			request.Block.ReceivedFrom = p

			// Mark the peer as owning the block and schedule it for import
			p.MarkBlock(request.Block.Hash())
			pm.fetcher.Enqueue(p.id, request.Block)

			// Assuming the block is importable by the peer, but possibly not yet done so,
			// calculate the head hash and TD that the peer truly must have.
Var (
				trueHead = request.Block.ParentHash()
				trueTD   = new(big.Int).Sub(request.TD, request.Block.Difficulty())
)
			// Update the peers total difficulty if better than the previous
			if _, td := p.Head(); trueTD.Cmp(td) &gt; 0 {
				// 如果peer的真实的TD和head和我们这边记载的不同， 设置peer真实的head和td，
				p.SetHead(trueHead, trueTD)

				// Schedule a sync if above ours. Note, this will not fire a sync for a gap of
				// a singe block (as the true TD is below the propagated block), however this
				// scenario should easily be covered by the fetcher.
				// 如果真实的TD比我们的TD大，那么请求和这个peer同步。
				currentBlock := pm.blockchain.CurrentBlock()
				if trueTD.Cmp(pm.blockchain.GetTd(currentBlock.Hash(), currentBlock.NumberU64())) &gt; 0 {
					go pm.synchronise(p)
}
}

		case msg.Code == TxMsg:
			// Transactions arrived, make sure we have a valid and fresh chain to handle them
			// 交易信息返回。 在我们没用同步完成之前不会接收交易信息。
			if atomic.LoadUint32(&amp;pm.acceptTxs) == 0 {
Break
}
			// Transactions can be processed, parse all of them and deliver to the pool
			var txs []*types.Transaction
			if err := msg.Decode(&amp;txs); err != nil {
				return errResp(ErrDecode, "msg %v: %v", msg, err)
}
			for i, tx := range txs {
				// Validate and mark the remote transaction
				if tx == nil {
					return errResp(ErrDecode, "transaction %d is nil", i)
}
				p.MarkTransaction(tx.Hash())
}
			// 添加到txpool
			pm.txpool.AddRemotes(txs)

Default:
			return errResp(ErrInvalidMsgCode, "%v", msg.Code)
}
Return nil
}

几种同步synchronise, 之前发现对方的节点比自己节点要更新的时候会调用这个方法synchronise，


	// synchronise tries to sync up our local block chain with a remote peer.
	// synchronise 尝试 让本地区块链跟远端同步。
	func (pm *ProtocolManager) synchronise(peer *peer) {
		// Short circuit if no peers are available
		if peer == nil {
Return
}
		// Make sure the peer's TD is higher than our own
		currentBlock := pm.blockchain.CurrentBlock()
		td := pm.blockchain.GetTd(currentBlock.Hash(), currentBlock.NumberU64())

		pHead, pTd := peer.Head()
		if pTd.Cmp(td) &lt;= 0 {
Return
}
		// Otherwise try to sync with the downloader
		mode := downloader.FullSync
		if atomic.LoadUint32(&amp;pm.fastSync) == 1 { //如果显式申明是fast
			// Fast sync was explicitly requested, and explicitly granted
			mode = downloader.FastSync
		} else if currentBlock.NumberU64() == 0 &amp;&amp; pm.blockchain.CurrentFastBlock().NumberU64() &gt; 0 {  //如果数据库是空白的
			// The database seems empty as the current block is the genesis. Yet the fast
			// block is ahead, so fast sync was enabled for this node at a certain point.
			// The only scenario where this can happen is if the user manually (or via a
			// bad block) rolled back a fast sync node below the sync point. In this case
			// however it's safe to reenable fast sync.
			atomic.StoreUint32(&amp;pm.fastSync, 1)
			mode = downloader.FastSync
}
		// Run the sync cycle, and disable fast sync if we've went past the pivot block
		err := pm.downloader.Synchronise(peer.id, pHead, pTd, mode)

		if atomic.LoadUint32(&amp;pm.fastSync) == 1 {
			// Disable fast sync if we indeed have something in our chain
			if pm.blockchain.CurrentBlock().NumberU64() &gt; 0 {
				log.Info("Fast sync complete, auto disabling")
				atomic.StoreUint32(&amp;pm.fastSync, 0)
}
}
If err != nil {
Return
}
		atomic.StoreUint32(&amp;pm.acceptTxs, 1) // Mark initial sync done
		// 同步完成 开始接收交易。
		if head := pm.blockchain.CurrentBlock(); head.NumberU64() &gt; 0 {
			// We've completed a sync cycle, notify all peers of new state. This path is
			// essential in star-topology networks where a gateway node needs to notify
			// all its out-of-date peers of the availability of a new block. This failure
			// scenario will most often crop up in private and hackathon networks with
			// degenerate connectivity, but it should be healthy for the mainnet too to
			// more reliably update peers or the local TD state.
			// 我们告诉所有的peer我们的状态。
			go pm.BroadcastBlock(head, false)
}
}


交易广播。txBroadcastLoop 在start的时候启动的goroutine。  txCh在txpool接收到一条合法的交易的时候会往这个上面写入事件。 然后把交易广播给所有的peers

	func (self *ProtocolManager) txBroadcastLoop() {
For {
Select {
			case event := &lt;-self.txCh:
				self.BroadcastTx(event.Tx.Hash(), event.Tx)

			// Err() channel will be closed when unsubscribing.
			case &lt;-self.txSub.Err():
Return
}
}
}


挖矿广播。当收到订阅的事件的时候把新挖到的矿广播出去。

	// Mined broadcast loop
	func (self *ProtocolManager) minedBroadcastLoop() {
		// automatically stops if unsubscribe
		for obj := range self.minedBlockSub.Chan() {
			switch ev := obj.Data.(type) {
			case core.NewMinedBlockEvent:
				self.BroadcastBlock(ev.Block, true)  // First propagate block to peers
				self.BroadcastBlock(ev.Block, false) // Only then announce to the rest
}
}
}

syncer负责定期和网络同步，

	// syncer is responsible for periodically synchronising with the network, both
	// downloading hashes and blocks as well as handling the announcement handler.
	//同步器负责周期性地与网络同步，下载散列和块以及处理通知处理程序。
	func (pm *ProtocolManager) syncer() {
		// Start and ensure cleanup of sync mechanisms
		pm.fetcher.Start()
		defer pm.fetcher.Stop()
		defer pm.downloader.Terminate()

		// Wait for different events to fire synchronisation operations
		forceSync := time.NewTicker(forceSyncCycle)
		defer forceSync.Stop()

For {
Select {
			case &lt;-pm.newPeerCh: //当有新的Peer增加的时候 会同步。 这个时候还可能触发区块广播。
				// Make sure we have peers to select from, then sync
				if pm.peers.Len() &lt; minDesiredPeerCount {
Break
}
				go pm.synchronise(pm.peers.BestPeer())

			case &lt;-forceSync.C:
				// 定时触发 10秒一次
				// Force a sync even if not enough peers are present
				// BestPeer() 选择总难度最大的节点。
				go pm.synchronise(pm.peers.BestPeer())

			case &lt;-pm.noMorePeers: // 退出信号
Return
}
}
}

txsyncLoop负责把pending的交易发送给新建立的连接。


	// txsyncLoop takes care of the initial transaction sync for each new
	// connection. When a new peer appears, we relay all currently pending
	// transactions. In order to minimise egress bandwidth usage, we send
	// the transactions in small packs to one peer at a time.

	txsyncLoop负责每个新连接的初始事务同步。 当新的对等体出现时，我们转发所有当前待处理的事务。 为了最小化出口带宽使用，我们一次将一个小包中的事务发送给一个对等体。
	func (pm *ProtocolManager) txsyncLoop() {
Var (
			pending = make(map[discover.NodeID]*txsync)
			sending = false               // whether a send is active
			pack    = new(txsync)         // the pack that is being sent
			done    = make(chan error, 1) // result of the send
)

		// send starts a sending a pack of transactions from the sync.
		send := func(s *txsync) {
			// Fill pack with transactions up to the target size.
			size := common.StorageSize(0)
			pack.p = s.p
			pack.txs = pack.txs[:0]
			for i := 0; i &lt; len(s.txs) &amp;&amp; size &lt; txsyncPackSize; i++ {
				pack.txs = append(pack.txs, s.txs[i])
				size += s.txs[i].Size()
}
			// Remove the transactions that will be sent.
			s.txs = s.txs[:copy(s.txs, s.txs[len(pack.txs):])]
			if len(s.txs) == 0 {
				delete(pending, s.p.ID())
}
			// Send the pack in the background.
			s.p.Log().Trace("Sending batch of transactions", "count", len(pack.txs), "bytes", size)
			sending = true
			go func() { done &lt;- pack.p.SendTransactions(pack.txs) }()
}

		// pick chooses the next pending sync.
		// 随机挑选一个txsync来发送。
		pick := func() *txsync {
			if len(pending) == 0 {
Return nil
}
			n := rand.Intn(len(pending)) + 1
			for _, s := range pending {
				if n--; n == 0 {
					return s
}
}
Return nil
}

For {
Select {
			case s := &lt;-pm.txsyncCh: //从这里接收txsyncCh消息。
				pending[s.p.ID()] = s
				if !sending {
					send(s)
}
			case err := &lt;-done:
				sending = false
				// Stop tracking peers that cause send failures.
If err != nil {
					pack.p.Log().Debug("Transaction send failed", "err", err)
					delete(pending, pack.p.ID())
}
				// Schedule the next send.
				if s := pick(); s != nil {
					send(s)
}
			case &lt;-pm.quitSync:
Return
}
}
}

txsyncCh队列的生产者，syncTransactions是在handle方法里面调用的。 在新链接刚刚创建的时候会被调用一次。

	// syncTransactions starts sending all currently pending transactions to the given peer.
	func (pm *ProtocolManager) syncTransactions(p *peer) {
		var txs types.Transactions
		pending, _ := pm.txpool.Pending()
		for _, batch := range pending {
			txs = append(txs, batch...)
}
		if len(txs) == 0 {
Return
}
Select {
		case pm.txsyncCh &lt;- &amp;txsync{p, txs}:
		case &lt;-pm.quitSync:
}
}


总结一下。 我们现在的一些大的流程。

区块同步

1. 如果是自己挖的矿。通过goroutine minedBroadcastLoop()来进行广播。
2. 如果是接收到其他人的区块广播，(NewBlockHashesMsg/NewBlockMsg),是否fetcher会通知的peer？ TODO
3. goroutine syncer()中会定时的同BestPeer()来同步信息。

交易同步

1. 新建立连接。 把pending的交易发送给他。
2. 本地发送了一个交易，或者是接收到别人发来的交易信息。 txpool会产生一条消息，消息被传递到txCh通道。 然后被goroutine txBroadcastLoop()处理， 发送给其他不知道这个交易的peer。</pre></body></html>