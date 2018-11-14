
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>The event package implements the event publishing and subscription mode within the same process.

## event.go

Currently this part of the code is marked as Deprecated, telling the user to use the feed object. However, it is still used in the code. And there is not much code in this part. Just a brief introduction.

data structure
TypeMux is the main use. Subm records all subscribers. You can see that there are many subscribers for each type.

// TypeMuxEvent is a time-tagged notification pushed to subscribers.
Type TypeMuxEvent struct {
Time time.Time
Data interface{}
}

// A TypeMux dispatches events to registered receivers. Receivers can be
//registered to handle events of certain type. Any operation
// called after mux is stopped will return ErrMuxClosed.
//
// The zero value is ready to use.
//
// Deprecated: use Feed
Type TypeMux struct {
Mutex sync.RWMutex
Subm map[reflect.Type][]*TypeMuxSubscription
Stopped bool
}


Create a subscription that can subscribe to multiple types at the same time.

// Subscribe creates a subscription for events of the given types. The
// subscription's channel is closed when it is unsubscribed
// or the mux is closed.
Func (mux *TypeMux) Subscribe(types ...interface{}) *TypeMuxSubscription {
Sub := newsub(mux)
mux.mutex.Lock()
Defer mux.mutex.Unlock()
If mux.stopped {
// set the status to closed so that calling Unsubscribe after this
// call will short circuit.
Sub.closed = true
Close(sub.postC)
} else {
If mux.subm == nil {
Mux.subm = make(map[reflect.Type][]*TypeMuxSubscription)
}
For _, t := range types {
Rtyp := reflect.TypeOf(t)
Oldsubs := mux.subm[rtyp]
If find(oldsubs, sub) != -1 {
Panic(fmt.Sprintf("event: duplicate type %s in Subscribe", rtyp))
}
Subs := make([]*TypeMuxSubscription, len(oldsubs)+1)
Copy(subs, oldsubs)
Subs[len(oldsubs)] = sub
Mux.subm[rtyp] = subs
}
}
Return sub
}

// TypeMuxSubscription is a subscription established through TypeMux.
Type TypeMuxSubscription struct {
Mux *TypeMux
Created time.Time
closeMu sync.Mutex
Closing chan struct{}
Closed bool

// these two are the same channel. they are stored separately so
// postC can be set to nil without affecting the return value of
// Chan.
postMu sync.RWMutex
// readC and postC are actually the same channel. But one is reading from the channel, one is only written from the channel
// Single direction channel
readC &lt;-chan *TypeMuxEvent
postC chan&lt;- *TypeMuxEvent
}

Func newsub(mux *TypeMux) *TypeMuxSubscription {
c := make(chan *TypeMuxEvent)
Return &amp;TypeMuxSubscription{
Mux: mux,
Created: time.Now(),
readC: c,
postC: c,
Closing: make(chan struct{}),
}
}

Publish an event to TypeMux. At this time, all messages that subscribe to this type will receive this message.

// Post sends an event to all receivers registered for the given type.
// It returns ErrMuxClosed if the mux has been stopped.
Func (mux *TypeMux) Post(ev interface{}) error {
Event := &amp;TypeMuxEvent{
Time: time.Now(),
Data: ev,
}
Rtyp := reflect.TypeOf(ev)
mux.mutex.RLock()
If mux.stopped {
mux.mutex.RUnlock()
Return ErrMuxClosed
}
Subs := mux.subm[rtyp]
mux.mutex.RUnlock()
For _, sub := range subs {
// Blocked delivery.
Sub.deliver(event)
}
Return nil
}


Func (s *TypeMuxSubscription) deliver(event *TypeMuxEvent) {
// Short circuit delivery if stale event
If s.created.After(event.Time) {
Return
}
// Otherwise deliver the event
s.postMu.RLock()
Defer s.postMu.RUnlock()

Select { // blocking method
Case s.postC &lt;- event:
Case &lt;-s.closing:
}
}


## feed.go
The main object currently used. Replaced the TypeMux inside the event.go.

Feed data structure

// Feed implements one-to-many subscriptions where the carrier of events is a channel.
// Values ​​sent to a Feed are delivered to all subscribed channels simultaneous.
// The feed implements a one-to-many subscription model that uses channels to pass events. The value sent to the feed is also passed to all subscribed channels.
// Feeds can only be used with a single type. The type is determined by the first Send or
// Subscribe operation. Subsequent calls to these methods panic if the type does not
// match.
// Feeds can only be used by a single type. This is different from the previous event, the event can use multiple types. The type is determined by the first Send call or the Subscribe call. Subsequent calls will panic if the type and its inconsistency
// The zero value is ready to use.
Type feed struct {
Once sync.Once // guarantee that init only works once
sendLock chan struct{} // sendLock has a one-element buffer and is empty when held. It protects sendCases.
removeSub chan interface{} // interrupts Send
sendCases caseList // the active set of select cases used by Send

// The inbox holds newly subscribed channels until they are added to sendCases.
Mu sync.Mutex
Inbox caseList
Etype reflect.Type
Closed bool
}

Initialization Initialization will be guaranteed by once and will only be executed once.

Func (f *Feed) init() {
f.removeSub = make(chan interface{})
f.sendLock = make(chan struct{}, 1)
f.sendLock &lt;- struct{}{}
f.sendCases = caseList{{Chan: reflect.ValueOf(f.removeSub), Dir: reflect.SelectRecv}}
}

Subscribe, subscribe to post a channel. Relative to the event. The event subscription is passed in the type that needs to be subscribed, and then the channel is built and returned in the event's subscription code. This mode of direct delivery of channels may be more flexible.
Then a SelectCase is generated based on the incoming channel. Put it in the inbox.

// Subscribe adds a channel to the feed. Future sends will be delivered on the channel
// until the subscription is canceled. All channels added must have the same element type.
//
// The channel should have ample buffer space to avoid blocking other subscribers.
// Slow subscribers are not dropped.
Func (f *Feed) Subscribe(channel interface{}) Subscription {
f.once.Do(f.init)

Chanval := reflect.ValueOf(channel)
Chantyp := chanval.Type()
If chantyp.Kind() != reflect.Chan || chantyp.ChanDir()&amp;reflect.SendDir == 0 { // If the type is not a channel. Or the direction of the channel cannot send data. Then the error exits.
Panic(errBadChannel)
}
Sub := &amp;feedSub{feed: f, channel: chanval, err: make(chan error, 1)}

f.mu.Lock()
Defer f.mu.Unlock()
If !f.typecheck(chantyp.Elem()) {
Panic(feedTypeError{op: "Subscribe", got: chantyp, want: reflect.ChanOf(reflect.SendDir, f.etype)})
}
// Add the select case to the inbox.
// The next Send will add it to f.sendCases.
Cas := reflect.SelectCase{Dir: reflect.SelectSend, Chan: chanval}
F.inbox = append(f.inbox, cas)
Return sub
}


Send method, the Send method of the feed does not traverse all the channels and then send the blocking mode. This can result in slow clients affecting fast clients. Instead, use SelectCase in a reflective manner. First call the non-blocking TrySend to try to send. This way if there is no slow client. The data will be sent directly to completion. If the TrySend part of the client fails. Then the subsequent is sent in the form of a loop Select. I guess this is why the feed will replace the event.


// Send delivers to all subscribed channels simultaneous.
// It returns the number of subscribers that the value was sent to.
Func (f *Feed) Send(value interface{}) (nsent int) {
f.once.Do(f.init)
&lt;-f.sendLock

// Add new cases from the inbox after taking the send lock.
f.mu.Lock()
f.sendCases = append(f.sendCases, f.inbox...)
F.inbox = nil
f.mu.Unlock()

// Set the sent value on all channels.
Rvalue := reflect.ValueOf(value)
If !f.typecheck(rvalue.Type()) {
f.sendLock &lt;- struct{}{}
Panic(feedTypeError{op: "Send", got: rvalue.Type(), want: f.etype})
}
For i := firstSubSendCase; i &lt; len(f.sendCases); i++ {
f.sendCases[i].Send = rvalue
}

// Send until all channels except removeSub have been chosen.
Cases := f.sendCases
For {
// Fast path: try sending without blocking before adding to the select set.
// This should ingredient succeed if subscribers are fast enough and have free
// buffer space.
For i := firstSubSendCase; i &lt; len(cases); i++ {
If cases[i].Chan.TrySend(rvalue) {
Nsent++
Cases = cases.deactivate(i)
I--
}
}
If len(cases) == firstSubSendCase {
Break
}
// Select on all the receivers, waiting for them to unblock.
Chosen, recv, _ := reflect.Select(cases)
If chosen == 0 /* &lt;-f.removeSub */ {
Index := f.sendCases.find(recv.Interface())
f.sendCases = f.sendCases.delete(index)
If index &gt;= 0 &amp;&amp; index &lt; len(cases) {
Cases = f.sendCases[:len(cases)-1]
}
} else {
Cases = cases.deactivate(chosen)
Nsent++
}
}

// Forget about the sent value and hand off the send lock.
For i := firstSubSendCase; i &lt; len(f.sendCases); i++ {
f.sendCases[i].Send = reflect.Value{}
}
f.sendLock &lt;- struct{}{}
Return nsent
}

</pre></body></html>