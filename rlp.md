
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>RLP is short for Recursive Length Prefix. In the serialization method in Ethereum, all objects in Ethereum are serialized into byte arrays using RLP methods. Here I hope to first understand the RLP method from the Yellow Book, and then analyze the actual implementation through code.

## The formal definition of the Yellow Book
We define the set T. T is defined by the following formula

![image](picture/rlp_1.png)

The O in the above figure represents a collection of all bytes, then B represents all possible byte arrays, and L represents a tree structure of more than one single node (such as a structure, or a branch node of a tree node, a non-leaf node) , T represents a combination of all byte arrays and tree structures.

We use two sub-functions to define the RLP functions, which handle the two structures (L or B) mentioned above.

![image](picture/rlp_2.png)

For all B type byte arrays. We have defined the following processing rules.

- If the byte array contains only one byte and the size of this byte is less than 128, then the data is not processed and the result is the original data.
- If the length of the byte array is less than 56, then the result of the processing is equal to the prefix of the original data (128 + bytes of data length).
- If it is not the above two cases, then the result of the processing is equal to the big endian representation of the original data length in front of the original data, and then preceded by (183 + the length of the big end of the original data)

The following uses a formulated language to represent

![image](picture/rlp_3.png)

**Explanation of some mathematical symbols**

- ||x|| represents the length of x
- (a).(b,c).(d,e) = (a,b,c,d,e) Represents the operation of concat, which is the addition of strings. "hello "+"world" = "hello world"
- The BE(x) function actually removes the big endian mode of leading 0. For example, the 4-byte integer 0x1234 is represented by big endian mode. 00 00 12 34 Then the result returned by the BE function is actually 12 34. The extra 00 at the beginning is removed.
- ^ The symbol represents and the meaning.
- The equal sign of the "three" form represents the meaning of identity

For all other types (tree structures), we define the following processing rules

First we use RLP processing for each element in the tree structure, and then concatenate these results.

- If the length of the connected byte is less than 56, then we add (192 + the length of the connection) in front of the result of the connection to form the final result.
- If the length of the byte after the connection is greater than or equal to 56, then we add the big endian mode of the length after the connection before the result of the connection, and then add the length of the big endian mode (247 + length after the connection) )

The following is expressed in a formulaic language, and it is found that the formula is clearer.

![image](picture/rlp_4.png)
It can be seen that the above is a recursive definition, and the RLP method is called in the process of obtaining s(x), so that RLP can process the recursive data structure.


If RLP is used to process scalar data, RLP can only be used to process positive integers. RLP can only handle integers processed in big endian mode. That is to say, if it is an integer x, then use the BE(x) function to convert x to the simplest big endian mode (with the 00 at the beginning removed), and then encode the result of BE(x) as a byte array.

If you use the formula to represent it is the following figure.

![image](picture/rlp_5.png)

When parsing RLP data. If you just need to parse the shaping data, this time you encounter the leading 00, this time you need to be treated as an exception.


**to sum up**

RLP treats all data as a combination of two types of data, one is a byte array, and the other is a data structure similar to List. I understand that these two classes basically contain all the data structures. For example, use more structs. Can be seen as a list of many different types of fields


### RLP source analysis
RLP source code is not a lot, mainly divided into three files

Decode.go decoder to decode RLP data into go data structure
Decode_tail_test.go decoder test code
Decode_test.go decoder test code
Doc.go document code
Encode.go encoder that serializes the data structure of GO to a byte array
Encode_test.go encoder test
Encode_example_test.go
Raw.go undecoded RLP data
Raw_test.go
The typecache.go type cache, the type cache records the contents of type-&gt;(encoder|decoder).


#### How to find the corresponding encoder and decoder according to the type typecache.go
In languages ​​such as C++ or Java that support overloading, you can override the same function name by different types to implement methods for different types of dispatch. For example, you can also use generics to implement function dispatch.

String encode(int);
String encode(long);
String encode(struct test*)

However, the GO language itself does not support overloading and there is no generics, so the assignment of functions needs to be implemented by itself. Typecache.go is mainly used for this purpose, to quickly find its own encoder function and decoder function by its own type.

Let's first look at the core data structure.

Var (
typeCacheMutex sync.RWMutex //Read-write lock, used to protect typeCache this map when multi-threaded
typeCache = make(map[typekey]*typeinfo) // core data structure, save type -&gt; codec function
)
Type typeinfo struct { // stores the encoder and decoder functions
Decoder
Writer
}
Type typekey struct {
reflect.Type
// the key must include the struct tags because they
//gene generate a different decoder.
Tags
}

You can see that the core data structure is the typeCache map, the map key is the type, and the value is the corresponding code and decoder.

Here's how the user gets the encoder and decoder functions.


Func cachedTypeInfo(typ reflect.Type, tags tags) (*typeinfo, error) {
typeCacheMutex.RLock() //Add a read lock to protect,
Info := typeCache[typekey{typ, tags}]
typeCacheMutex.RUnlock()
If info != nil { //If you get the information successfully, then return
Return info, nil
}
// not in the cache, need to generate info for this type.
typeCacheMutex.Lock() // Otherwise add write lock to call cachedTypeInfo1 function to create and return. It should be noted that in multi-threaded environment, it is possible that multiple threads can call this place at the same time, so you need to judge when you enter cachedTypeInfo1 method. Has it been successfully created by another thread?
Defer typeCacheMutex.Unlock()
Return cachedTypeInfo1(typ, tags)
}

Func cachedTypeInfo1(typ reflect.Type, tags tags) (*typeinfo, error) {
Key := typekey{typ, tags}
Info := typeCache[key]
If info != nil {
// Other threads may have been created successfully, then we get the information directly and then return
Return info, nil
}
// put a dummmy value into the cache before generating.
// if the generator tries to lookup itself, it will get
// the dummy value and won't call itself recursively.
/ / This place first created a value to fill the position of this type, to avoid encountering some recursively defined data types to form an infinite loop
typeCache[key] = new(typeinfo)
Info, err := genTypeInfo(typ, tags)
If err != nil {
// remove the dummy value if the generator fails
Delete(typeCache, key)
Return nil, err
}
*typeCache[key] = *info
Return typeCache[key], err
}

genTypeInfo is a codec function that generates the corresponding type.

Func genTypeInfo(typ reflect.Type, tags tags) (info *typeinfo, err error) {
Info = new(typeinfo)
If info.decoder, err = makeDecoder(typ, tags); err != nil {
Return nil, err
}
If info.writer, err = makeWriter(typ, tags); err != nil {
Return nil, err
}
Return info, nil
}

The processing logic of makeDecoder is roughly the same as the processing logic of makeWriter. Here I only post the processing logic of makeWriter.

// makeWriter creates a writer function for the given type.
Func makeWriter(typ reflect.Type, ts tags) (writer, error) {
Kind := typ.Kind()
Switch {
Case typ == rawValueType:
Return writeRawValue, nil
Case typ.Implements(encoderInterface):
Return writeEncoder, nil
Case kind != reflect.Ptr &amp;&amp; reflect.PtrTo(typ).Implements(encoderInterface):
Return writeEncoderNoPtr, nil
Case kind == reflect.Interface:
Return writeInterface, nil
Case typ.AssignableTo(reflect.PtrTo(bigInt)):
Return writeBigIntPtr, nil
Case typ.AssignableTo(bigInt):
Return writeBigIntNoPtr, nil
Case isUint(kind):
Return writeUint, nil
Case kind == reflect.Bool:
Return writeBool, nil
Case kind == reflect.String:
Return writeString, nil
Case kind == reflect.Slice &amp;&amp; isByte(typ.Elem()):
Return writeBytes, nil
Case kind == reflect.Array &amp;&amp; isByte(typ.Elem()):
Return writeByteArray, nil
Case kind == reflect.Slice || kind == reflect.Array:
Return makeSliceWriter(typ, ts)
Case kind == reflect.Struct:
Return makeStructWriter(typ)
Case kind == reflect.Ptr:
Return makePtrWriter(typ)
Default:
Return nil, fmt.Errorf("rlp: type %v is not RLP-serializable", typ)
}
}

You can see that it is a switch case, assigning different handlers depending on the type. This processing logic is still very simple. It is very simple for the simple type, and it can be processed according to the description above in the yellow book. However, the handling of the structure type is quite interesting, and this part of the detailed processing logic can not be found in the Yellow Book.

Type field struct {
Index int
Info *typeinfo
}
Func makeStructWriter(typ reflect.Type) (writer, error) {
Fields, err := structFields(typ)
If err != nil {
Return nil, err
}
Writer := func(val reflect.Value, w *encbuf) error {
Lh := w.list()
For _, f := range fields {
//f is the field structure, f.info is a pointer to typeinfo, so here is actually the encoder method that calls the field.
If err := f.info.writer(val.Field(f.index), w); err != nil {
Return err
}
}
w.listEnd(lh)
Return nil
}
Return writer, nil
}

This function defines the encoding of the structure. The structFields method gets the encoders for all the fields, and then returns a method that traverses all the fields, each of which calls its encoder method.

Func structFields(typ reflect.Type) (fields []field, err error) {
For i := 0; i &lt; typ.NumField(); i++ {
If f := typ.Field(i); f.PkgPath == "" { //
Tags, err := parseStructTag(typ, i)
If err != nil {
Return nil, err
}
If tags.ignored {
Continue
}
Info, err := cachedTypeInfo1(f.Type, tags)
If err != nil {
Return nil, err
}
Fields = append(fields, field{i, info})
}
}
Return fields, nil
}

The structFields function iterates over all the fields and then calls cachedTypeInfo1 for each field. You can see that this is a recursive calling process. One of the above code to note is that f.PkgPath == "" This is for all exported fields. The so-called exported field is the field that starts with an uppercase letter.


#### Encoder encode.go
First define the value of the empty string and the empty List, which are 0x80 and 0xC0. Note that the corresponding value of the shaped zero value is also 0x80. This is not defined above the yellow book. Then define an interface type to implement EncodeRLP for other types.

Var (
// Common encoded values.
// These are useful when implementing EncodeRLP.
EmptyString = []byte{0x80}
EmptyList = []byte{0xC0}
)

// Encoder is implemented by types that require custom
// encoding rules or want to encode private fields.
Type Encoder interface {
// EncodeRLP should write the RLP encoding of its receiver to w.
// If the implementation is a pointer method, it may also be
// called for nil pointers.
//
// Implementations should generate valid RLP. The data written is
// not verified at the moment, but a future version might. It is
// recommended to write only a single value but writing multiple
// values ​​or no value at all is also permitted.
EncodeRLP(io.Writer) error
}

Then define a most important method, most of the EncodeRLP methods directly call this method Encode method. This method first gets an encbuf object. Then call the encode method of this object. In the encode method, the reflection type of the object is first obtained, its encoder is obtained according to the reflection type, and then the writer method of the encoder is called. This is related to the typecache mentioned above.

Func Encode(w io.Writer, val interface{}) error {
If outer, ok := w.(*encbuf); ok {
// Encode was called by some type's EncodeRLP.
// Avoid copying by writing to the outer encbuf directly.
Return outer.encode(val)
}
Eb := encbufPool.Get().(*encbuf)
Defer encbufPool.Put(eb)
Eb.reset()
If err := eb.encode(val); err != nil {
Return err
}
Return eb.toWriter(w)
}
Func (w *encbuf) encode(val interface{}) error {
Rval := reflect.ValueOf(val)
Ti, err := cachedTypeInfo(rval.Type(), tags{})
If err != nil {
Return err
}
Return ti.writer(rval, w)
}

##### Introduction of encbuf
Encbuf is short for encode buffer (I guess). Encbuf appears in the Encode method, and in many Writer methods. As the name suggests, this is the role of the buffer in the process of encoding. Let's take a look at the definition of encbuf.

Type encbuf struct {
Str []byte // string data, contains everything except list headers
Lheads []*listhead // all list headers
Lhsize int // sum of sizes of all encoded list headers
Sizebuf []byte // 9-byte auxiliary buffer for uint encoding
}

Type listhead struct {
Offset int // index of this header in string data
Size int // total size of encoded data (including list headers)
}

As you can see from the comments, the str field contains all the content except the head of the list. The header of the list is recorded in the lheads field. The lhsize field records the length of lheads. The sizebuf is a 9-byte auxiliary buffer that is used to process the encoding of the uint. The listhead consists of two fields, the offset field records where the list data is in the str field, and the size field records the total length of the encoded data containing the list header. You can see the picture below.

![image](picture/rlp_6.png)

For ordinary types, such as string, integer, bool and other data, it is directly filled into the str field. However, for the processing of structure types, special processing methods are required. Take a look at the makeStructWriter method mentioned above.

Func makeStructWriter(typ reflect.Type) (writer, error) {
Fields, err := structFields(typ)
...
Writer := func(val reflect.Value, w *encbuf) error {
Lh := w.list()
For _, f := range fields {
If err := f.info.writer(val.Field(f.index), w); err != nil {
Return err
}
}
w.listEnd(lh)
}
}

You can see that the above code reflects the special processing method for processing structure data, that is, first call the w.list() method, and then call the listEnd(lh) method after processing. The reason for adopting this method is that we do not know how long the length of the processed structure is when we first start processing the structure, because it is necessary to determine the processing method of the head according to the length of the structure (recall the structure inside the yellow book) The processing method of the body), so we record the position of the str before processing, and then start processing each field. After processing, look at how much the str data is added to know how long the processed structure is.

Func (w *encbuf) list() *listhead {
Lh := &amp;listhead{offset: len(w.str), size: w.lhsize}
W.lheads = append(w.lheads, lh)
Return lh
}

Func (w *encbuf) listEnd(lh *listhead) {
Lh.size = w.size() - lh.offset - lh.size //lh.size records the length of the queue header when the list starts. w.size() returns the length of str plus lhsize
If lh.size &lt; 56 {
W.lhsize += 1 // length encoded into kind tag
} else {
W.lhsize += 1 + intsize(uint64(lh.size))
}
}
Func (w *encbuf) size() int {
Return len(w.str) + w.lhsize
}

Then we can look at the final processing logic of encbuf, process the listhead and assemble it into complete RLP data.

Func (w *encbuf) toBytes() []byte {
Out := make([]byte, w.size())
Strpos := 0
Pos := 0
For _, head := range w.lheads {
// write string data before header
n := copy(out[pos:], w.str[strpos:head.offset])
Pos += n
Strpos += n
// write the header
Enc := head.encode(out[pos:])
Pos += len(enc)
}
// copy string data after the last list header
Copy(out[pos:], w.str[strpos:])
Return out
}


#####writer Introduction
The rest of the process is actually quite simple. It is to fill each of the different data into the encbuf according to the yellow book.

Func writeBool(val reflect.Value, w *encbuf) error {
If val.Bool() {
W.str = append(w.str, 0x01)
} else {
W.str = append(w.str, 0x80)
}
Return nil
}
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


#### Decoder decode.go
The general flow of the decoder is similar to that of the encoder. Understand the general flow of the encoder and know the general flow of the decoder.

Func (s *Stream) Decode(val interface{}) error {
If val == nil {
Return errDecodeIntoNil
}
Rval := reflect.ValueOf(val)
Rtyp := rval.Type()
If rtyp.Kind() != reflect.Ptr {
Return errNoPointer
}
If rval.IsNil() {
Return errDecodeIntoNil
}
Info, err := cachedTypeInfo(rtyp.Elem(), tags{})
If err != nil {
Return err
}
Err = info.decoder(s, rval.Elem())
If decErr, ok := err.(*decodeError); ok &amp;&amp; len(decErr.ctx) &gt; 0 {
// add decode target type to error so context has more meaning
decErr.ctx = append(decErr.ctx, fmt.Sprint("(", rtyp.Elem(), ")"))
}
Return err
}

Func makeDecoder(typ reflect.Type, tags tags) (dec decoder, err error) {
Kind := typ.Kind()
Switch {
Case typ == rawValueType:
Return decodeRawValue, nil
Case typ.Implements(decoderInterface):
Return decodeDecoder, nil
Case kind != reflect.Ptr &amp;&amp; reflect.PtrTo(typ).Implements(decoderInterface):
Return decodeDecoderNoPtr, nil
Case typ.AssignableTo(reflect.PtrTo(bigInt)):
Return decodeBigInt, nil
Case typ.AssignableTo(bigInt):
Return decodeBigIntNoPtr, nil
Case isUint(kind):
Return decodeUint, nil
Case kind == reflect.Bool:
Return decodeBool, nil
Case kind == reflect.String:
Return decodeString, nil
Case kind == reflect.Slice || kind == reflect.Array:
Return makeListDecoder(typ, tags)
Case kind == reflect.Struct:
Return makeStructDecoder(typ)
Case kind == reflect.Ptr:
If tags.nilOK {
Return makeOptionalPtrDecoder(typ)
}
Return makePtrDecoder(typ)
Case kind == reflect.Interface:
Return decodeInterface, nil
Default:
Return nil, fmt.Errorf("rlp: type %v is not RLP-serializable", typ)
}
}

We also look at the specific decoding process through the decoding process of the structure type. Similar to the encoding process, first get all the fields that need to be decoded through structFields, and then decode each field. There is almost a List() and ListEnd() operation with the encoding process, but the processing flow here is not the same as the encoding process, which will be described in detail in subsequent chapters.

Func makeStructDecoder(typ reflect.Type) (decoder, error) {
Fields, err := structFields(typ)
If err != nil {
Return nil, err
}
Dec := func(s *Stream, val reflect.Value) (err error) {
If _, err := s.List(); err != nil {
Return wrapStreamError(err, typ)
}
For _, f := range fields {
Err := f.info.decoder(s, val.Field(f.index))
If err == EOL {
Return &amp;decodeError{msg: "too few elements", typ: typ}
} else if err != nil {
Return addErrorContext(err, "."+typ.Field(f.index).Name)
}
}
Return wrapStreamError(s.ListEnd(), typ)
}
Return dec, nil
}

Let's look at the decoding process of strings. Because strings of different lengths have different ways of encoding, we can get the type of the string by the difference of the prefix. Here we get the type that needs to be parsed by s.Kind() method. The length, if it is a Byte type, directly returns the value of Byte. If it is a String type, it reads the value of the specified length and returns. This is the purpose of the kind() method.

Func (s *Stream) Bytes() ([]byte, error) {
Kind, size, err := s.Kind()
If err != nil {
Return nil, err
}
Switch kind {
Case Byte:
S.kind = -1 // rearm Kind
Return []byte{s.byteval}, nil
Case String:
b := make([]byte, size)
If err = s.readFull(b); err != nil {
Return nil, err
}
If size == 1 &amp;&amp; b[0] &lt; 128 {
Return nil, ErrCanonSize
}
Return b, nil
Default:
Return nil, ErrExpectedString
}
}

##### Stream Structure Analysis
The other code of the decoder is similar to the structure of the encoder, but there is a special structure that is not in the encoder. That is Stream.
This is an auxiliary class used to read the RLP in a streaming manner. Earlier we talked about the general decoding process is to first get the type and length of the object to be decoded through the Kind () method, and then decode the data according to the length and type. So how do we deal with the structure's fields and the structure's data? Recall that when we process the structure, we first call the s.List() method, then decode each field, and finally call s.EndList(). method. The trick is in these two methods. Let's take a look at these two methods.

Type Stream struct {
r ByteReader
// number of bytes remaining to be read from r.
Remaining uint64
Limited bool
// auxiliary buffer for integer decoding
Uintbuf []byte
Kind Kind // kind of value ahead
Size uint64 // size of value ahead
Byteval byte // value of single byte in type tag
Kinderr error // error from last readKind
Stack []listpos
}
Type listpos struct{ pos, size uint64 }

Stream's List method, when calling the List method. We first call the Kind method to get the type and length. If the type doesn't match, we throw an error. Then we push a listpos object onto the stack. This object is the key. The pos field of this object records how many bytes of data the current list has read, so it must be 0 at the beginning. The size field records how many bytes of data the list object needs to read. So when I process each subsequent field, every time I read some bytes, it will increase the value of the pos field. After processing, it will compare whether the pos field and the size field are equal. If they are not equal, an exception will be thrown. .

Func (s *Stream) List() (size uint64, err error) {
Kind, size, err := s.Kind()
If err != nil {
Return 0, err
}
If kind != List {
Return 0, ErrExpectedList
}
S.stack = append(s.stack, listpos{0, size})
S.kind = -1
S.size = 0
Return size, nil
}

Stream's ListEnd method, if the current number of data read pos is not equal to the declared data length size, throw an exception, and then pop operation on the stack, if the current stack is not empty, then add pos on the top of the stack The length of the currently processed data (used to handle this situation - the structure's field is the structure, this recursive structure)

Func (s *Stream) ListEnd() error {
If len(s.stack) == 0 {
Return errNotInList
}
Tos := s.stack[len(s.stack)-1]
If tos.pos != tos.size {
Return errNotAtEOL
}
S.stack = s.stack[:len(s.stack)-1] // pop
If len(s.stack) &gt; 0 {
S.stack[len(s.stack)-1].pos += tos.size
}
S.kind = -1
S.size = 0
Return nil
}


</pre></body></html>