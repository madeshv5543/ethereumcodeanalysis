
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>## scheduler.go

The scheduler is a schedule for a single bit value retrieval based on the section's Bloom filter. In addition to scheduling retrieval operations, this structure can deduplicate requests and cache results, minimizing network/database overhead even in complex filtering situations.

### data structure
Request represents a bloom search task to prioritize retrieval from the local database or from the network. Section indicates the block segment number, 4096 blocks per segment, and the bit represents which one of the Bloom filters is retrieved (a total of 2048 bits). This was introduced in the previous (eth-bloombits and filter source code analysis.md).

// request represents a bloom retrieval task to prioritize and pull from the local
// database or remotely from the network.

Type request struct {
Section uint64 // Section index to retrieve the a bit-vector from
Bit uint // Bit index within the section to retrieve the vector of
}

The status of the currently scheduled request. Failure to send a request will generate a response object to finalize the state of the request.
Cached is used to cache the results of this section.

// response represents the state of a requested bit-vector through a scheduler.
Type response struct {
Cached []byte // Cached bits to dedup multiple requests
Done chan struct{} // Channel to allow waiting for completion
}

Scheduler

// scheduler handles the scheduling of bloom-filter
// entire section-batches belonging to a single bloom bit. Beside scheduling the
//edit operations, this struct also deduplicates the requests and caches
// the results to minimize network/database overhead even in complex filtering
// scenarios.
Type scheduler struct {
Bit uint // Index of the bit in the bloom filter this scheduler is responsible for which bit of the Bloom filter (0-2047)
Responses map[uint64]*response // Currently pending retrieval requests or already cached responses The currently in progress request is either a cached result.
Lock sync.Mutex // Lock protecting the responses from concurrent access
}

### Constructor
newScheduler and reset methods

// newScheduler creates a new bloom-filter retrieval scheduler for a specific
// bit index.
Func newScheduler(idx uint) *scheduler {
Return &amp;scheduler{
Bit: idx,
Responses: make(map[uint64]*response),
}
}
// reset cleans up any leftovers from previous runs. This is required before a
// restart to ensure the no previously requested but never delivered state will
// cause a lockup.
The reset usage is used to clean up any previous requests.
Func (s *scheduler) reset() {
s.lock.Lock()
Defer s.lock.Unlock()

For section, res := range s.responses {
If res.cached == nil {
Delete(s.responses, section)
}
}
}


### Running the run method
The run method creates a pipeline, receives the sections that need to be requested from the sections channel, and returns the results in the order of the requests through the donee channel. It is okay to run the same scheduler concurrently, which will cause the task to be repeated.

// run creates a retrieval pipeline, receiving section indexes from sections and
// returning the results in the same order through the done channel. Concurrent
// runs of the same scheduler are allowed, leading to retrieval task deduplication.
Func (s *scheduler) run(sections chan uint64, dist chan *request, done chan []byte, quit chan struct{}, wg *sync.WaitGroup) {
// sections channel type This is the channel used to pass the section to be retrieved, input parameters
// dist channel type, which belongs to the output channel (possibly networked or localized), sends a request to this channel, and then gets a response on done.
// done The channel used to pass the search result, which can be understood as the return value channel.

// Create a forwarder channel between requests and responses of the same size as
// the distribution channel (since that will block the pipeline anyway).
Create a repeater channel of the same size as the distribution channel between the request and the response (because this will block the pipe)
Pend := make(chan uint64, cap(dist))

// Start the pipeline schedulers to forward between user -&gt; distributor -&gt; user
wg.Add(2)
Go s.scheduleRequests(sections, dist, pend, quit, wg)
Go s.scheduleDeliveries(pend, done, quit, wg)
}

### Scheduler flow chart

![image](picture/chainindexer_2.png)
The ellipse in the figure represents the goroutine. The rectangle represents the channel. The triangle represents the external method call.

1. scheduleRequests goroutine receives section messages from sections
2. scheduleRequests assembles the received sections into requtest and sends them to the dist channel, and builds the object response[section]
3. scheduleRequests sends the previous section to the pend queue. scheduleDelivers receives the pend message and blocks it on response[section].done
4. Externally call the deliver method, write the result request of seciton to response[section].cached. and close response[section].done channel
5. scheduleDelivers receives the response[section].done message. Send response[section].cached to the done channel

### scheduleRequests

// scheduleRequests reads section retrieval requests from the input channel,
// deduplicates the stream and pushes unique retrieval tasks into the distribution
// channel for a database or network layer to honour.
Func (s *scheduler) scheduleRequests(reqs chan uint64, dist chan *request, pend chan uint64, quit chan struct{}, wg *sync.WaitGroup) {
// Clean up the goroutine and pipeline when done
Defer wg.Done()
Defer close(pend)

// Keep reading and scheduling section requests
For {
Select {
Case &lt;-quit:
Return

Case section, ok := &lt;-reqs:
// New section retrieval requested
If !ok {
Return
}
// Deduplicate retrieval requests
Unique := false

s.lock.Lock()
If s.responses[section] == nil {
S.responses[section] = &amp;response{
Done: make(chan struct{}),
}
Unique = true
}
s.lock.Unlock()

// Schedule the section for retrieval and notify the deliverer to expect this section
If unique {
Select {
Case &lt;-quit:
Return
Case dist &lt;- &amp;request{bit: s.bit, section: section}:
}
}
Select {
Case &lt;-quit:
Return
Case pend &lt;- section:
}
}
}
}



## generator.go
The generator is used to generate an object based on the section's Bloom filter index data. The main data structure inside the generator is the data structure of bloom[2048][4096]bit. The input is 4096 header.logBloom data. For example, the logBloom of the 20th header is stored in bloom[0:2048][20]

data structure:

// Generator takes a number of bloom filters and generates the rotated bloom bits
// to be used for batched filtering.
Type Generator struct {
Blooms [types.BloomBitLength][]byte // Rotated blooms for per-bit matching
Sections uint // Number of sections to batch together // The number of block headers contained in a section. The default is 4096
nextBit uint // Next bit to set when adding a bloom When adding a bloom, which bit position to set
}

structure:

// NewGenerator creates a rotated bloom generator that can iteratively fill a
// batched bloom filter's bits.
//
Func NewGenerator(sections uint) (*Generator, error) {
If sections%8 != 0 {
Return nil, errors.New("section count not multiple of 8")
}
b := &amp;Generator{sections: sections}
For i := 0; i &lt; types.BloomBitLength; i++ { //BloomBitLength=2048
B.blooms[i] = make([]byte, sections/8) // divide by 8 because a byte is 8 bits
}
Return b, nil
}

AddBloom adds a block header to the searchBloom

//AddBloom takes a single bloom filter and sets the corresponding bit column
// in memory accordingly.
Func (b *Generator) AddBloom(index uint, bloom types.Bloom) error {
// Make sure we're not adding more bloom filters than our capacity
If b.nextBit &gt;= b.sections { //Exceeded the maximum number of sections
Return errSectionOutOfBounds
}
If b.nextBit != index { //index is the subscript of the bloom in the section
Return errors.New("bloom filter with unexpected index")
}
// Rotate the bloom and insert into our collection
byteIndex := b.nextBit / 8 // Find the corresponding byte, you need to set this byte position
bitMask := byte(1) &lt;&lt; byte(7-b.nextBit%8) // Find the bit of the bit that needs to be set in the byte

For i := 0; i &lt; types.BloomBitLength; i++ {
bloomByteIndex := types.BloomByteLength - 1 - i/8
bloomBitMask := byte(1) &lt;&lt; byte(i%8)

If (bloom[bloomByteIndex] &amp; bloomBitMask) != 0 {
B.blooms[i][byteIndex] |= bitMask
}
}
b.nextBit++

Return nil
}

Bitset returns

// Bitset returns the bit vector belonging to the given bit index after all
// blooms have been added.
// After all Blooms are added, Bitset returns the data belonging to the positioning index.
Func (b *Generator) Bitset(idx uint) ([]byte, error) {
If b.nextBit != b.sections {
Return nil, errors.New("bloom not fully generated yet")
}
If idx &gt;= b.sections {
Return nil, errSectionOutOfBounds
}
Return b.blooms[idx], nil
}


## matcher.go
Matcher is a pipeline system scheduler and logic matcher that performs binary and/or operations on the bitstream, creating a stream of potential blocks to examine the data content.

data structure

// partialMatches with a non-nil vector represents a section in which some sub-
// matchers have already found potential matches. Subsequent sub-matchers will
// binary AND their matches with this vector. If vector is nil, it represents a
// section to be processed by the first sub-matcher.
// partialMatches represents the result of a partial match. There are three conditions that need to be filtered, addr1, addr2, addr3 , and you need to find data that matches these three conditions at the same time. Then we start the pipeline that contains these three conditions.
// The result of the first match is sent to the second. The second performs the bit and operation on the first result and its own result, and then sends it to the third process as a result of the match.
Type partialMatches struct {
Section uint64
Bitset []byte
}

// Retrieval represents a request for retrieval task assignments for a given
// bit with the given number of fetch elements, or a response for such a request.
// It can also have the actual results set to be used as a delivery data struct.
// Retrieval represents the retrieval of a block Bloom filter index. This object is sent to startBloomHandlers in eth/bloombits.go. This method loads the Bloom filter index from the database and returns it in Bitsets. .
Type Retrieval struct {
Bit uint
Sections []uint64
Bitsets [][]byte
}

// Matcher is a pipelined system of schedulers and logic matchers which perform
// binary AND/OR operations on the bit-streams, creating a stream of potential
// blocks to inspect for data content.
Type Matcher struct {
sectionSize uint64 // Size of the data batches to filter on

Filters [][]bloomIndexes // Filter the system is matching for
Schedulers map[uint]*scheduler // Retrieval schedulers for loading bloom bits

Retrievers chan chan uint // Retriever processes waiting for bit allocations
Counters chan chan uint // Retriever processes waiting for task count reports to return the current number of tasks
Retrieves chan chan *Retrieval // Retriever processes waiting for task allocations to pass the assignment of the retrieval task
Deliveries chan *Retrieval // Retriever processes waiting for task response deliveries The results of the search are passed to this channel

Running uint32 // Atomic flag whether a session is live or not
}


The general process picture of the matcher, the ellipse on the way represents the goroutine. The rectangle represents the channel. Triangles represent method calls.

![image](picture/matcher_1.png)

1. First, Matcher creates a corresponding number of subMatch based on the number of incoming filters. Each subMatch corresponds to a filter object. Each subMatch will get new results by bitwise matching the results of its own search and the previous search result. If all the bits of the new result are set, the result of the search will be passed to the next one. This is a short-circuit algorithm that implements the summation of the results of all filters. If the previous calculations can't match anything, then there is no need to match the following conditions.
2. Matcher will start the corresponding number of schedules according to the number of subscripts of the blender's Bloom filter.
3. subMatch will send the request to the corresponding schedule.
4. schedule will dispatch the request to the distributor through dist and manage it in the distributor.
5. Multiple (16) multiplex threads are started, requests are retrieved from the distributor, and the request is sent to the bloomRequests queue. startBloomHandlers accesses the database, gets the data and returns it to the multiplex.
6. Multiplex tells the distributor the answer via the deliveries channel.
7. The distributor calls the dispatch method of the schedule and sends the result to the schedule.
8. schedule returns the result to subMatch.
9. subMatch calculates the result and sends it to the next subMatch for processing. If it is the last subMatch, the result is processed and sent to the results channel.


Matcher

Filter := New(backend, 0, -1, []common.Address{addr}, [][]common.Hash{{hash1, hash2, hash3, hash4}})
The relationship between groups is the relationship between the group and the group. (addr &amp;&amp; hash1) ||(addr &amp;&amp; hash2)||(addr &amp;&amp; hash3)||(addr &amp;&amp; hash4)


The constructor, which requires special attention, is the input filters parameter. This parameter is a three-dimensional array [][]bloomIndexes === [first dimension][second dimension][3] .

// This is the code inside filter.go, which is useful for understanding the parameters of filters. Filter.go is the caller of the Matcher.

// You can see that no matter how many addresses are there, they only occupy one position in filters. Filters[0]=addresses
// filters[1] = topics[0] = multiple topics
// filters[2] = topics[1] = multiple topics
// filters[n] = topics[n] = multiple topics

// filter's parameter addresses and topics filter algorithm is (with any address in addresses) and (with any topic in topics[0]) and (with any topic in topics[1]) and (with topics Any of the topics in [n]

// It can be seen that for the filter is the execution and operation of the data in the first dimension, the execution or operation of the data in the second dimension.

// In the NewMatcher method, the specific data of the third dimension is converted into the specified three positions of the Bloom filter. So in the filter.go var filters [][][]byte in the Matcher filter becomes [][][3]

Func New(backend Backend, begin, end int64, addresses []common.Address, topics [][]common.Hash) *Filter {
// Flatten the address and topic filter clauses into a single bloombits filter
// system. Since the bloombits are not positional, nil topics are permitted,
// which get flattened into a nil byte slice.
Var filters [][][]byte
If len(addresses) &gt; 0 {
Filter := make([][]byte, len(addresses))
For i, address := range addresses {
Filter[i] = address.Bytes()
}
Filters = append(filters, filter)
}
For _, topicList := range topics {
Filter := make([][]byte, len(topicList))
For i, topic := range topicList {
Filter[i] = topic.Bytes()
}
Filters = append(filters, filter)
}

// NewMatcher creates a new pipeline for retrieving bloom bit streams and doing
// address and topic filtering on them. Setting a filter component to `nil` is
// allowed and will result in that filter rule being skipped (OR 0x11...1).
Func NewMatcher(sectionSize uint64, filters [][][]byte) *Matcher {
// Create the matcher instance
m := &amp;Matcher{
sectionSize: sectionSize,
Schedulers: make(map[uint]*scheduler),
Retrievers: make(chan chan uint),
Counters: make(chan chan uint),
Retrieves: make(chan chan *Retrieval),
Deliveries: make(chan *Retrieval),
}
// Calculate the bloom bit indexes for the groups we're interested in
M.filters = nil

For _, filter := range filters {
// Gather the bit indexes of the filter rule, special casing the nil filter
If len(filter) == 0 {
Continue
}
bloomBits := make([]bloomIndexes, len(filter))
For i, clause := range filter {
If clause == nil {
bloomBits = nil
Break
}
// clause corresponds to the data of the third dimension of the input, which may be an address or a topic
// calcBloomIndexes calculates the three subscripts in the Bloom filter corresponding to this data (0-2048), that is, if the corresponding three bits in the Bloom filter are both 1, then the data may be clause it's here.
bloomBits[i] = calcBloomIndexes(clause)
}
// Accumulate the filter rules if no nil rule was within
// In the calculation, if only one of the bloomBits can be found. Then think that the whole is established.
If bloomBits != nil {
// Different bloomBits need to be established at the same time, the whole result can be established.
M.filters = append(m.filters, bloomBits)
}
}
// For every bit, create a scheduler to load/download the bit vectors
For _, bloomIndexLists := range m.filters {
For _, bloomIndexList := range bloomIndexLists {
For _, bloomIndex := range bloomIndexList {
// For all possible subscripts. We all generate a scheduler to perform the corresponding position.
// Bronze filter data retrieval.
m.addScheduler(bloomIndex)
}
}
}
Return m
}



Start start

// Start starts the matching process and returns a stream of bloom matches in
// a given range of blocks. If there are no more matches in the range, the result
// channel is closed.
Func (m *Matcher) Start(begin, end uint64, results chan uint64) (*MatcherSession, error) {
// Make sure we're not creating concurrent sessions
If atomic.SwapUint32(&amp;m.running, 1) == 1 {
Return nil, errors.New("matcher already running")
}
Defer atomic.StoreUint32(&amp;m.running, 0)

// Initiate a new matching round
// Start a session, as a return value, manage the lifecycle of the lookup.
Session := &amp;MatcherSession{
Matcher: m,
Quit: make(chan struct{}),
Kill: make(chan struct{}),
}
For _, scheduler := range m.schedulers {
Scheduler.reset()
}
// This run will build the process and return a partialMatches type pipeline representing some of the results of the query.
Sink := m.run(begin, end, cap(results), session)

// Read the output from the result sink and deliver to the user
session.pend.Add(1)
Go func() {
Defer session.pend.Done()
Defer close(results)

For {
Select {
Case &lt;-session.quit:
Return

Case res, ok := &lt;-sink:
// New match result found
// Find the return result because the return value is a bitmap of which sections in the section and section may have values
// So you need to iterate through the bitmap, find the blocks that are set, and return the block number back.
If !ok {
Return
}
// Calculate the first and last blocks of the section
sectionStart := res.section * m.sectionSize

First := sectionStart
If begin &gt; first {
First = begin
}
Last := sectionStart + m.sectionSize - 1
If end &lt; last {
Last = end
}
// Iterate over all the blocks in the section and return the matching ones
For i := first; i &lt;= last; i++ {
// Skip the entire byte if no matches are found inside
Next := res.bitset[(i-sectionStart)/8]
If next == 0 {
i += 7
Continue
}
// Some bit it set, do the actual submatching
If bit := 7 - i%8; next&amp;(1&lt;&lt;bit) != 0 {
Select {
Case &lt;-session.quit:
Return
Case results &lt;- i:
}
}
}
}
}
}()
Return session, nil
}


Run method

// run creates a daisy-chain of sub-matchers, one for the address set and one
// for each topic set, each sub-matcher receiving a section only if the previous
// ones have all found a potential match in one of the blocks of the section,
// then binary AND-ing its own matches and forwaring the result to the next one.
// Create a pipeline of child matchers, one for the address set and one for each topic set, each child matcher receiving only if all previous child blocks find a possible match in one of the blocks in that part A part, then match the received one with yourself and pass the result to the next one.
// The method starts feeding the section indexes into the first sub-matcher on a
// new goroutine and returns a sink channel receiving the results.

The method starts the section indexer and sends it to the first sub-matcher and returns the receiver channel that receives the result.
Func (m *Matcher) run(begin, end uint64, buffer int, session *MatcherSession) chan *partialMatches {
// Create the source channel and feed section indexes into
Source := make(chan *partialMatches, buffer)

session.pend.Add(1)
Go func() {
Defer session.pend.Done()
Defer close(source)

For i := begin / m.sectionSize; i &lt;= end/m.sectionSize; i++ {
// This for loop constructs the first input source of subMatch, and the remaining subMatch takes the previous result as its own source.
// The bitset field of this source is 0xff, which represents a complete match. It will be compared with the match of our step to get the result of this step match.
Select {
Case &lt;-session.quit:
Return
Case source &lt;- &amp;partialMatches{i, bytes.Repeat([]byte{0xff}, int(m.sectionSize/8))}:
}
}
}()
// Assemble the daisy-chained filtering pipeline
Next := source
Dist := make(chan *request, buffer)

For _, bloom := range m.filters { //Build the pipeline, the previous output as the input to the next subMatch.
Next = m.subMatch(next, dist, bloom, session)
}
// Start the request distribution
session.pend.Add(1)
// Start the distributor thread.
Go m.distributor(dist, session)

Return next
}


subMatch function


// subMatch creates a sub-matcher that filters for a set of addresses or topics, binary OR-s those matches, then
// binary AND-s the result to the daisy-chain input (source) and forwards it to the daisy-chain output.
// The matches of each address/topic are calculated by fetching the given sections of the three bloom bit indexes belonging to
// that address/topic, and binary AND-ing those vectors together.
// subMatch creates a sub-matcher that filters a set of addresses or topics, performs bitwise operations on these topics, and then performs bitwise operations on the previous result with the current filtered result. If the result is not empty, the result is Pass to the next child matcher. The matching of each address/topic is calculated by taking a given portion of the three Bloom filter bit indices belonging to the address/topic and combining these vectors in binary AND.

SubMatch is the most important function that combines the first dimension of the filters [][][3] with the second dimension or the third dimension.


Func (m *Matcher) subMatch(source chan *partialMatches, dist chan *request, bloom []bloomIndexes, session *MatcherSession) chan *partialMatches {
// Start the concurrent schedulers for each bit required by the bloom filter
// The incoming bloom []bloomIndexes parameter is the second, third dimension of filters [][3]


sectionSources := make([][3]chan uint64, len(bloom))
sectionSinks := make([][3]chan []byte, len(bloom))
For i, bits := range bloom { // i represents the number of second dimensions
For j, bit := range bits { //j represents the subscript of the Bloom filter. There are definitely only three values ​​(0-2048).
sectionSources[i][j] = make(chan uint64, cap(source)) // Create the input channel for the scheduler
sectionSinks[i][j] = make(chan []byte, cap(source)) // Create the output channel of the scheduler
// Initiate a scheduling request for this bit, passing the section that needs to be queried via sectionSources[i][j]
// Receive results via sectionSinks[i][j]
// dist is the channel through which the scheduler passes the request. This is in the introduction of the scheduler.
M.schedulers[bit].run(sectionSources[i][j], dist, sectionSinks[i][j], session.quit, &amp;session.pend)
}
}

Process := make(chan *partialMatches, cap(source)) // entries from source are forwarded here after fetches have been initiated
Results := make(chan *partialMatches, cap(source)) // return value channel

session.pend.Add(2)
Go func() {
// Tear down the goroutine and terminate all source channels
Defer session.pend.Done()
Defer close(process)

Defer func() {
For _, bloomSources := range sectionSources {
For _, bitSource := range bloomSources {
Close(bitSource)
}
}
}()
// Read sections from the source channel and multiplex into all bit-schedulers
// Read the sections from the source channel and pass the data to the scheduler via sectionSources
For {
Select {
Case &lt;-session.quit:
Return

Case subres, ok := &lt;-source:
// New subresult from previous link
If !ok {
Return
}
// Multiplex the section index to all bit-schedulers
For _, bloomSources := range sectionSources {
For _, bitSource := range bloomSources {
// Pass to the input channel of all the schedulers above. Apply for these
// The specified bit of the section is searched. The result will be sent to sectionSinks[i][j]
Select {
Case &lt;-session.quit:
Return
Case bitSource &lt;- subres.section:
}
}
}
// Notify the processor that this section will become available
Select {
Case &lt;-session.quit:
Return
Case process &lt;- subres: // Wait until all requests are submitted to the scheduler to send a message to the process.
}
}
}
}()

Go func() {
// Tear down the goroutine and terminate the final sink channel
Defer session.pend.Done()
Defer close(results)

// Read the source notifications and collect the delivered results
For {
Select {
Case &lt;-session.quit:
Return

Case subres, ok := &lt;-process:
// There is a problem here. Is it possible to order out. Because the channels are all cached. May be queried quickly
// View the implementation of the scheduler, the scheduler is guaranteed to be in order. How come in, how will you go out.
// Notified of a section being retrieved
If !ok {
Return
}
// Gather all the sub-results and merge them together
Var orVector []byte
For _, bloomSinks := range sectionSinks {
Var andVector []byte
For _, bitSink := range bloomSinks { // Here we can receive three values. Each represents the value of the Bloom filter corresponding to the subscript, and the three values ​​are operated and operated.
It is possible to get those blocks that may have corresponding values.
Var data []byte
Select {
Case &lt;-session.quit:
Return
Case data = &lt;-bitSink:
}
If andVector == nil {
andVector = make([]byte, int(m.sectionSize/8))
Copy(andVector, data)
} else {
bitutil.ANDBytes(andVector, andVector, data)
}
}
If orVector == nil { Performs an Or operation on the data of the first dimension.
orVector = andVector
} else {
bitutil.ORBytes(orVector, orVector, andVector)
}
}

If orVector == nil { //The channel may be closed. Did not query any value
orVector = make([]byte, int(m.sectionSize/8))
}
If subres.bitset != nil {
// Perform the AND operation with the last result entered. Remember that this value was initialized to all 1 at the beginning.
bitutil.ANDBytes(orVector, orVector, subres.bitset)
}
If bitutil.TestBytes(orVector) { // If not all 0 then add to the result. May give the next match. Or return.
Select {
Case &lt;-session.quit:
Return
Case results &lt;- &amp;partialMatches{subres.section, orVector}:
}
}
}
}
}()
Return results
}

Distributor, accept requests from the scheduler and put them in a set. Then assign these tasks to retrievers to populate them.

// distributor receives requests from the schedulers and queues them into a set
// of pending requests, which are assigned to retrievers wanting to fulfil them.
Func (m *Matcher) distributor(dist chan *request, session *MatcherSession) {
Defer session.pend.Done()

Var (
Requests = make(map[uint][]uint64) // Per-bit list of section requests, ordered by section number
Unallocs = make(map[uint]struct{}) // Bits with pending requests but not allocated to any retriever
Retrievers chan chan uint // Waiting retrievers (toggled to nil if unallocs is empty)
)
Var (
Allocs int // Number of active allocations to handle graceful shutdown requests
Shutdown = session.quit // Shutdown request channel, will gracefully wait for pending requests
)

// assign is a helper method fo try to assign a pending bit an an active
// listening servicer, or schedule it up for later when one arrives.
Assign := func(bit uint) {
Select {
Case fetcher := &lt;-m.retrievers:
Allocs++
Fetcher &lt;- bit
Default:
// No retrievers active, start listening for new ones
Retrievers = m.retrievers
Unallocs[bit] = struct{}{}
}
}

For {
Select {
Case &lt;-shutdown:
// Graceful shutdown requested, wait until all pending requests are honoured
If allocs == 0 {
Return
}
Shutdown = nil

Case &lt;-session.kill:
// Pending requests not honoured in time, hard terminate
Return

Case req := &lt;-dist: // The request sent by the scheduler is added to the queue of the specified bit position.
// New retrieval request arrived to be distributed to some fetcher process
Queue := requests[req.bit]
Index := sort.Search(len(queue), func(i int) bool { return queue[i] &gt;= req.section })
Requests[req.bit] = append(queue[:index], append([]uint64{req.section}, queue[index:]...)...)

// If it's a new bit and we have waiting fetchers, allocate to them
// If this bit is a new one. Not yet assigned, then we assign him to the waiting fetchers
If len(queue) == 0 {
Assign(req.bit)
}

Case fetcher := &lt;-retrievers:
// New retriever arrived, find the lowest section-ed bit to assign
// If the new retrievers come in, then we check to see if any tasks are not assigned
Bit, best := uint(0), uint64(math.MaxUint64)
For idx := range unallocs {
If requests[idx][0] &lt; best {
Bit, best = idx, requests[idx][0]
}
}
// Stop tracking this bit (and alloc notifications if no more work is available)
Delete(unallocs, bit)
If len(unallocs) == 0 { //If all tasks are assigned. Then stop paying attention to retrievers
Retrievers = nil
}
Allocs++
Fetcher &lt;- bit

Case fetcher := &lt;-m.counters:
// New task count request arrives, return number of items
// A new request was made to access the number of specified bits of the request.
Fetcher &lt;- uint(len(requests[&lt;-fetcher]))

Case fetcher := &lt;-m.retrievals:
// New fetcher waiting for tasks to retrieve, assign
// Someone came to pick up the task.
Task := &lt;-fetcher
If want := len(task.Sections); want &gt;= len(requests[task.Bit]) {
task.Sections = requests[task.Bit]
Delete(requests, task.Bit)
} else {
task.Sections = append(task.Sections[:0], requests[task.Bit][:want]...)
Requests[task.Bit] = append(requests[task.Bit][:0], requests[task.Bit][want:]...)
}
Fetcher &lt;- task

// If anything was left unallocated, try to assign to someone else
// If there are still tasks not assigned. Try to assign it to others.
If len(requests[task.Bit]) &gt; 0 {
Assign(task.Bit)
}

Case result := &lt;-m.deliveries:
// New retrieval task response from fetcher, split out missing sections and
// deliver complete ones
// Received the result of the task.
Var (
Sections = make([]uint64, 0, len(result.Sections))
Bitsets = make([][]byte, 0, len(result.Bitsets))
Missing = make([]uint64, 0, len(result.Sections))
)
For i, bitset := range result.Bitsets {
If len(bitset) == 0 { //If the task result is missing, record it
Missing = append(missing, result.Sections[i])
Continue
}
Sections = append(sections, result.Sections[i])
Bitsets = append(bitsets, bitset)
}
// delivery result
M.schedulers[result.Bit].deliver(sections, bitsets)
Allocs--

				// Reschedule missing sections and allocate bit if newly available
				if len(missing) &gt; 0 { //如果有缺失， 那么重新生成新的任务。
					queue := requests[result.Bit]
					for _, section := range missing {
						index := sort.Search(len(queue), func(i int) bool { return queue[i] &gt;= section })
						queue = append(queue[:index], append([]uint64{section}, queue[index:]...)...)
					}
					requests[result.Bit] = queue

					if len(queue) == len(missing) {
						assign(result.Bit)
					}
}
				// If we're in the process of shutting down, terminate
				if allocs == 0 &amp;&amp; shutdown == nil {
					return
}
}
}
}


任务领取AllocateRetrieval。 任务领取了一个任务。 会返回指定的bit的检索任务。

	// AllocateRetrieval assigns a bloom bit index to a client process that can either
	// immediately reuest and fetch the section contents assigned to this bit or wait
	// a little while for more sections to be requested.
	func (s *MatcherSession) AllocateRetrieval() (uint, bool) {
		fetcher := make(chan uint)

		select {
		case &lt;-s.quit:
			return 0, false
		case s.matcher.retrievers &lt;- fetcher:
			bit, ok := &lt;-fetcher
			return bit, ok
}
}

AllocateSections,领取指定bit的section查询任务。

	// AllocateSections assigns all or part of an already allocated bit-task queue
	// to the requesting process.
	func (s *MatcherSession) AllocateSections(bit uint, count int) []uint64 {
		fetcher := make(chan *Retrieval)

		select {
		case &lt;-s.quit:
			return nil
		case s.matcher.retrievals &lt;- fetcher:
			task := &amp;Retrieval{
				Bit:      bit,
				Sections: make([]uint64, count),
}
			fetcher &lt;- task
			return (&lt;-fetcher).Sections
}
}

DeliverSections，把结果投递给deliveries 通道。

	// DeliverSections delivers a batch of section bit-vectors for a specific bloom
	// bit index to be injected into the processing pipeline.
	func (s *MatcherSession) DeliverSections(bit uint, sections []uint64, bitsets [][]byte) {
		select {
		case &lt;-s.kill:
			return
		case s.matcher.deliveries &lt;- &amp;Retrieval{Bit: bit, Sections: sections, Bitsets: bitsets}:
}
}

任务的执行Multiplex,Multiplex函数不断的领取任务，把任务投递给bloomRequest队列。从队列获取结果。然后投递给distributor。 完成了整个过程。

	// Multiplex polls the matcher session for rerieval tasks and multiplexes it into
	// the reuested retrieval queue to be serviced together with other sessions.
	//
	// This method will block for the lifetime of the session. Even after termination
	// of the session, any request in-flight need to be responded to! Empty responses
	// are fine though in that case.
	func (s *MatcherSession) Multiplex(batch int, wait time.Duration, mux chan chan *Retrieval) {
For {
			// Allocate a new bloom bit index to retrieve data for, stopping when done
			bit, ok := s.AllocateRetrieval()
			if !ok {
Return
}
			// Bit allocated, throttle a bit if we're below our batch limit
			if s.PendingSections(bit) &lt; batch {
				select {
				case &lt;-s.quit:
					// Session terminating, we can't meaningfully service, abort
					s.AllocateSections(bit, 0)
					s.DeliverSections(bit, []uint64{}, [][]byte{})
					return

				case &lt;-time.After(wait):
					// Throttling up, fetch whatever's available
}
}
			// Allocate as much as we can handle and request servicing
			sections := s.AllocateSections(bit, batch)
			request := make(chan *Retrieval)

Select {
			case &lt;-s.quit:
				// Session terminating, we can't meaningfully service, abort
				s.DeliverSections(bit, sections, make([][]byte, len(sections)))
Return

			case mux &lt;- request:
				// Retrieval accepted, something must arrive before we're aborting
				request &lt;- &amp;Retrieval{Bit: bit, Sections: sections}

				result := &lt;-request
				s.DeliverSections(result.Bit, result.Sections, result.Bitsets)
}
}
}

</pre></body></html>