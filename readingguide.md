
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>It is recommended to first understand the Appendix B. Recursive Length Prefix in the Yellow Book.

Auxiliary reading can refer to: [https://segmentfault.com/a/1190000011763339] (https://segmentfault.com/a/1190000011763339)
Or directly search for rlp related content

From an engineering point of view, rlp is divided into two types of data definitions, and one of them can recursively contain another type:

```
T ≡ L∪B // T consists of L or B
L ≡ {t:t=(t[0],t[1],...) ∧ \forall n&lt;‖t‖ t[n]∈T} // Any member of L belongs to T (T again Is composed of L or B: note recursive definition)
B ≡ {b:b=(b[0],b[1],...) ∧ \forall n&lt;‖b‖ b[n]∈O} // Any member of T belongs to O
```

one of them

* [\forall](https://en.wikibooks.org/wiki/LaTeX/Mathematics#Symbols) Reference LaTeX Grammar Standard
* O is defined as a bytes collection
* If you think of T as a tree-like data structure, then B is the leaf, which only contains the byte sequence structure; and L is the trunk, containing multiple B or itself

That is, we rely on the basis of B to form a recursive definition of T and L: the whole RLP consists of T, T contains L and B, and the members of L are both T. Such recursive definitions can describe very flexible data structures

In the specific encoding, only the coding space of the first byte can be used to distinguish these structural differences:

```
B coding rules: leaves
RLP_B0 [0000 0001, 0111 1111] If the byte is not 0 and less than 128[1000 0000], no header is needed, and the content is encoded.
RLP_B1 [1000 0000, 1011 0111] If the length of the byte content is less than 56, that is, 55[0011 0111], compress its length into a byte in big endian order, add 128 to form the header, and then connect the actual Content
RLP_B2 (1011 0111, 1100 0000) For longer contents, describe the length of the content length in the space where the 2nd bit is not 1. The space is (192-1)[1011 1111]-(183+1)[1011 1000]=7[0111], that is, the length needs to be less than 2^(7*8) is a huge to impossible number
L coding rules: branches
RLP_L1 [1100 0000, 1111 0111) If it is a combination of multiple above encoding contents, it is expressed by 1 in the second bit. Subsequent content length is less than 56, that is, 55[0011 0111], first compress the length and put it into the first byte (add 192[1100 0000]), then connect the actual content.
RLP_L2 [1111 0111, 1111 1111] For longer content, the length of the content length is described in the remaining space. The space is 255[1111 1111]-247[1111 0111]=8[1000], that is, the length needs to be less than 2^(8*8). It is also a huge to impossible number.
```

Please copy the following code into the editor with text folding for code review (Recommended Notepad++ will properly collapse all functions under the github reference format for easy global understanding)

```code
[/rlp/encode_test.go#TestEncode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode_test.go#L272)
Func TestEncode(t *testing.T) {
runEncTests(t, func(val interface{}) ([]byte, error) {
b := new(bytes.Buffer)
Err := Encode(b, val)
[/rlp/encode.go#Encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L80)
Func Encode(w io.Writer, val interface{}) error {
If outer, ok := w.(*encbuf); ok {
// Encode was called by some type's EncodeRLP.
// Avoid copying by writing to the outer encbuf directly.
Return outer.encode(val)
}
Eb := encbufPool.Get().(*encbuf)
[/rlp/encode.go#encbuf](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L121)
Type encbuf struct { // stateful encoder
Str []byte // string data, contains everything except list headers // encoded content, does not contain L headers
Lheads []*listhead // all list headers // L header information array for the current recursive hierarchy

[/rlp/encode.go#listhead](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L128)
Type listhead struct {
Offset int // index of this header in string data // TODO
Size int // total size of encoded data (including list headers) // TODO
}

Lhsize int // sum of sizes of all encoded list headers // length of L header information at the current recursive level TODO
Sizebuf []byte // 9-byte auxiliary buffer for uint encoding // buf for size encoding where buf[0] is the header and the remaining 8 bytes for size encoding // TODO
}

Defer encbufPool.Put(eb)
Eb.reset()
If err := eb.encode(val); err != nil { // encbuf.encode as a B-encoded (internal) function, eb is a stateful encbuf
[/rlp/encode.go#encbuf.encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L181) The function is implemented as
Func (w *encbuf) encode(val interface{}) error {
Rval := reflect.ValueOf(val)
Ti, err := cachedTypeInfo(rval.Type(), tags{}) // Get the current type of content encoding function from the cache
If err != nil {
Return err
}
Return ti.writer(rval, w) // execute function
[/rlp/encode.go#writeUint](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L392)
Func writeUint(val reflect.Value, w *encbuf) error {
i := val.Uint()
If i == 0 {
W.str = append(w.str, 0x80) // Prevent all 0 exceptions in the sense of encoding, encoding 0 as 0x80
} else if i &lt; 128 { // Implement RLP_B0 encoding logic
// fits single byte
W.str = append(w.str, byte(i))
} else { // Implement RLP_B1 encoding logic: because uint has a byte length of 8 and will not exceed 56
// TODO: encode int to w.str directly
s := putint(w.sizebuf[1:], i) // remove the bit with uint high zero, remove by byte granularity, and return the removed byte number
W.sizebuf[0] = 0x80 + byte(s) // For uint, the maximum length is 64 bit/8 byte, so compressing the length to byte[0,256) is quite enough.
W.str = append(w.str, w.sizebuf[:s+1]...) // Output all available bytes in sizebuf as encoded content
}
Return nil
}
[/rlp/encode.go#writeUint](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L461)
Func writeString(val reflect.Value, w *encbuf) error {
s := val.String()
If len(s) == 1 &amp;&amp; s[0] &lt;= 0x7f { // 0x7f=127 Implement RLP_B0 encoding logic, note that the empty string will go below else
// fits single byte, no string header
W.str = append(w.str, s[0])
} else {
w.encodeStringHeader(len(s)) // Implement RLP_B1, RLP_B2 encoding logic
[/rlp/encode.go#encbuf.encodeStringHeader](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L191)
Func (w *encbuf) encodeStringHeader(size int) {
If size &lt; 56 { // implement RLP_B1 encoding logic
W.str = append(w.str, 0x80+byte(size))
} else { // implement RLP_B2 encoding logic
// TODO: encode to w.str directly
Sizesize := putint(w.sizebuf[1:], uint64(size)) // Divide size in big endian order, remove the high order [0000 0000] by byte granularity and range byte number
W.sizebuf[0] = 0xB7 + byte(sizesize) // 0xB7[1011 0111]183 Encode the length of the header into the first byte
W.str = append(w.str, w.sizebuf[:sizesize+1]...) // Completion of the length information encoding of the header
}
}
W.str = append(w.str, s...) // append the actual content to the header
}
Return nil
}
[/rlp/encode.go#makeStructWriter](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L529)
Func makeStructWriter(typ reflect.Type) (writer, error) {
Fields, err := structFields(typ) // Get field information by reflection to encode one by one
If err != nil {
Return nil, err
}
Writer := func(val reflect.Value, w *encbuf) error {
Lh := w.list() // Create a listhead storage object
[/rlp/encode.go#encbuf.list](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L212)
Func (w *encbuf) list() *listhead {
Lh := &amp;listhead{offset: len(w.str), size: w.lhsize} // Create a new listhead object with the current total length of the encoding as offset and the current total header length lhsize as size
W.lheads = append(w.lheads, lh)
Return lh // Add the header sequence and return it to listEnd
}
For _, f := range fields {
If err := f.info.writer(val.Field(f.index), w); err != nil {
Return err
}
}
w.listEnd(lh) // set the value of the listhead object
[/rlp/encode.go#encbuf.listEnd](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L218)
Func (w *encbuf) listEnd(lh *listhead) {
Lh.size = w.size() - lh.offset - lh.size // Is the new header size equal to the newly added code length minus? TODO
If lh.size &lt; 56 {
W.lhsize += 1 // length encoded into kind tag
} else {
W.lhsize += 1 + intsize(uint64(lh.size))
}
}
Return nil
}
Return writer, nil
}
}
Return err
}
Return eb.toWriter(w) // encbuf.toWriter as L header encoding (internal) function
[/rlp/encode.go#encbuf.toWriter](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L249)
Func (w *encbuf) toWriter(out io.Writer) (err error) {
Strpos := 0
For _, head := range w.lheads { // For the case of no lheads data, ignore the following header encoding logic
// write string data before header
If head.offset-strpos &gt; 0 {
n, err := out.Write(w.str[strpos:head.offset])
Strpos += n
If err != nil {
Return err
}
}
// write the header
Enc := head.encode(w.sizebuf)
[/rlp/encode.go#encbuf.listhead.encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L135)
Func (head *listhead) encode(buf []byte) []byte {
// Convert binary
// 0xC0 192: 1100 0000
// 0xF7 247: 1111 0111
Return buf[:puthead(buf, 0xC0, 0xF7, uint64(head.size))]
[/rlp/encode.go#puthead](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L150)
Func puthead(buf []byte, smalltag, largetag byte, size uint64) int {
If size &lt; 56 {
Buf[0] = smalltag + byte(size)
Return 1
} else {
Sizesize := putint(buf[1:], size)
Buf[0] = largetag + byte(sizesize)
Return sizesize + 1
}
}
}
If _, err = out.Write(enc); err != nil {
Return err
}
}
If strpos &lt; len(w.str) { // strpos is 0. It must be true, directly with the contents of the header encoding as the final output.
// write string data after the last list header
_, err = out.Write(w.str[strpos:])
}
Return err
}
}
Return b.Bytes(), err
})
}
```



#测试文件Format Appendix

After understanding the general meaning of RLP, it is recommended to start reading from the test code.
[/rlp/encode_test.go](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode_test.go#L272)
Code segment in

Func TestEncode(t *testing.T) {
runEncTests(t, func(val interface{}) ([]byte, error) {
b := new(bytes.Buffer)
Err := Encode(b, val)
Return b.Bytes(), err
})
}

Err := Encode(b, val) The Encode called is used as the entry function of the encoding.
[/rlp/encode.go#Encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L80)

Func Encode(w io.Writer, val interface{}) error {
If outer, ok := w.(*encbuf); ok {
// Encode was called by some type's EncodeRLP.
// Avoid copying by writing to the outer encbuf directly.
Return outer.encode(val)
}
Eb := encbufPool.Get().(*encbuf)
Defer encbufPool.Put(eb)
Eb.reset()
If err := eb.encode(val); err != nil { // encbuf.encode as content encoding (internal) function
Return err
}
Return eb.toWriter(w) // encbuf.toWriter as header encoding (internal) function
}

[/rlp/encode.go#encbuf.encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L181) The function is implemented as

Func (w *encbuf) encode(val interface{}) error {
Rval := reflect.ValueOf(val)
Ti, err := cachedTypeInfo(rval.Type(), tags{}) // Get the current type of content encoding function from the cache
If err != nil {
Return err
}
Return ti.writer(rval, w) // execute function
}

We ignore the cache type and the encoding function's get and generate function cachedTypeInfo , and move the focus directly to the specific type of content encoding function.

The normal uint encoding implementation is at [/rlp/encode.go#writeUint](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L392)

Func writeUint(val reflect.Value, w *encbuf) error {
i := val.Uint()
If i == 0 {
W.str = append(w.str, 0x80) // Prevent all 0 exceptions in the sense of encoding, encoding 0 as 0x80
} else if i &lt; 128 {
// fits single byte
W.str = append(w.str, byte(i))
} else {
// TODO: encode int to w.str directly
s := putint(w.sizebuf[1:], i) // remove the bit with uint high zero, remove by byte granularity, and return the removed byte number
W.sizebuf[0] = 0x80 + byte(s) // For uint, the maximum length is 64 bit/8 byte, so compressing the length to byte[0,256) is quite enough.
W.str = append(w.str, w.sizebuf[:s+1]...) // Output all available bytes in sizebuf as encoded content
}
Return nil
}

Then follow-up logic in [/rlp/encode.go#Encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L80)[/rlp/encode.go #encbuf.toWriter](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L249)

Func (w *encbuf) toWriter(out io.Writer) (err error) {
Strpos := 0
For _, head := range w.lheads { // For the case of no lheads data, ignore the following header encoding logic
// write string data before header
If head.offset-strpos &gt; 0 {
n, err := out.Write(w.str[strpos:head.offset])
Strpos += n
If err != nil {
Return err
}
}
// write the header
Enc := head.encode(w.sizebuf)
If _, err = out.Write(enc); err != nil {
Return err
}
}
If strpos &lt; len(w.str) { // strpos is 0. It must be true, directly with the contents of the header encoding as the final output.
// write string data after the last list header
_, err = out.Write(w.str[strpos:])
}
Return err
}

For string encoding [/rlp/encode.go#writeString](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L461) Similar to uint, without header encoding information. Can refer to the yellow book, I will not repeat

Func writeString(val reflect.Value, w *encbuf) error {
s := val.String()
If len(s) == 1 &amp;&amp; s[0] &lt;= 0x7f {
// fits single byte, no string header
W.str = append(w.str, s[0])
} else {
w.encodeStringHeader(len(s))
W.str = append(w.str, s...)
}
Return nil
}

[/rlp/encode.go#makeStructWriter](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L529) The function contains more complicated

Func makeStructWriter(typ reflect.Type) (writer, error) {
Fields, err := structFields(typ)
If err != nil {
Return nil, err
}
Writer := func(val reflect.Value, w *encbuf) error {
Lh := w.list() // Create a listhead storage object
For _, f := range fields {
If err := f.info.writer(val.Field(f.index), w); err != nil {
Return err
}
}
w.listEnd(lh) // set the value of the listhead object
Return nil
}
Return writer, nil
}

[/rlp/encode.go#encbuf.list/listEnd](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L212)

Func (w *encbuf) list() *listhead {
Lh := &amp;listhead{offset: len(w.str), size: w.lhsize} // Create a new listhead object with the current total length of the encoding as offset and the current total header length lhsize as size
W.lheads = append(w.lheads, lh)
Return lh // Add the header sequence and return it to listEnd
}

Func (w *encbuf) listEnd(lh *listhead) {
Lh.size = w.size() - lh.offset - lh.size // Is the new header size equal to the newly added code length minus? TODO
If lh.size &lt; 56 {
W.lhsize += 1 // length encoded into kind tag
} else {
W.lhsize += 1 + intsize(uint64(lh.size))
}
}

[encbuf.toWriter](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L248) The function is implemented as

Func (w *encbuf) toWriter(out io.Writer) (err error) {
Strpos := 0
For _, head := range w.lheads {
// write string data before header
If head.offset-strpos &gt; 0 {
n, err := out.Write(w.str[strpos:head.offset])
Strpos += n
If err != nil {
Return err
}
}
// write the header
Enc := head.encode(w.sizebuf)
If _, err = out.Write(enc); err != nil {
Return err
}
}
If strpos &lt; len(w.str) {
// write string data after the last list header
_, err = out.Write(w.str[strpos:])
}
Return err
}

https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L135

Func (head *listhead) encode(buf []byte) []byte {
// Convert binary
// 0xC0 192: 1100 0000
// 0xF7 247: 1111 0111
Return buf[:puthead(buf, 0xC0, 0xF7, uint64(head.size))]
}

https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L150

Func puthead(buf []byte, smalltag, largetag byte, size uint64) int {
If size &lt; 56 {
Buf[0] = smalltag + byte(size)
Return 1
} else {
Sizesize := putint(buf[1:], size)
Buf[0] = largetag + byte(sizesize)
Return sizesize + 1
}
}

...



Goroutine 1 [running]:
Main.f(0x0)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;D:/coding/ztesoft/golang/src/defer2.go:30 +0x1b8
Main.f(0x1)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;D:/coding/ztesoft/golang/src/defer2.go:32 +0x187
Main.f(0x2)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;D:/coding/ztesoft/golang/src/defer2.go:32 +0x187
Main.f(0x3)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;D:/coding/ztesoft/golang/src/defer2.go:32 +0x187
Main.main()
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;D:/coding/ztesoft/golang/src/defer2.go:26 +0xc9
Exit status 2


&nbsp;0 1 2 3 4 5 6 7 8
+---+
|
+---+

```flow
St=&gt;start: Start
Op=&gt;operation: Your Operation
Cond=&gt;condition: Yes or No?
e=&gt;end
St-&gt;op-&gt;cond
Cond(yes)-&gt;e
Cond(no)-&gt;op
```


</pre></body></html>