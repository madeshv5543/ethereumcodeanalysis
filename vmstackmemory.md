
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>Vm uses the Stack of objects in stack.go as the stack of the virtual machine. Memory represents the memory object used in the virtual machine.

## stack
The simpler is to use 1024 big.Int fixed-length arrays as the storage of the stack.

structure

// stack is an object for basic stack operations. Items popped to the stack are
// expected to be changed and modified. stack does not take care of adding newly
// initialised objects.
Type Stack struct {
Data []*big.Int
}

Func newstack() *Stack {
Return &amp;Stack{data: make([]*big.Int, 0, 1024)}
}

Push operation

Func (st *Stack) push(d *big.Int) { // appended to the end
// NOTE push limit (1024) is checked in baseCheck
//stackItem := new(big.Int).Set(d)
//st.data = append(st.data, stackItem)
St.data = append(st.data, d)
}
Func (st *Stack) pushN(ds ...*big.Int) {
St.data = append(st.data, ds...)
}

Pop operation


Func (st *Stack) pop() (ret *big.Int) { //Remove from the end.
Ret = st.data[len(st.data)-1]
St.data = st.data[:len(st.data)-1]
Return
}
The value operation of the exchange element, and the operation?

Func (st *Stack) swap(n int) { The value of the element at the top of the swap stack and the element at the top of the stack.
St.data[st.len()-n], st.data[st.len()-1] = st.data[st.len()-1], st.data[st.len()-n ]
}

Dup operation like copying the value of the specified position to the top of the heap

Func (st *Stack) dup(pool *intPool, n int) {
St.push(pool.get().Set(st.data[st.len()-n]))
}

Peek operation. Peeking at the top of the stack

Func (st *Stack) peek() *big.Int {
Return st.data[st.len()-1]
}
Back peek at the elements of the specified location

// Back returns the n'th item in stack
Func (st *Stack) Back(n int) *big.Int {
Return st.data[st.len()-n-1]
}

Require guarantees that the number of stack elements is greater than or equal to n.

Func (st *Stack) require(n int) error {
If st.len() &lt; n {
Return fmt.Errorf("stack underflow (%d &lt;=&gt; %d)", len(st.data), n)
}
Return nil
}

## intpool
Very simple. It is a 256-sized pool of big.int to speed up the allocation of bit.Int.

Var checkVal = big.NewInt(-42)

Const poolLimit = 256

// intPool is a pool of big integers that
// can be reused for all big.Int operations.
Type intPool struct {
Pool *Stack
}

Func newIntPool() *intPool {
Return &amp;intPool{pool: newstack()}
}

Func (p *intPool) get() *big.Int {
If p.pool.len() &gt; 0 {
Return p.pool.pop()
}
Return new(big.Int)
}
Func (p *intPool) put(is ...*big.Int) {
If len(p.pool.data) &gt; poolLimit {
Return
}

For _, i := range is {
// verifyPool is a build flag. Pool verification makes sure the integrity
// of the integer pool by comparing values ​​to a default value.
If verifyPool {
i.Set(checkVal)
}

P.pool.push(i)
}
}

## memory

Construction, memory storage is byte[]. There is also a record of lastGasCost.

Type Memory struct {
Store []byte
lastGasCost uint64
}

Func NewMemory() *Memory {
Return &amp;Memory{}
}

Use first need to use Resize to allocate space

// Resize resizes the memory to size
Func (m *Memory) Resize(size uint64) {
If uint64(m.Len()) &lt; size {
M.store = append(m.store, make([]byte, size-uint64(m.Len()))...)
}
}

Then use Set to set the value

// Set sets offset + size to value
Func (m *Memory) Set(offset, size uint64, value []byte) {
// length of store may never be less than offset + size.
// The store should be resized PRIOR to setting the memory
If size &gt; uint64(len(m.store)) {
Panic("INVALID memory: store empty")
}

// It's possible the offset is greater than 0 and size equals 0. This is because
// the calcMemSize (common.go) could potentially return 0 when size is zero (NO-OP)
If size &gt; 0 {
Copy(m.store[offset:offset+size], value)
}
}
Get to get the value, one is to get the copy, one is to get the pointer.

// Get returns offset + size as a new slice
Func (self *Memory) Get(offset, size int64) (cpy []byte) {
If size == 0 {
Return nil
}

If len(self.store) &gt; int(offset) {
Cpy = make([]byte, size)
Copy(cpy, self.store[offset:offset+size])

Return
}

Return
}

// GetPtr returns the offset + size
Func (self *Memory) GetPtr(offset, size int64) []byte {
If size == 0 {
Return nil
}

If len(self.store) &gt; int(offset) {
Return self.store[offset : offset+size]
}

Return nil
}


## Some extra helper functions in stack_table.go


Func makeStackFunc(pop, push int) stackValidationFunc {
Return func(stack *Stack) error {
If err := stack.require(pop); err != nil {
Return err
}

If stack.len()+push-pop &gt; int(params.StackLimit) {
Return fmt.Errorf("stack limit reached %d (%d)", stack.len(), params.StackLimit)
}
Return nil
}
}

Func makeDupStackFunc(n int) stackValidationFunc {
Return makeStackFunc(n, n+1)
}

Func makeSwapStackFunc(n int) stackValidationFunc {
Return makeStackFunc(n, n)
}


</pre></body></html>