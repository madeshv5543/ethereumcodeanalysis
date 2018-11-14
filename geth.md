
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Geth is our main command line tool for go-ethereum. It is also the access point for our various networks (main network main-net test network test-net and private network). Supports running in full node mode or lightweight node mode. Other programs can access the functionality of the Ethereum network through its exposed JSON RPC calls.

If you do not enter any commands, run geoth directly. A node in full node mode is started by default. Connect to the main network. Let's take a look at what the main process of startup is and what components are involved.


## Main function started cmd/geth/main.go
When you see the main function, it runs directly. It was a bit aggressive at first. Later found that there are two default functions in the go language, one is the main () function. One is the init() function. The go language will automatically call the init() function of all packages first in a certain order. Then the main() function is called.

Func main() {
If err := app.Run(os.Args); err != nil {
fmt.Fprintln(os.Stderr, err)
os.Exit(1)
}
}


Main.go init function
App is an example of a three-way package gopkg.in/urfave/cli.v1. The usage of this three-way package is roughly the first to construct this app object. Provide some callback functions by configuring the behavior of the app object through code. Then run app.Run(os.Args) directly in the main function.

Import (
...
"gopkg.in/urfave/cli.v1"
)

Var (

App = utils.NewApp(gitCommit, "the go-ethereum command line interface")
//flag that configure the node
nodeFlags = []cli.Flag{
utils.IdentityFlag,
utils.UnlockedAccountFlag,
utils.PasswordFileFlag,
utils.BootnodesFlag,
...
}

rpcFlags = []cli.Flag{
utils.RPCEnabledFlag,
utils.RPCListenAddrFlag,
...
}

whisperFlags = []cli.Flag{
utils.WhisperEnabledFlag,
...
}
)
Func init() {
// Initialize the CLI app and start Geth
// The Action field indicates that if the user does not enter another subcommand, the function pointed to by this field will be called.
app.Action = geth
app.HideVersion = true // we have a command to print the version
app.Copyright = "Copyright 2013-2017 The go-ethereum Authors"
// Commands are all supported subcommands
app.Commands = []cli.Command{
// See chaincmd.go:
initCommand,
importCommand,
exportCommand,
removedbCommand,
dumpCommand,
// See monitorcmd.go:
monitorCommand,
// See accountcmd.go:
accountCommand,
walletCommand,
// See consolecmd.go:
consoleCommand,
attachCommand,
javascriptCommand,
// See misccmd.go:
makecacheCommand,
makedagCommand,
versionCommand,
bugCommand,
licenseCommand,
// See config.go
dumpConfigCommand,
}
sort.Sort(cli.CommandsByName(app.Commands))
// All options that can be parsed
app.Flags = append(app.Flags, nodeFlags...)
app.Flags = append(app.Flags, rpcFlags...)
app.Flags = append(app.Flags, consoleFlags...)
app.Flags = append(app.Flags, debug.Flags...)
app.Flags = append(app.Flags, whisperFlags...)

app.Before = func(ctx *cli.Context) error {
runtime.GOMAXPROCS(runtime.NumCPU())
If err := debug.Setup(ctx); err != nil {
Return err
}
// Start system runtime metrics collection
Go metrics.CollectProcessMetrics(3 * time.Second)

utils.SetupNetwork(ctx)
Return nil
}

app.After = func(ctx *cli.Context) error {
debug.Exit()
console.Stdin.Close() // Resets terminal mode.
Return nil
}
}

If we don't enter any parameters, the geth method is called automatically.

// geth is the main entry point into the system if no special subcommand is ran.
// It creates a default node based on the command line arguments and runs it in
// blocking mode, waiting for it to be shut down.
// If no special subcommand is specified, then the geth is the main entry for the system.
// It creates a default node based on the supplied parameters. And run this node in blocking mode, waiting for the node to be terminated.
Func geth(ctx *cli.Context) error {
Node := makeFullNode(ctx)
startNode(ctx, node)
node.Wait()
Return nil
}

makeFullNode function,

Func makeFullNode(ctx *cli.Context) *node.Node {
// Create a node based on command line arguments and some special configuration
Stack, cfg := makeConfigNode(ctx)
// Register the eth service on this node. Eth services are the main services of Ethereum. The provider of the Ethereum function.
utils.RegisterEthService(stack, &amp;cfg.Eth)

// Whisper must be explicitly enabled by specifying at least 1 whisper flag or in dev mode
// Whisper is a new module for encrypting communications. You need to explicitly provide parameters to enable, or be in development mode.
shhEnabled := enableWhisper(ctx)
shhAutoEnabled := !ctx.GlobalIsSet(utils.WhisperEnabledFlag.Name) &amp;&amp; ctx.GlobalIsSet(utils.DevModeFlag.Name)
If shhEnabled || shhAutoEnabled {
If ctx.GlobalIsSet(utils.WhisperMaxMessageSizeFlag.Name) {
cfg.Shh.MaxMessageSize = uint32(ctx.Int(utils.WhisperMaxMessageSizeFlag.Name))
}
If ctx.GlobalIsSet(utils.WhisperMinPOWFlag.Name) {
cfg.Shh.MinimumAcceptedPOW = ctx.Float64(utils.WhisperMinPOWFlag.Name)
}
// Register Shh service
utils.RegisterShhService(stack, &amp;cfg.Shh)
}

// Add the Ethereum Stats daemon if requested.
If cfg.Ethstats.URL != "" {
// Register Ethereum's status service. By default it is not started.
utils.RegisterEthStatsService(stack, cfg.Ethstats.URL)
}

// Add the release oracle service so it boots along with node.
// The release oracle service is used to check if the client version is the latest version of the service.
// If you need to update. Then the version will be prompted by printing the log.
// release is run as a smart contract. This service will be discussed in detail later.
If err := stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
Config := release.Config{
Oracle: relOracle,
Major: uint32(params.VersionMajor),
Minor: uint32(params.VersionMinor),
Patch: uint32(params.VersionPatch),
}
Commit, _ := hex.DecodeString(gitCommit)
Copy(config.Commit[:], commit)
Return release.NewReleaseService(ctx, config)
}); err != nil {
utils.Fatalf("Failed to register the Geth release oracle service: %v", err)
}
Return stack
}

makeConfigNode. This function mainly generates the running configuration of the entire system through the configuration file and flag.

Func makeConfigNode(ctx *cli.Context) (*node.Node, gethConfig) {
// Load defaults.
Cfg := gethConfig{
Eth: eth.DefaultConfig,
Shh: whisper.DefaultConfig,
Node: defaultNodeConfig(),
}

// Load config file.
If file := ctx.GlobalString(configFileFlag.Name); file != "" {
If err := loadConfig(file, &amp;cfg); err != nil {
utils.Fatalf("%v", err)
}
}

// Apply flags.
utils.SetNodeConfig(ctx, &amp;cfg.Node)
Stack, err := node.New(&amp;cfg.Node)
If err != nil {
utils.Fatalf("Failed to create the protocol stack: %v", err)
}
utils.SetEthConfig(ctx, stack, &amp;cfg.Eth)
If ctx.GlobalIsSet(utils.EthStatsURLFlag.Name) {
cfg.Ethstats.URL = ctx.GlobalString(utils.EthStatsURLFlag.Name)
}

utils.SetShhConfig(ctx, stack, &amp;cfg.Shh)

Return stack, cfg
}

RegisterEthService

// RegisterEthService adds an Ethereum client to the stack.
Func RegisterEthService(stack *node.Node, cfg *eth.Config) {
Var err error
// If the sync mode is a lightweight sync mode. Then start a lightweight client.
If cfg.SyncMode == downloader.LightSync {
Err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
Return les.New(ctx, cfg)
})
} else {
// Otherwise it will start the full node
Err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
fullNode, err := eth.New(ctx, cfg)
If fullNode != nil &amp;&amp; cfg.LightServ &gt; 0 {
// The default LightServ size is 0, which means that LesServer will not be started.
// LesServer is a service for lightweight nodes.
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


startNode

// startNode boots up the system node and all registered protocols, after which
// it unlocks any requested accounts, and starts the RPC/IPC interfaces and the
// miner.
Func startNode(ctx *cli.Context, stack *node.Node) {
// Start up the node itself
utils.StartNode(stack)

// Unlock any account specifically requested
Ks := stack.AccountManager().Backends(keystore.KeyStoreType)[0].(*keystore.KeyStore)

Passwords := utils.MakePasswordList(ctx)
Unlocks := strings.Split(ctx.GlobalString(utils.UnlockedAccountFlag.Name), ",")
For i, account := range unlocks {
If trimmed := strings.TrimSpace(account); trimmed != "" {
unlockAccount(ctx, ks, trimmed, i, passwords)
}
}
// Register wallet event handlers to open and auto-derive wallets
Events := make(chan accounts.WalletEvent, 16)
stack.AccountManager().Subscribe(events)

Go func() {
// Create an chain state reader for self-derivation
rpcClient, err := stack.Attach()
If err != nil {
utils.Fatalf("Failed to attach to self: %v", err)
}
stateReader := ethclient.NewClient(rpcClient)

// Open any wallets already attached
For _, wallet := range stack.AccountManager().Wallets() {
If err := wallet.Open(""); err != nil {
log.Warn("Failed to open wallet", "url", wallet.URL(), "err", err)
}
}
// Listen for wallet event till termination
For event := range events {
Switch event.Kind {
Case accounts.WalletArrived:
If err := event.Wallet.Open(""); err != nil {
log.Warn("New wallet appeared, failed to open", "url", event.Wallet.URL(), "err", err)
}
Case accounts.WalletOpened:
Status, _ := event.Wallet.Status()
log.Info("New wallet appeared", "url", event.Wallet.URL(), "status", status)

If event.Wallet.URL().Scheme == "ledger" {
event.Wallet.SelfDerive(accounts.DefaultLedgerBaseDerivationPath, stateReader)
} else {
event.Wallet.SelfDerive(accounts.DefaultBaseDerivationPath, stateReader)
}

Case accounts.WalletDropped:
log.Info("Old wallet dropped", "url", event.Wallet.URL())
event.Wallet.Close()
}
}
}()
// Start auxiliary services if enabled
If ctx.GlobalBool(utils.MiningEnabledFlag.Name) {
// Mining only makes sense if a full Ethereum node is running
Var ethereum *eth.Ethereum
If err := stack.Service(&amp;ethereum); err != nil {
utils.Fatalf("ethereum service not running: %v", err)
}
// Use a reduced number of threads if requested
If threads := ctx.GlobalInt(utils.MinerThreadsFlag.Name); threads &gt; 0 {
Type threaded interface {
SetThreads(threads int)
}
If th, ok := ethereum.Engine().(threaded); ok {
th.SetThreads(threads)
}
}
// Set the gas price to the limits from the CLI and start mining
ethereum.TxPool().SetGasPrice(utils.GlobalBig(ctx, utils.GasPriceFlag.Name))
If err := ethereum.StartMining(true); err != nil {
utils.Fatalf("Failed to start mining: %v", err)
}
}
}

to sum up:

The entire startup process is actually parsing parameters. Then create and start the node. Then inject the service into the node. All functions related to Ethereum are implemented in the form of services.


If you remove all registered services. What are the goroutines that the system opens at this time. Make a summary here.


Currently all resident goroutines have the following. Mainly p2p related services. And RPC related services.

![image](picture/geth_1.png)

</pre></body></html>