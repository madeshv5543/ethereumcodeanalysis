
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>## RPC package official documentation

Package rpc provides access to the exported methods of an object across a network
Or other I/O connection. After creating a server instance objects can be registered,
Making it visible from the outside. Exported methods that follow specific
Conventions can be called remotely. It also has support for the publish/subscribe
Pattern.

The rpc package provides the ability to access methods by which objects are exported over a network or other I/O connection. After creating a server, objects can be registered to the server and then made accessible to the outside world. Methods exported via fat can be called remotely. The publish/subscribe mode is also supported.

Methods that satisfy the following criteria are made available for remote access:

- object must be exported
- method must be exported
- method returns 0, 1 (response or error) or 2 (response and error) values
- method argument(s) must be exported or builtin types
- method returned value(s) must be exported or builtin types

Methods that meet the following criteria are available for remote access:

- Object must be exported
- Method must be exported
- Method returns 0, 1 (response or error) or 2 (response and error) values
- Method parameters must be exported or built-in types
- method return value must be exported or built-in type

An example method:

Func (s *CalcService) Add(a, b int) (int, error)

When the returned error isn't nil the returned integer is ignored and the error is
Send back to the client. Otherwise the returned integer is send back to the client.

When the returned error is not equal to nil, the returned integer value is ignored and the error is sent back to the client. Otherwise the integer will be sent back to the client.

Optional arguments are supported by accepting pointer values ​​as arguments. E.g.
If we want to do the addition in an optional finite field we can accept a mod
Argument as pointer value.
The method supports optional parameters by providing a pointer type parameter. I can't understand it later.

Func (s *CalService) Add(a, b int, mod *int) (int, error)

This RPC method can be called with 2 integers and a null value as third argument.
In that case the mod argument will be nil. Or it can be called with 3 integers,
In that case mod will be pointing to the given third argument.
Argument is the last argument the RPC package will also accept 2 integers as
Arguments. It will pass the mod argument as nil to the RPC method.

The RPC method can be called by passing two integers and a null value as the third argument. In this case the mod parameter will be set to nil. Or you can pass three integers so that the mod will be set to point to the third argument. Although the optional argument is the last argument, the RPC packet will still receive and pass two integers, so the mod argument will be set to nil.

The server offers the ServeCodec method which accepts a ServerCodec instance. It will
Read requests from the codec, process the request and sends the response back to the
Client using the codec. The server can execute requests concurrently.
Can be sent back to the client out of order.

The server provides the ServerCodec method, which receives the ServerCodec instance as a parameter. The server will use codec to read the request, process the request, and then send a response to the client via codec. The server can execute requests concurrently. The order of the responses may not match the order of the requests.

//An example server which uses the JSON codec:
Type CalculatorService struct {}

Func (s *CalculatorService) Add(a, b int) int {
Return a + b
}

Func (s *CalculatorService Div(a, b int) (int, error) {
If b == 0 {
Return 0, errors.New("divide by zero")
}
Return a/b, nil
}
Calculator := new(CalculatorService)
Server := NewServer()
server.RegisterName("calculator", calculator")

l, _ := net.ListenUnix("unix", &amp;net.UnixAddr{Net: "unix", Name: "/tmp/calculator.sock"})
For {
c, _ := l.AcceptUnix()
Codec := v2.NewJSONCodec(c)
Go server.ServeCodec(codec)
}


The package also supports the publish subscribe pattern through the use of subscriptions.
A method that is considered eligible for notifications must satisfy the following criteria:

&nbsp;- object must be exported
&nbsp;- method must be exported
&nbsp;- first method argument type must be context.Context
&nbsp;- method argument(s) must be exported or builtin types
&nbsp;- method must return the tuple Subscription, error


The package also supports the publish subscription mode by using subscriptions.
The method considered to meet the notification conditions must meet the following conditions:

- Object must be exported
- Method must be exported
- The first method parameter type must be context.Context
- Method parameters must be exported or built-in types
- method must return tuple subscription, error

An example method:

Func (s *BlockChainService) NewBlocks(ctx context.Context) (Subscription, error) {
...
}

Subscriptions are deleted when:

&nbsp;- the user sends an unsubscribe request
&nbsp;- the connection which was used to create the subscription is closed. This can be initiated
&nbsp;&nbsp;&nbsp;By the client and server. The server will close the connection on an write error or when
&nbsp;&nbsp;&nbsp;The queue of buffered notifications gets too big.

Subscriptions will be deleted in the following cases

- The user sent a request to cancel the subscription
- The connection to create the subscription is closed. This situation can be triggered by the client or server. The server chooses to close the connection when there is a write error or if the notification queue is too long.

## RPC package's general structure
Network Protocols The encoding and decoding of requests and responses in the channels and Json formats are classes that deal with both the server and the client. Network protocol channels mainly provide the function of connection and data transmission. The encoding and decoding of the json format mainly provides serialization and deserialization of requests and responses (Json -&gt; Go objects).

![image](picture/rpc_1.png)


## Source Resolution

### server.go
Server.go mainly implements the core logic of the RPC server. It includes logic for registering RPC methods, reading requests, processing requests, and sending responses.
The core data structure of the server is the Server structure. The services field is a map that records all registered methods and classes. The run parameter is used to control the running and stopping of the server. Codecs is a set. Used to store all codecs, in fact, all connections. codecsMu is a lock used to protect multithreaded access to codecs.

The value type of the services field is the service type. Service represents an instance registered to Server, a combination of objects and methods. The name of the service field represents the namespace of the service, the type of the typ instance, callbacks is the callback method of the instance, and subscriptions is the subscription method of the instance.


Type serviceRegistry map[string]*service // collection of services
Type callbacks map[string]*callback // collection of RPC callbacks
Type subscriptions map[string]*callback
Type Server struct {
Services serviceRegistry

Run int32
codecsMu sync.Mutex
Codecs *set.Set
}

// callback is a method callback which was registered in the server
Type callback struct {
Rcvr reflect.Value // receiver of method
Method reflect.Method // callback
argTypes []reflect.Type // input argument types
hasCtx bool // method's first argument is a context (not included in argTypes)
errPos int // err return idx, of -1 when method cannot return error
isSubscribe bool // indication if the callback is a subscription
}

// service represents a registered object
Type service struct {
Name string // name for service
Typ reflect.Type // receiver type
Callbacks callbacks // registered handlers
Subscriptions subscriptions // available subscriptions/notifications
}


Server creation, when the server is created, register its own instance by calling server.RegisterName, and provide some meta information of the RPC service.

Const MetadataApi = "rpc"
// NewServer will create a new server instance with no registered handlers.
Func NewServer() *Server {
Server := &amp;Server{
Services: make(serviceRegistry),
Codecs: set.New(),
Run: 1,
}

// register a default service which will provide meta information about the RPC service such as the services and
// methods it offers.
rpcService := &amp;RPCService{server}
server.RegisterName(MetadataApi, rpcService)

Return server
}

The service registers server.RegisterName, the RegisterName method will create a service object through the passed parameters. If the passed rcvr instance does not find any suitable method, it will return an error. If there are no errors, add the created service instance to the serviceRegistry.


// RegisterName will create a service for the given rcvr type under the given name. When no methods on the given rcvr
// match the criteria to be either a RPC method or a subscription an error is returned. Otherwise a new service is
// created and added to the service collection this server instance serves.
Func (s *Server) RegisterName(name string, rcvr interface{}) error {
If s.services == nil {
S.services = make(serviceRegistry)
}

Svc := new(service)
Svc.typ = reflect.TypeOf(rcvr)
rcvrVal := reflect.ValueOf(rcvr)

If name == "" {
Return fmt.Errorf("no service name for type %s", svc.typ.String())
}
// If the instance's class name is not exported (the first letter of the class name is capitalized), an error is returned.
If !isExported(reflect.Indirect(rcvrVal).Type().Name()) {
Return fmt.Errorf("%s is not exported", reflect.Indirect(rcvrVal).Type().Name())
}
/ / Find the appropriate callbacks and subscriptions method by reflecting information
Methods, subscriptions := suitableCallbacks(rcvrVal, svc.typ)
//If the name is currently registered, then if there is a method with the same name, use the new one, or insert it directly.
// already a previous service register under given sname, merge methods/subscriptions
If regsvc, present := s.services[name]; present {
If len(methods) == 0 &amp;&amp; len(subscriptions) == 0 {
Return fmt.Errorf("Service %T doesn't have any suitable methods/subscriptions to expose", rcvr)
}
For _, m := range methods {
Regsvc.callbacks[formatName(m.method.Name)] = m
}
For _, s := range subscriptions {
Regsvc.subscriptions[formatName(s.method.Name)] = s
}
Return nil
}

Svc.name = name
Svc.callbacks, svc.subscriptions = methods, subscriptions

If len(svc.callbacks) == 0 &amp;&amp; len(svc.subscriptions) == 0 {
Return fmt.Errorf("Service %T doesn't have any suitable methods/subscriptions to expose", rcvr)
}

S.services[svc.name] = svc
Return nil
}

Find the appropriate method by reflecting the information, suitableCallbacks, which is in utils.go. This method will iterate over all methods of this type, find methods that match the RPC callback or subscription callback type criteria and return. For the RPC standard, please refer to the RPC standard at the beginning of the document.

// suitableCallbacks iterates over the methods of the given type. It will determine if a method satisfies the criteria
// for a RPC callback or a subscription callback and adds it to the collection of callbacks or subscriptions. See server
// Documentation for a summary of these criteria.
Func suitableCallbacks(rcvr reflect.Value, typ reflect.Type) (callbacks, subscriptions) {
Callbacks := make(callbacks)
Subscriptions := make(subscriptions)

METHODS:
For m := 0; m &lt; typ.NumMethod(); m++ {
Method := typ.Method(m)
Mtype := method.Type
Mname := formatName(method.Name)
If method.PkgPath != "" { // method must be exported
Continue
}

Var h callback
h.isSubscribe = isPubSub(mtype)
H.rcvr = rcvr
H.method = method
h.errPos = -1

firstArg := 1
numIn := mtype.NumIn()
If numIn &gt;= 2 &amp;&amp; mtype.In(1) == contextType {
h.hasCtx = true
firstArg = 2
}

If h.isSubscribe {
h.argTypes = make([]reflect.Type, numIn-firstArg) // skip rcvr type
For i := firstArg; i &lt; numIn; i++ {
argType := mtype.In(i)
If isExportedOrBuiltinType(argType) {
h.argTypes[i-firstArg] = argType
} else {
Continue METHODS
}
}

Subscriptions[mname] = &amp;h
Continue METHODS
}

// determine method arguments, ignore first arg since it's the receiver type
// Arguments must be exported or builtin types
h.argTypes = make([]reflect.Type, numIn-firstArg)
For i := firstArg; i &lt; numIn; i++ {
argType := mtype.In(i)
If !isExportedOrBuiltinType(argType) {
Continue METHODS
}
h.argTypes[i-firstArg] = argType
}

// check that all returned values ​​are exported or builtin types
For i := 0; i &lt; mtype.NumOut(); i++ {
If !isExportedOrBuiltinType(mtype.Out(i)) {
Continue METHODS
}
}

// when a method returns an error it must be the last returned value
h.errPos = -1
For i := 0; i &lt; mtype.NumOut(); i++ {
If isErrorType(mtype.Out(i)) {
h.errPos = i
Break
}
}

If h.errPos &gt;= 0 &amp;&amp; h.errPos != mtype.NumOut()-1 {
Continue METHODS
}

Switch mtype.NumOut() {
Case 0, 1, 2:
If mtype.NumOut() == 2 &amp;&amp; h.errPos == -1 { // method must one return value and 1 error
Continue METHODS
}
Callbacks[mname] = &amp;h
}
}

Return callbacks, subscriptions
}


Server startup and services, server startup and services here refer to some of the code in ipc.go. You can see a link for each Accept(), and start a goroutine call srv.ServeCodec for service. Here you can also see the function of JsonCodec. Codec is similar to the decorator mode, and it has a layer of bread on the connection. Codec will be introduced later, here is a brief look.

Func (srv *Server) ServeListener(l net.Listener) error {
For {
Conn, err := l.Accept()
If err != nil {
Return err
}
log.Trace(fmt.Sprint("accepted conn", conn.RemoteAddr()))
Go srv.ServeCodec(NewJSONCodec(conn), OptionMethodInvocation|OptionSubscriptions)
}
}

ServeCodec, this method is very simple, provides the shutdown function of codec.Close. The second parameter of serveRequest, singleShot, is a parameter that controls long or short connections. If singleShot is true, it will exit after processing a request. However, our serveRequest method is an infinite loop, no exception is encountered, or the client actively closes, the server will not close. So rpc provides the function of long connection.

// ServeCodec reads incoming requests from codec, calls the appropriate callback and writes the
// response back using the given codec. It will block until the codec is closed or the server is
// stopped. In either case the codec is closed.
Func (s *Server) ServeCodec(codec ServerCodec, options CodecOption) {
Defer codec.Close()
s.serveRequest(codec, false, options)
}

Our heavy method finally came out, the serveRequest method is the main processing flow of Server. Read the request from codec, find the corresponding method and call it, then write the response to codec.

Some standard library code can refer to the online tutorial, sync.WaitGroup implements a semaphore function. Context implements context management.


// serveRequest will reads requests from the codec, calls the RPC callback and
// writes the response to the given codec.
//
// If singleShot is true it will process a single request, otherwise it will handle
// requests until the codec returns an error when reading a request (in most cases
// an EOF). It executes requests in parallel when singleShot is false.
Func (s *Server) serveRequest(codec ServerCodec, singleShot bool, options CodecOption) error {
Var pend sync.WaitGroup
Defer func() {
If err := recover(); err != nil {
Const size = 64 &lt;&lt; 10
Buf := make([]byte, size)
Buf = buf[:runtime.Stack(buf, false)]
log.Error(string(buf))
}
s.codecsMu.Lock()
s.codecs.Remove(codec)
s.codecsMu.Unlock()
}()

Ctx, cancel := context.WithCancel(context.Background())
Defer cancel()

// if the codec supports notification include a notifier that callbacks can use
// to send notification to clients. It is thight to the codec/connection. If the
// connection is closed the notifier will stop and cancels all active subscriptions.
If options&amp;OptionSubscriptions == OptionSubscriptions {
Ctx = context.WithValue(ctx, notifierKey{}, newNotifier(codec))
}
s.codecsMu.Lock()
If atomic.LoadInt32(&amp;s.run) != 1 { // server stopped
s.codecsMu.Unlock()
Return &amp;shutdownError{}
}
s.codecs.Add(codec)
s.codecsMu.Unlock()

// test if the server is ordered to stop
For atomic.LoadInt32(&amp;s.run) == 1 {
Reqs, batch, err := s.readRequest(codec)
If err != nil {
// If a parsing error occurred, send an error
If err.Error() != "EOF" {
log.Debug(fmt.Sprintf("read error %v\n", err))
codec.Write(codec.CreateErrorResponse(nil, err))
}
// Error or end of stream, wait for requests and tear down
/ / Here is mainly to consider the multi-thread processing when waiting for all the request processing is completed,
//everever a go thread is started will call pend.Add(1).
// Calling pend.Done() after processing is complete will subtract 1. When it is 0, the Wait() method will return.
pend.Wait()
Return nil
}

// check if server is ordered to shutdown and return an error
// telling the client that his request failed.
If atomic.LoadInt32(&amp;s.run) != 1 {
Err = &amp;shutdownError{}
If batch {
Resps := make([]interface{}, len(reqs))
For i, r := range reqs {
Resps[i] = codec.CreateErrorResponse(&amp;r.id, err)
}
codec.Write(resps)
} else {
codec.Write(codec.CreateErrorResponse(&amp;reqs[0].id, err))
}
Return nil
}
// If a single shot request is executing, run and return immediately
/ / If only once, then return after execution.
If singleShot {
If batch {
s.execBatch(ctx, codec, reqs)
} else {
S.exec(ctx, codec, reqs[0])
}
Return nil
}
// For multi-shot connections, start a goroutine to serve and loop back
pend.Add(1)
/ / Start the thread to service the request.
Go func(reqs []*serverRequest, batch bool) {
Defer pend.Done()
If batch {
s.execBatch(ctx, codec, reqs)
} else {
S.exec(ctx, codec, reqs[0])
}
}(reqs, batch)
}
Return nil
}


The readRequest method reads the request from codec and then assembles the corresponding method into the requests object according to the request.
rpcRequest is the type of request returned by codec. j

Type rpcRequest struct {
Service string
Method string
Id interface{}
isPubSub bool
Params interface{}
Err Error // invalid batch element
}

Request returned after serverRequest is processed

// serverRequest is an incoming request
Type serverRequest struct {
Id interface{}
Svcname string
Callb *callback
Args []reflect.Value
isUnsubscribe bool
Err Error
}

The readRequest method reads the request from codec and processes the request to generate a serverRequest object.

// readRequest requests the next (batch) request from the codec. It will return the collection
// of requests, an indication if the request was a batch, the invalid request identifier and an
// error when the request could not be read/parsed.
Func (s *Server) readRequest(codec ServerCodec) ([]*serverRequest, bool, Error) {
Reqs, batch, err := codec.ReadRequestHeaders()
If err != nil {
Return nil, batch, err
}
Requests := make([]*serverRequest, len(reqs))
// build requests based on reqs
// verify requests
For i, r := range reqs {
Var ok bool
Var svc *service

If r.err != nil {
Requests[i] = &amp;serverRequest{id: r.id, err: r.err}
Continue
}
// If the request is a send/subscribe request, and the method name has a _unsubscribe suffix.
If r.isPubSub &amp;&amp; strings.HasSuffix(r.method, unsubscribeMethodSuffix) {
Requests[i] = &amp;serverRequest{id: r.id, isUnsubscribe: true}
argTypes := []reflect.Type{reflect.TypeOf("")} // expect subscription id as first arg
If args, err := codec.ParseRequestArguments(argTypes, r.params); err == nil {
Requests[i].args = args
} else {
Requests[i].err = &amp;invalidParamsError{err.Error()}
}
Continue
}
//If you have not registered this service.
If svc, ok = s.services[r.service]; !ok { // rpc method isn't available
Requests[i] = &amp;serverRequest{id: r.id, err: &amp;methodNotFoundError{r.service, r.method}}
Continue
}
// If it is a publish and subscribe mode. Call the subscription method.
If r.isPubSub { // eth_subscribe, r.method contains the subscription method name
If callb, ok := svc.subscriptions[r.method]; ok {
Requests[i] = &amp;serverRequest{id: r.id, svcname: svc.name, callb: callb}
If r.params != nil &amp;&amp; len(callb.argTypes) &gt; 0 {
argTypes := []reflect.Type{reflect.TypeOf("")}
argTypes = append(argTypes, callb.argTypes...)
If args, err := codec.ParseRequestArguments(argTypes, r.params); err == nil {
Requests[i].args = args[1:] // first one is service.method name which isn't an actual argument
} else {
Requests[i].err = &amp;invalidParamsError{err.Error()}
}
}
} else {
Requests[i] = &amp;serverRequest{id: r.id, err: &amp;methodNotFoundError{r.method, r.method}}
}
Continue
}

If callb, ok := svc.callbacks[r.method]; ok { // lookup RPC method
Requests[i] = &amp;serverRequest{id: r.id, svcname: svc.name, callb: callb}
If r.params != nil &amp;&amp; len(callb.argTypes) &gt; 0 {
If args, err := codec.ParseRequestArguments(callb.argTypes, r.params); err == nil {
Requests[i].args = args
} else {
Requests[i].err = &amp;invalidParamsError{err.Error()}
}
}
Continue
}

Requests[i] = &amp;serverRequest{id: r.id, err: &amp;methodNotFoundError{r.service, r.method}}
}

Return requests, batch, nil
}

The exec and execBatch methods call the s.handle method to process the request.

// exec executes the given request and writes the result back using the codec.
Func (s *Server) exec(ctx context.Context, codec ServerCodec, req *serverRequest) {
Var response interface{}
Var callback func()
If req.err != nil {
Response = codec.CreateErrorResponse(&amp;req.id, req.err)
} else {
Response, callback = s.handle(ctx, codec, req)
}

If err := codec.Write(response); err != nil {
log.Error(fmt.Sprintf("%v\n", err))
codec.Close()
}

// when request was a subscribe request this allows these subscriptions to be actived
If callback != nil {
Callback()
}
}

// execBatch executes the given requests and writes the result back using the codec.
// It will only write the response back when the last request is processed.
Func (s *Server) execBatch(ctx context.Context, codec ServerCodec, requests []*serverRequest) {
Responses := make([]interface{}, len(requests))
Var callbacks []func()
For i, req := range requests {
If req.err != nil {
Responses[i] = codec.CreateErrorResponse(&amp;req.id, req.err)
} else {
Var callback func()
If responses[i], callback = s.handle(ctx, codec, req); callback != nil {
Callbacks = append(callbacks, callback)
}
}
}

If err := codec.Write(responses); err != nil {
log.Error(fmt.Sprintf("%v\n", err))
codec.Close()
}

// when request holds one of more subscribe requests this allows these subscriptions to be activated
For _, c := range callbacks {
c()
}
}

Handle method, execute a request, then return response

// handle executes a request and returns the response from the callback.
Func (s *Server) handle(ctx context.Context, codec ServerCodec, req *serverRequest) (interface{}, func()) {
If req.err != nil {
Return codec.CreateErrorResponse(&amp;req.id, req.err), nil
}
//If it is a message to unsubscribe. NotifierFromContext(ctx) gets the notifier we stored in ctx before.
If req.isUnsubscribe { // cancel subscription, first param must be the subscription id
If len(req.args) &gt;= 1 &amp;&amp; req.args[0].Kind() == reflect.String {
Notifier, supported := NotifierFromContext(ctx)
If !supported { // interface doesn't support subscriptions (e.g. http)
Return codec.CreateErrorResponse(&amp;req.id, &amp;callbackError{ErrNotificationsUnsupported.Error()}), nil
}

Subid := ID(req.args[0].String())
If err := notifier.unsubscribe(subid); err != nil {
Return codec.CreateErrorResponse(&amp;req.id, &amp;callbackError{err.Error()}), nil
}

Return codec.CreateResponse(req.id, true), nil
}
Return codec.CreateErrorResponse(&amp;req.id, &amp;invalidParamsError{"Expected subscription id as first argument"}), nil
}
//If it is a subscription message. Then create a subscription. And activate the subscription.
If req.callb.isSubscribe {
Subid, err := s.createSubscription(ctx, codec, req)
If err != nil {
Return codec.CreateErrorResponse(&amp;req.id, &amp;callbackError{err.Error()}), nil
}

// active the subscription after the sub id was successfully sent to the client
activateSub := func() {
Notifier, _ := NotifierFromContext(ctx)
Notifier.activate(subid, req.svcname)
}

Return codec.CreateResponse(req.id, subid), activateSub
}

// regular RPC call, prepare arguments
If len(req.args) != len(req.callb.argTypes) {
rpcErr := &amp;invalidParamsError{fmt.Sprintf("%s%s%s expects %d parameters, got %d",
Req.svcname, serviceMethodSeparator, req.callb.method.Name,
Len(req.callb.argTypes), len(req.args))}
Return codec.CreateErrorResponse(&amp;req.id, rpcErr), nil
}

Arguments := []reflect.Value{req.callb.rcvr}
If req.callb.hasCtx {
Arguments = append(arguments, reflect.ValueOf(ctx))
}
If len(req.args) &gt; 0 {
Arguments = append(arguments, req.args...)
}
/ / Call the provided rpc method, and get reply
// execute RPC method and return result
Reply := req.callb.method.Func.Call(arguments)
If len(reply) == 0 {
Return codec.CreateResponse(req.id, nil), nil
}

If req.callb.errPos &gt;= 0 { // test if method returned an error
If !reply[req.callb.errPos].IsNil() {
e := reply[req.callb.errPos].Interface().(error)
Res := codec.CreateErrorResponse(&amp;req.id, &amp;callbackError{e.Error()})
Return res, nil
}
}
Return codec.CreateResponse(req.id, reply[0].Interface()), nil
}

### subscription.go Publish subscription mode.
In the previous server.go there have been some code to publish the subscription mode, which is elaborated here.

We have this code in the code of the serveRequest.

If codec supports it, you can send a message to the client via a callback function called notifier.
He has a close relationship with codec/connection. If the connection is closed, the notifier will close and all active subscriptions will be revoked.
// if the codec supports notification include a notifier that callbacks can use
// to send notification to clients. It is thight to the codec/connection. If the
// connection is closed the notifier will stop and cancels all active subscriptions.
If options&amp;OptionSubscriptions == OptionSubscriptions {
Ctx = context.WithValue(ctx, notifierKey{}, newNotifier(codec))
}

When serving a client connection, the newNotifier method is called to create a notifier object stored in ctx. It can be observed that the Notifier object holds an instance of codec, which means that the Notifier object holds the network connection and is used to send data when needed.

// newNotifier creates a new notifier that can be used to send subscription
// notifications to the client.
Func newNotifier(codec ServerCodec) *Notifier {
Return &amp;Notifier{
Codec: codec,
Active: make(map[ID]*Subscription),
Inactive: make(map[ID]*Subscription),
}
}

Then in the handle method, we handle a special kind of method, which is identified as isSubscribe. Calling the createSubscription method creates a Subscription and calls the notifier.activate method to store it in the activation queue of the notifier. There is a trick in the code. After the method call is completed, the subscription is not directly activated, but the code of the active part is returned as a function. Then wait for codec.CreateResponse(req.id, subid) in the exec or execBatch code to be called after the response is sent to the client. If the client does not receive the subscription ID, it will receive the subscription information.

If req.callb.isSubscribe {
Subid, err := s.createSubscription(ctx, codec, req)
If err != nil {
Return codec.CreateErrorResponse(&amp;req.id, &amp;callbackError{err.Error()}), nil
}

// active the subscription after the sub id was successfully sent to the client
activateSub := func() {
Notifier, _ := NotifierFromContext(ctx)
Notifier.activate(subid, req.svcname)
}

Return codec.CreateResponse(req.id, subid), activateSub
}

The createSubscription method will call the specified registered method and get a response.

// createSubscription will call the subscription callback and returns the subscription id or error.
Func (s *Server) createSubscription(ctx context.Context, c ServerCodec, req *serverRequest) (ID, error) {
// subscription have as first argument the context following optional arguments
Args := []reflect.Value{req.callb.rcvr, reflect.ValueOf(ctx)}
Args = append(args, req.args...)
Reply := req.callb.method.Func.Call(args)

If !reply[1].IsNil() { // subscription creation failed
Return "", reply[1].Interface().(error)
}

Return reply[0].Interface().(*Subscription).ID, nil
}

Take a look at our activate method, which activates the subscription. The subscription is activated after the subscription ID is sent to the client, preventing the client from receiving the subscription information when it has not received the subscription ID.

// activate enables a subscription. Until a subscription is enabled all
// notifications are dropped. This method is called by the RPC server after
// the subscription ID was sent to client. This prevents notifications being
// send to the client before the subscription ID is send to the client.
Func (n *Notifier) ​​activate(id ID, namespace string) {
n.subMu.Lock()
Defer n.subMu.Unlock()
If sub, found := n.inactive[id]; found {
Sub.namespace = namespace
N.active[id] = sub
Delete(n.inactive, id)
}
}

Let's look at a function to unsubscribe.

// unsubscribe a subscription.
// If the subscription could not be found ErrSubscriptionNotFound is returned.
Func (n *Notifier) ​​unsubscribe(id ID) error {
n.subMu.Lock()
Defer n.subMu.Unlock()
If s, found := n.active[id]; found {
Close(s.err)
Delete(n.active, id)
Return nil
}
Return ErrSubscriptionNotFound
}

Finally, a function that sends a subscription, calling this function to send data to the client, is also relatively simple.

// Notify sends a notification to the client with the given data as payload.
// If an error occurs the RPC connection is closed and the error is returned.
Func (n *Notifier) ​​Notify(id ID, data interface{}) error {
n.subMu.RLock()
Defer n.subMu.RUnlock()

Sub, active := n.active[id]
If active {
Notification := n.codec.CreateNotification(string(id), sub.namespace, data)
If err := n.codec.Write(notification); err != nil {
n.codec.Close()
Return err
}
}
Return nil
}


How to use the recommendations through the test_test.go TestNotifications to view the complete process.

### client.go RPC client source code analysis.

The main function of the client is to send the request to the server, then receive the response, and then pass the response to the caller.

Client data structure

// Client represents a connection to an RPC server.
Type Client struct {
idCounter uint32
/ / Generate a connection function, the client will call this function to generate a network connection object.
connectFunc func(ctx context.Context) (net.Conn, error)
//The HTTP protocol and the non-HTTP protocol have different processing flow. The HTTP protocol does not support long connections. It only supports one mode for requesting one response, and does not support the publish/subscribe mode.
isHTTP bool

// writeConn is only safe to access outside dispatch, with the
// write lock held. The write lock is taken by sending on
// requestOp and released by sending on sendDone.
// As you can see from the comments here, writeConn is the network connection object that is used to write the request.
// Only call outside the dispatch method is safe, and you need to get the lock by sending a request to the requestOp queue.
// After the lock is acquired, the request can be written to the network. After the write is completed, the request is sent to the sendDone queue to release the lock for use by other requests.
writeConn net.Conn

// for dispatch
/ / There are a lot of channels below, channel is generally used to communicate between the goroutine, followed by the code to introduce how the channel is used.
Close chan struct{}
didQuit chan struct{} //close when client quits
Reconnected chan net.Conn // where write/reconnect sends the new connection
readErr chan error // errors from read
readResp chan []*jsonrpcMessage // valid messages from read
		requestOp   chan *requestOp                // for registering response IDs
		sendDone    chan error                     // signals write completion, releases write lock
		respWait    map[string]*requestOp          // active requests
		subs        map[string]*ClientSubscription // active subscriptions
}


newClient， 新建一个客户端。 通过调用connectFunc方法来获取一个网络连接，如果网络连接是httpConn对象的化，那么isHTTP设置为true。然后是对象的初始化， 如果是HTTP连接的化，直接返回，否者就启动一个goroutine调用dispatch方法。 dispatch方法是整个client的指挥中心，通过上面提到的channel来和其他的goroutine来进行通信，获取信息，根据信息做出各种决策。后续会详细介绍dispatch。 因为HTTP的调用方式非常简单， 这里先对HTTP的方式做一个简单的阐述。


	func newClient(initctx context.Context, connectFunc func(context.Context) (net.Conn, error)) (*Client, error) {
		conn, err := connectFunc(initctx)
If err != nil {
Return nil, err
}
		_, isHTTP := conn.(*httpConn)

		c := &amp;Client{
			writeConn:   conn,
			isHTTP:      isHTTP,
			connectFunc: connectFunc,
			close:       make(chan struct{}),
			didQuit:     make(chan struct{}),
			reconnected: make(chan net.Conn),
			readErr:     make(chan error),
			readResp:    make(chan []*jsonrpcMessage),
			requestOp:   make(chan *requestOp),
			sendDone:    make(chan error, 1),
			respWait:    make(map[string]*requestOp),
			subs:        make(map[string]*ClientSubscription),
}
		if !isHTTP {
			go c.dispatch(conn)
}
		return c, nil
}


请求调用通过调用client的 Call方法来进行RPC调用。

	// Call performs a JSON-RPC call with the given arguments and unmarshals into
	// result if no error occurred.
//
	// The result must be a pointer so that package json can unmarshal into it. You
	// can also pass nil, in which case the result is ignored.
	返回值必须是一个指针，这样才能把json值转换成对象。 如果你不关心返回值，也可以通过传nil来忽略。
	func (c *Client) Call(result interface{}, method string, args ...interface{}) error {
		ctx := context.Background()
		return c.CallContext(ctx, result, method, args...)
}

	func (c *Client) CallContext(ctx context.Context, result interface{}, method string, args ...interface{}) error {
		msg, err := c.newMessage(method, args...)
If err != nil {
Return err
}
		//构建了一个requestOp对象。 resp是读取返回的队列，队列的长度是1。
		op := &amp;requestOp{ids: []json.RawMessage{msg.ID}, resp: make(chan *jsonrpcMessage, 1)}

		if c.isHTTP {
			err = c.sendHTTP(ctx, op, msg)
} else {
			err = c.send(ctx, op, msg)
}
If err != nil {
Return err
}

		// dispatch has accepted the request and will close the channel it when it quits.
		switch resp, err := op.wait(ctx); {
		case err != nil:
Return err
		case resp.Error != nil:
			return resp.Error
		case len(resp.Result) == 0:
			return ErrNoResult
Default:
			return json.Unmarshal(resp.Result, &amp;result)
}
}

sendHTTP,这个方法直接调用doRequest方法进行请求拿到回应。然后写入到resp队列就返回了。

	func (c *Client) sendHTTP(ctx context.Context, op *requestOp, msg interface{}) error {
		hc := c.writeConn.(*httpConn)
		respBody, err := hc.doRequest(ctx, msg)
If err != nil {
Return err
}
		defer respBody.Close()
		var respmsg jsonrpcMessage
		if err := json.NewDecoder(respBody).Decode(&amp;respmsg); err != nil {
Return err
}
		op.resp &lt;- &amp;respmsg
Return nil
}


在看看上面的另一个方法 op.wait()方法，这个方法会查看两个队列的信息。如果是http那么从resp队列获取到回应就会直接返回。 这样整个HTTP的请求过程就完成了。 中间没有涉及到多线程问题，都在一个线程内部完成了。

	func (op *requestOp) wait(ctx context.Context) (*jsonrpcMessage, error) {
Select {
		case &lt;-ctx.Done():
			return nil, ctx.Err()
		case resp := &lt;-op.resp:
			return resp, op.err
}
}

如果不是HTTP请求呢。 那处理的流程就比较复杂了， 还记得如果不是HTTP请求。在newClient的时候是启动了一个goroutine 调用了dispatch方法。 我们先看非http的 send方法。

从注释来看。这个方法把op写入到requestOp这个队列，注意的是这个队列是没有缓冲区的，也就是说如果这个时候这个队列没有人处理的化，这个调用是会阻塞在这里的。这就相当于一把锁，如果发送op到requestOp成功了就拿到了锁，可以继续下一步，下一步是调用write方法把请求的全部内容发送到网络上。然后发送消息给sendDone队列。 sendDone可以看成是锁的释放，后续在dispatch方法里面会详细分析这个过程。 Then return.返回之后方法会阻塞在op.wait方法里面。直到从op.resp队列收到一个回应，或者是收到一个ctx.Done()消息(这个消息一般会在完成或者是强制退出的时候获取到。)

	// send registers op with the dispatch loop, then sends msg on the connection.
	// if sending fails, op is deregistered.
	func (c *Client) send(ctx context.Context, op *requestOp, msg interface{}) error {
Select {
		case c.requestOp &lt;- op:
			log.Trace("", "msg", log.Lazy{Fn: func() string {
				return fmt.Sprint("sending ", msg)
			}})
			err := c.write(ctx, msg)
			c.sendDone &lt;- err
Return err
		case &lt;-ctx.Done():
			// This can happen if the client is overloaded or unable to keep up with
			// subscription notifications.
			return ctx.Err()
		case &lt;-c.didQuit:
			//已经退出，可能被调用了Close
			return ErrClientQuit
}
}

dispatch方法


	// dispatch is the main loop of the client.
	// It sends read messages to waiting calls to Call and BatchCall
	// and subscription notifications to registered subscriptions.
	func (c *Client) dispatch(conn net.Conn) {
		// Spawn the initial read loop.
		go c.read(conn)

Var (
			lastOp        *requestOp    // tracks last send operation
			requestOpLock = c.requestOp // nil while the send lock is held
			reading       = true        // if true, a read loop is running
)
		defer close(c.didQuit)
Defer func() {
			c.closeRequestOps(ErrClientQuit)
			conn.Close()
			if reading {
				// Empty read channels until read is dead.
For {
Select {
					case &lt;-c.readResp:
					case &lt;-c.readErr:
Return
}
}
}
}()

For {
Select {
			case &lt;-c.close:
Return

			// Read path.
			case batch := &lt;-c.readResp:
				//读取到一个回应。调用相应的方法处理
				for _, msg := range batch {
					switch {
					case msg.isNotification():
						log.Trace("", "msg", log.Lazy{Fn: func() string {
							return fmt.Sprint("&lt;-readResp: notification ", msg)
						}})
						c.handleNotification(msg)
					case msg.isResponse():
						log.Trace("", "msg", log.Lazy{Fn: func() string {
							return fmt.Sprint("&lt;-readResp: response ", msg)
						}})
						c.handleResponse(msg)
					default:
						log.Debug("", "msg", log.Lazy{Fn: func() string {
							return fmt.Sprint("&lt;-readResp: dropping weird message", msg)
						}})
						// TODO: maybe close
}
}

			case err := &lt;-c.readErr:
				//接收到读取失败信息，这个是read线程传递过来的。
				log.Debug(fmt.Sprintf("&lt;-readErr: %v", err))
				c.closeRequestOps(err)
				conn.Close()
				reading = false

			case newconn := &lt;-c.reconnected:
				//接收到一个重连接信息
				log.Debug(fmt.Sprintf("&lt;-reconnected: (reading=%t) %v", reading, conn.RemoteAddr()))
				if reading {
					//等待之前的连接读取完成。
					// Wait for the previous read loop to exit. This is a rare case.
					conn.Close()
					&lt;-c.readErr
}
				//开启阅读的goroutine
				go c.read(newconn)
				reading = true
				conn = newconn

			// Send path.
			case op := &lt;-requestOpLock:
				// Stop listening for further send ops until the current one is done.
				//接收到一个requestOp消息，那么设置requestOpLock为空，
				//这个时候如果有其他人也希望发送op到requestOp，会因为没有人处理而阻塞。
				requestOpLock = nil
				lastOp = op
				//把这个op加入等待队列。
				for _, id := range op.ids {
					c.respWait[string(id)] = op
}

			case err := &lt;-c.sendDone:
				//当op的请求信息已经发送到网络上。会发送信息到sendDone。如果发送过程出错，那么err !=nil。
If err != nil {
					// Remove response handlers for the last send. We remove those here
					// because the error is already handled in Call or BatchCall. When the
					// read loop goes down, it will signal all other current operations.
					//把所有的id从等待队列删除。
					for _, id := range lastOp.ids {
						delete(c.respWait, string(id))
}
}
				// Listen for send ops again.
				//重新开始处理requestOp的消息。
				requestOpLock = c.requestOp
				lastOp = nil
}
}
}


下面通过下面这种图来说明dispatch的主要流程。下面图片中圆形是线程。 蓝色矩形是channel。 箭头代表了channel的数据流动方向。

![image](picture/rpc_2.png)

- 多线程串行发送请求到网络上的流程 首先发送requestOp请求到dispatch获取到锁， 然后把请求信息写入到网络，然后发送sendDone信息到dispatch解除锁。 通过requestOp和sendDone这两个channel以及dispatch代码的配合完成了串行的发送请求到网络上的功能。
- 读取返回信息然后返回给调用者的流程。 把请求信息发送到网络上之后， 内部的goroutine read会持续不断的从网络上读取信息。 read读取到返回信息之后，通过readResp队列发送给dispatch。 dispatch查找到对应的调用者，然后把返回信息写入调用者的resp队列中。完成返回信息的流程。
- 重连接流程。 重连接在外部调用者写入失败的情况下被外部调用者主动调用。 调用完成后发送新的连接给dispatch。 dispatch收到新的连接之后，会终止之前的连接，然后启动新的read goroutine来从新的连接上读取信息。
- 关闭流程。 调用者调用Close方法，Close方法会写入信息到close队列。 dispatch接收到close信息之后。 关闭didQuit队列，关闭连接，等待read goroutine停止。 所有等待在didQuit队列上面的客户端调用全部返回。


#### 客户端 订阅模式的特殊处理
上面提到的主要流程是方法调用的流程。 以太坊的RPC框架还支持发布和订阅的模式。 

我们先看看订阅的方法，以太坊提供了几种主要service的订阅方式(EthSubscribe ShhSubscribe).同时也提供了自定义服务的订阅方法(Subscribe)，


	// EthSubscribe registers a subscripion under the "eth" namespace.
	func (c *Client) EthSubscribe(ctx context.Context, channel interface{}, args ...interface{}) (*ClientSubscription, error) {
		return c.Subscribe(ctx, "eth", channel, args...)
}

	// ShhSubscribe registers a subscripion under the "shh" namespace.
	func (c *Client) ShhSubscribe(ctx context.Context, channel interface{}, args ...interface{}) (*ClientSubscription, error) {
		return c.Subscribe(ctx, "shh", channel, args...)
}

	// Subscribe calls the "&lt;namespace&gt;_subscribe" method with the given arguments,
	// registering a subscription. Server notifications for the subscription are
	// sent to the given channel. The element type of the channel must match the
	// expected type of content returned by the subscription.
//
	// The context argument cancels the RPC request that sets up the subscription but has no
	// effect on the subscription after Subscribe has returned.
//
	// Slow subscribers will be dropped eventually. Client buffers up to 8000 notifications
	// before considering the subscriber dead. The subscription Err channel will receive
	// ErrSubscriptionQueueOverflow. Use a sufficiently large buffer on the channel or ensure
	// that the channel usually has at least one reader to prevent this issue.
	//Subscribe会使用传入的参数调用"&lt;namespace&gt;_subscribe"方法来订阅指定的消息。 
	//服务器的通知会写入channel参数指定的队列。 channel参数必须和返回的类型相同。 
	//ctx参数可以用来取消RPC的请求，但是如果订阅已经完成就不会有效果了。 
	//处理速度太慢的订阅者的消息会被删除，每个客户端有8000个消息的缓存。
	func (c *Client) Subscribe(ctx context.Context, namespace string, channel interface{}, args ...interface{}) (*ClientSubscription, error) {
		// Check type of channel first.
		chanVal := reflect.ValueOf(channel)
		if chanVal.Kind() != reflect.Chan || chanVal.Type().ChanDir()&amp;reflect.SendDir == 0 {
			panic("first argument to Subscribe must be a writable channel")
}
		if chanVal.IsNil() {
			panic("channel given to Subscribe must not be nil")
}
		if c.isHTTP {
			return nil, ErrNotificationsUnsupported
}

		msg, err := c.newMessage(namespace+subscribeMethodSuffix, args...)
If err != nil {
Return nil, err
}
		//requestOp的参数和Call调用的不一样。 多了一个参数sub.
		op := &amp;requestOp{
			ids:  []json.RawMessage{msg.ID},
			resp: make(chan *jsonrpcMessage),
			sub:  newClientSubscription(c, namespace, chanVal),
}

		// Send the subscription request.
		// The arrival and validity of the response is signaled on sub.quit.
		if err := c.send(ctx, op, msg); err != nil {
Return nil, err
}
		if _, err := op.wait(ctx); err != nil {
Return nil, err
}
		return op.sub, nil
}

newClientSubscription方法，这个方法创建了一个新的对象ClientSubscription，这个对象把传入的channel参数保存起来。 然后自己又创建了三个chan对象。后续会对详细介绍这三个chan对象


	func newClientSubscription(c *Client, namespace string, channel reflect.Value) *ClientSubscription {
		sub := &amp;ClientSubscription{
			client:    c,
			namespace: namespace,
			etype:     channel.Type().Elem(),
			channel:   channel,
			quit:      make(chan struct{}),
			err:       make(chan error, 1),
			in:        make(chan json.RawMessage),
}
Return sub
}

从上面的代码可以看出。订阅过程根Call过程差不多，构建一个订阅请求。调用send发送到网络上，然后等待返回。 我们通过dispatch对返回结果的处理来看看订阅和Call的不同。


	func (c *Client) handleResponse(msg *jsonrpcMessage) {
		op := c.respWait[string(msg.ID)]
		if op == nil {
			log.Debug(fmt.Sprintf("unsolicited response %v", msg))
Return
}
		delete(c.respWait, string(msg.ID))
		// For normal responses, just forward the reply to Call/BatchCall.
		如果op.sub是nil，普通的RPC请求，这个字段的值是空白的，只有订阅请求才有值。
		if op.sub == nil {
			op.resp &lt;- msg
Return
}
		// For subscription responses, start the subscription if the server
		// indicates success. EthSubscribe gets unblocked in either case through
		// the op.resp channel.
		defer close(op.resp)
		if msg.Error != nil {
			op.err = msg.Error
Return
}
		if op.err = json.Unmarshal(msg.Result, &amp;op.sub.subid); op.err == nil {
			//启动一个新的goroutine 并把op.sub.subid记录起来。
			go op.sub.start()
			c.subs[op.sub.subid] = op.sub
}
}


op.sub.start方法。 这个goroutine专门用来处理订阅消息。主要的功能是从in队列里面获取订阅消息，然后把订阅消息放到buffer里面。 如果能够数据能够发送。就从buffer里面发送一些数据给用户传入的那个channel。 如果buffer超过指定的大小，就丢弃。


	func (sub *ClientSubscription) start() {
		sub.quitWithError(sub.forward())
}

	func (sub *ClientSubscription) forward() (err error, unsubscribeServer bool) {
		cases := []reflect.SelectCase{
			{Dir: reflect.SelectRecv, Chan: reflect.ValueOf(sub.quit)},
			{Dir: reflect.SelectRecv, Chan: reflect.ValueOf(sub.in)},
			{Dir: reflect.SelectSend, Chan: sub.channel},
}
		buffer := list.New()
		defer buffer.Init()
For {
			var chosen int
			var recv reflect.Value
			if buffer.Len() == 0 {
				// Idle, omit send case.
				chosen, recv, _ = reflect.Select(cases[:2])
} else {
				// Non-empty buffer, send the first queued item.
				cases[2].Send = reflect.ValueOf(buffer.Front().Value)
				chosen, recv, _ = reflect.Select(cases)
}

			switch chosen {
			case 0: // &lt;-sub.quit
				return nil, false
			case 1: // &lt;-sub.in
				val, err := sub.unmarshal(recv.Interface().(json.RawMessage))
If err != nil {
					return err, true
}
				if buffer.Len() == maxClientSubscriptionBuffer {
					return ErrSubscriptionQueueOverflow, true
}
				buffer.PushBack(val)
			case 2: // sub.channel&lt;-
				cases[2].Send = reflect.Value{} // Don't hold onto the value.
				buffer.Remove(buffer.Front())
}
}
}


当接收到一条Notification消息的时候会调用handleNotification方法。会把消息传送给in队列。

	func (c *Client) handleNotification(msg *jsonrpcMessage) {
		if !strings.HasSuffix(msg.Method, notificationMethodSuffix) {
			log.Debug(fmt.Sprint("dropping non-subscription message: ", msg))
Return
}
		var subResult struct {
			ID     string          `json:"subscription"`
			Result json.RawMessage `json:"result"`
}
		if err := json.Unmarshal(msg.Params, &amp;subResult); err != nil {
			log.Debug(fmt.Sprint("dropping invalid subscription message: ", msg))
Return
}
		if c.subs[subResult.ID] != nil {
			c.subs[subResult.ID].deliver(subResult.Result)
}
}
	func (sub *ClientSubscription) deliver(result json.RawMessage) (ok bool) {
Select {
		case sub.in &lt;- result:
Return true
		case &lt;-sub.quit:
Return false
}
}

</pre></body></html>