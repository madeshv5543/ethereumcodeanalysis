
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Inside the p2p code. The peer represents a created network link. Multiple protocols may be running on a single link. For example, the agreement of Ethereum (eth). Swarm's agreement. Or the agreement of Whisper.

Peer structure

Type protoRW struct {
Protocol
In chan Msg // receices read messages
Closed &lt;-chan struct{} // receive when peer is shutting down
Wstart &lt;-chan struct{} // receive when write may start
Werr chan&lt;- error // for write results
Offset uint64
w MsgWriter
}

// Protocol represents a P2P subprotocol implementation.
Type Protocol struct {
// Name should contain the official protocol name,
// often a three-letter word.
Name string

// Version should contain the version number of the protocol.
Version uint

// Length should contain the number of message codes used
// by the protocol.
Length uint64

// Run is called in a new groutine when the protocol has been
// negotiated with a peer. It should read and write messages from
// rw. The Payload for each message must be fully consumed.
//
// The peer connection is closed when Start returns. It should return
// any protocol-level error (such as an I/O error) that is
//Having encountered.
Run func(peer *Peer, rw MsgReadWriter) error

// NodeInfo is an optional helper method to retrieve protocol specific metadata
// about the host node.
NodeInfo func() interface{}

// PeerInfo is an optional helper method to retrieve protocol specific metadata
//about a certain peer in the network. If an info retrieval function is set,
// but returns nil, it is assumed that the protocol handshake is still running.
PeerInfo func(id discover.NodeID) interface{}
}

// Peer represents a connected remote node.
Type Peer struct {
Rw *conn
Running map[string]*protoRW //Running protocol
Log log.Logger
Created mclock.AbsTime

Wg sync.WaitGroup
protoErr chan error
Closed chan struct{}
Disc chan DiscReason

// events receives message send / receive events if set
Events *event.Feed
}

The peer is created, and the protomap supported by the current peer is found according to the match.

Func newPeer(conn *conn, protocols []Protocol) *Peer {
Protomap := matchProtocols(protocols, conn.caps, conn)
p := &amp;Peer{
Rw: conn,
Running: protomap,
Created: mclock.Now(),
Disc: make(chan DiscReason),
protoErr: make(chan error, len(protomap)+1), // protocols + pingLoop
Closed: make(chan struct{}),
Log: log.New("id", conn.id, "conn", conn.flags),
}
Return p
}

The peer starts, and two goroutine threads are started. One is reading. One is to perform a ping operation.

Func (p *Peer) run() (remoteRequested bool, err error) {
Var (
writeStart = make(chan struct{}, 1) // is used to control when the pipe can be written.
writeErr = make(chan error, 1)
readErr = make(chan error, 1)
Reason DiscReason // sent to the peer
)
p.wg.Add(2)
Go p.readLoop(readErr)
Go p.pingLoop()

// Start all protocol handlers.
writeStart &lt;- struct{}{}
//Start all protocols.
p.startProtocols(writeStart, writeErr)

// Wait for an error or disconnect.
Loop:
For {
Select {
Case err = &lt;-writeErr:
// A write finished. Allow the next write to start if
// there was no error.
If err != nil {
Reason = DiscNetworkError
Break loop
}
writeStart &lt;- struct{}{}
Case err = &lt;-readErr:
If r, ok := err.(DiscReason); ok {
remoteRequested = true
Reason = r
} else {
Reason = DiscNetworkError
}
Break loop
Case err = &lt;-p.protoErr:
Reason = discReasonForError(err)
Break loop
Case err = &lt;-p.disc:
Break loop
}
}

Close(p.closed)
P.rw.close(reason)
p.wg.Wait()
Return remoteRequested, err
}

The startProtocols method, which traverses all protocols.

Func (p *Peer) startProtocols(writeStart &lt;-chan struct{}, writeErr chan&lt;- error) {
p.wg.Add(len(p.running))
For _, proto := range p.running {
Proto := proto
Proto.closed = p.closed
Proto.wstart = writeStart
Proto.werr = writeErr
Var rw MsgReadWriter = proto
If p.events != nil {
Rw = newMsgEventer(rw, p.events, p.ID(), proto.Name)
}
p.log.Trace(fmt.Sprintf("Starting protocol %s/%d", proto.Name, proto.Version))
// This is equivalent to opening a goroutine for each protocol. Call its Run method.
Go func() {
// proto.Run(p, rw) This method should be an infinite loop. If you return, you have encountered an error.
Err := proto.Run(p, rw)
If err == nil {
p.log.Trace(fmt.Sprintf("Protocol %s/%d returned", proto.Name, proto.Version))
Err = errProtocolReturned
} else if err != io.EOF {
p.log.Trace(fmt.Sprintf("Protocol %s/%d failed", proto.Name, proto.Version), "err", err)
}
p.protoErr &lt;- err
p.wg.Done()
}()
}
}


Go back and look at the readLoop method. This method is also an infinite loop. Call p.rw to read an Msg (this rw is actually the object of the frameRLPx mentioned earlier, that is, the object after the framing. Then the corresponding processing is performed according to the type of Msg, if the type of Msg is the protocol of the internal running Type. Then send it to the corresponding protocol's proto.in queue.


Func (p *Peer) readLoop(errc chan&lt;- error) {
Defer p.wg.Done()
For {
Msg, err := p.rw.ReadMsg()
If err != nil {
Errc &lt;- err
Return
}
msg.ReceivedAt = time.Now()
If err = p.handle(msg); err != nil {
Errc &lt;- err
Return
}
}
}
Func (p *Peer) handle(msg Msg) error {
Switch {
Case msg.Code == pingMsg:
msg.Discard()
Go SendItems(p.rw, pongMsg)
Case msg.Code == discMsg:
Var reason [1]DiscReason
// This is the last message. We don't need to discard or
// check errors because, the connection will be closed after it.
rlp.Decode(msg.Payload, &amp;reason)
Return reason[0]
Case msg.Code &lt; baseProtocolLength:
// ignore other base protocol messages
Return msg.Discard()
Default:
// it's a subprotocol message
Proto, err := p.getProto(msg.Code)
If err != nil {
Return fmt.Errorf("msg code out of range: %v", msg.Code)
}
Select {
Case proto.in &lt;- msg:
Return nil
Case &lt;-p.closed:
Return io.EOF
}
}
Return nil
}

Take a look at pingLoop. This method is very simple. It is time to send a pingMsg message to the peer.

Func (p *Peer) pingLoop() {
Ping := time.NewTimer(pingInterval)
Defer p.wg.Done()
Defer ping.Stop()
For {
Select {
Case &lt;-ping.C:
If err := SendItems(p.rw, pingMsg); err != nil {
p.protoErr &lt;- err
Return
}
ping.Reset(pingInterval)
Case &lt;-p.closed:
Return
}
}
}

Finally, look at the read and write methods of protoRW. You can see that both reads and writes are blocking.

Func (rw *protoRW) WriteMsg(msg Msg) (err error) {
If msg.Code &gt;= rw.Length {
Return newPeerError(errInvalidMsgCode, "not handled")
}
msg.Code += rw.offset
Select {
Case &lt;-rw.wstart: // Wait until the write that can be written is executed. Is this for multi-threaded control?
Err = rw.w.WriteMsg(msg)
// Report write status back to Peer.run. It will initiate
// shutdown if the error is non-nil and unblock the next write
// otherwise. The calling protocol code should exit for errors
// as well but we don't want to rely on that.
Rw.werr &lt;- err
Case &lt;-rw.closed:
Err = fmt.Errorf("shutting down")
}
Return err
}

Func (rw *protoRW) ReadMsg() (Msg, error) {
Select {
Case msg := &lt;-rw.in:
msg.Code -= rw.offset
Return msg, nil
Case &lt;-rw.closed:
Return Msg{}, io.EOF
}
}
</pre></body></html>