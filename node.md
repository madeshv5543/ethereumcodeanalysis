
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>The node represents a node in go ethereum. May be a full node, possibly a lightweight node. Node can be understood as a process, Ethereum consists of many types of nodes running around the world.

A typical node is a p2p node. The p2p network protocol is run, and different service layer protocols are run according to different node types (to distinguish the network layer protocol. Refer to the protocol interface in the p2p peer).

The structure of the node.

// Node is a container on which services can be registered.
Type Node struct {
Eventmux *event.TypeMux // Event multiplexer used between the services of a stack
Config *Config
Accman *accounts.Manager

ephemeralKeystore string // if non-empty, the key directory that will be removed by Stop
instanceDirLock flock.Releaser //detect concurrent use of instance directory

serverConfig p2p.Config
Server *p2p.Server // Currently running P2P networking layer

serviceFuncs []ServiceConstructor // Service constructors (in dependency order)
Services map[reflect.Type]Service // Currently running services

rpcAPIs []rpc.API // List of APIs currently provided by the node
inprocHandler *rpc.Server // In-process RPC request handler to process the API requests

ipcEndpoint string // IPC endpoint to listen at (empty = IPC disabled)
ipcListener net.Listener // IPC RPC listener socket to serve API requests
ipcHandler *rpc.Server // IPC RPC request handler to process the API requests

httpEndpoint string // HTTP endpoint (interface + port) to listen at (empty = HTTP disabled)
httpWhitelist []string // HTTP RPC modules to allow through this endpoint
httpListener net.Listener // HTTP RPC listener socket to server API requests
httpHandler *rpc.Server // HTTP RPC request handler to process the API requests

wsEndpoint string // Websocket endpoint (interface + port) to listen at (empty = websocket disabled)
wsListener net.Listener // Websocket RPC listener socket to server API requests
wsHandler *rpc.Server // Websocket RPC request handler to process the API requests

Stop chan struct{} // Channel to wait for termination notifications
Lock sync.RWMutex
}


The initialization of the node, the initialization of the node does not depend on other external components, only rely on a Config object.

// New creates a new P2P node, ready for protocol registration.
Func New(conf *Config) (*Node, error) {
// Copy config and resolve the datadir so future changes to the current
// working directory don't affect the node.
confCopy := *conf
Conf = &amp;confCopy
If conf.DataDir != "" { //Convert to an absolute path.
Absdatadir, err := filepath.Abs(conf.DataDir)
If err != nil {
Return nil, err
}
conf.DataDir = absdatadir
}
// Ensure that the instance name doesn't cause weird conflicts with
// other files in the data directory.
If strings.ContainsAny(conf.Name, `/\`) {
Return nil, errors.New(`Config.Name must not contain '/' or '\'`)
}
If conf.Name == datadirDefaultKeyStore {
Return nil, errors.New(`Config.Name cannot be "` + datadirDefaultKeyStore + `"`)
}
If strings.HasSuffix(conf.Name, ".ipc") {
Return nil, errors.New(`Config.Name cannot end in ".ipc"`)
}
// Ensure that the AccountManager method works before the node has started.
// We rely on this in cmd/geth.
Am, ephemeralKeystore, err := makeAccountManager(conf)
If err != nil {
Return nil, err
}
// Note: any interaction with Config that would create/touch files
// in the data directory or instance directory is cancelled until Start.
Return &amp;Node{
Accman: am,
ephemeralKeystore: ephemeralKeystore,
Config: conf,
serviceFuncs: []ServiceConstructor{},
ipcEndpoint: conf.IPCEndpoint(),
httpEndpoint: conf.HTTPEndpoint(),
wsEndpoint: conf.WSEndpoint(),
Eventmux: new(event.TypeMux),
}, nil
}


### node Registration of services and agreements
Because node is not responsible for the specific business logic. So the specific business logic is registered to the node through registration.
Other modules register a service constructor via the Register method. Use this service constructor to generate a service.


// Register injects a new service into the node's stack. The service created by
// the passed constructor must be unique in its type with regard to sibling ones.
Func (n *Node) Register(constructor ServiceConstructor) error {
n.lock.Lock()
Defer n.lock.Unlock()

If n.server != nil {
Return ErrNodeRunning
}
n.serviceFuncs = append(n.serviceFuncs, constructor)
Return nil
}

What is the service?

Type ServiceConstructor func(ctx *ServiceContext) (Service, error)
// Service is an individual protocol that can be registered into a node.
//
// Notes:
//
// • Service life-cycle management is delegated to the node. The service is allowed to
// initialize itself upon creation, but no goroutines should be spun up outside of the
// Start method.
//
// • Restart logic is not required as the node will create a fresh instance
// every time a service is started.

// Service lifecycle management has been delegated to node management. This service allows automatic initialization when it is created, but goroutines should not be started outside of the Start method.
// Restart logic is not required because the node will create a new instance each time the service is started.
Type Service interface {
// Protocols retrieves the P2P protocols the service wishes to start.
// The p2p protocol that the service wishes to provide
Protocols() []p2p.Protocol

// APIs retrieves the list of RPC descriptors the service provides
// Description of the RPC method the service wishes to provide
APIs() []rpc.API

// Start is called after all services have been constructed and the networking
// layer was also initialized to spawn any goroutines required by the service.
// After all services have been built, the call begins and the network layer is initialized to produce any gorouts needed for the service.
Start(server *p2p.Server) error

// Stop terminates all goroutines belonging to the service, blocking until they
// are all terminated.

// The Stop method will stop all gorouts owned by this service. Need to block until all goroutines have been terminated
Stop() error
}


### node startup
The node startup process creates and runs a p2p node.

// Start create a live P2P node and starts running it.
Func (n *Node) Start() error {
n.lock.Lock()
Defer n.lock.Unlock()

// Short circuit if the node's already running
If n.server != nil {
Return ErrNodeRunning
}
If err := n.openDataDir(); err != nil {
Return err
}

// Initialize the p2p server. This creates the node key and
// discovery databases.
n.serverConfig = n.config.P2P
n.serverConfig.PrivateKey = n.config.NodeKey()
n.serverConfig.Name = n.config.NodeName()
If n.serverConfig.StaticNodes == nil {
/ / Processing configuration file static-nodes.json
n.serverConfig.StaticNodes = n.config.StaticNodes()
}
If n.serverConfig.TrustedNodes == nil {
// Process the configuration file trusted-nodes.json
n.serverConfig.TrustedNodes = n.config.TrustedNodes()
}
If n.serverConfig.NodeDatabase == "" {
n.serverConfig.NodeDatabase = n.config.NodeDB()
}
/ / Created a p2p server
Running := &amp;p2p.Server{Config: n.serverConfig}
log.Info("Starting peer-to-peer node", "instance", n.serverConfig.Name)

// Otherwise copy and specialize the P2P configuration
Services := make(map[reflect.Type]Service)
For _, constructor := range n.serviceFuncs {
// Create a new context for the particular service
Ctx := &amp;ServiceContext{
Config: n.config,
Services: make(map[reflect.Type]Service),
EventMux: n.eventmux,
AccountManager: n.accman,
}
For kind, s := range services { // copy needed for threaded access
Ctx.services[kind] = s
}
// Construct and save the service
// Create all registered services.
Service, err := constructor(ctx)
If err != nil {
Return err
}
Kind := reflect.TypeOf(service)
If _, exists := services[kind]; exists {
Return &amp;DuplicateServiceError{Kind: kind}
}
Services[kind] = service
}
// Gather the protocols and start the freshly assembled P2P server
// Collect all p2p protocols and insert p2p.Rrotocols
For _, service := range services {
running.Protocols = append(running.Protocols, service.Protocols()...)
}
// started the p2p server
If err := running.Start(); err != nil {
Return convertFileLockError(err)
}
// Start each of the services
// start every service
Started := []reflect.Type{}
For kind, service := range services {
// Start the next service, stopping all previous upon failure
If err := service.Start(running); err != nil {
For _, kind := range started {
Services[kind].Stop()
}
running.Stop()

Return err
}
// Mark the service started for potential cleanup
Started = append(started, kind)
}
// Lastly start the configured RPC interfaces
// Finally start the RPC service
If err := n.startRPC(services); err != nil {
For _, service := range services {
service.Stop()
}
running.Stop()
Return err
}
// Finish initializing the startup
N.services = services
N.server = running
N.stop = make(chan struct{})

Return nil
}


startRPC, this method collects all apis. And in turn call to start each RPC server, the default is to start InProc and IPC. If specified, you can also configure whether to start HTTP and websocket.

// startRPC is a helper method to start all the various RPC endpoint during node
// startup. It's not meant to be called at any time afterwards as it makes certain
// assumptions about the state of the node.
Func (n *Node) startRPC(services map[reflect.Type]Service) error {
// Gather all the possible APIs to surface
Apis := n.apis()
For _, service := range services {
Apis = append(apis, service.APIs()...)
}
// Start the various API endpoints, terminating all in case of errors
If err := n.startInProc(apis); err != nil {
Return err
}
If err := n.startIPC(apis); err != nil {
n.stopInProc()
Return err
}
If err := n.startHTTP(n.httpEndpoint, apis, n.config.HTTPModules, n.config.HTTPCors); err != nil {
n.stopIPC()
n.stopInProc()
Return err
}
If err := n.startWS(n.wsEndpoint, apis, n.config.WSModules, n.config.WSOrigins, n.config.WSExposeAll); err != nil {
n.stopHTTP()
n.stopIPC()
n.stopInProc()
Return err
}
// All API endpoints started successfully
n.rpcAPIs = apis
Return nil
}


StartXXX is the start of a specific RPC, and the process is similar. In the v1.8.12 version, the specific startup methods of startIPC(), startHTTP(), and startWS() in the node\node.go file are encapsulated into the corresponding function of the rpc\endpoints.go file.

// StartWSEndpoint starts a websocket endpoint
Func StartWSEndpoint(endpoint string, apis []API, modules []string, wsOrigins []string, exposeAll bool) (net.Listener, *Server, error) {

// Generate the whitelist based on the allowed modules
// Generate a whitelist
Whitelist := make(map[string]bool)
For _, module := range modules {
Whitelist[module] = true
}
// Register all the APIs exposed by the services
Handler := NewServer()
For _, api := range apis {
If exposeAll || whitelist[api.Namespace] || (len(whitelist) == 0 &amp;&amp; api.Public) { // This api will only be registered in these cases.
If err := handler.RegisterName(api.Namespace, api.Service); err != nil {
Return nil, nil, err
}
log.Debug("WebSocket registered", "service", api.Service, "namespace", api.Namespace)
}
}
// All APIs registered, start the HTTP listener
// All APIs are already registered, starting the HTTP listener
Var (
Listener net.Listener
Err error
)
If listener, err = net.Listen("tcp", endpoint); err != nil {
Return nil, nil, err
}
Go NewWSServer(wsOrigins, handler).Serve(listener)
Return listener, handler, err

}

</pre></body></html>