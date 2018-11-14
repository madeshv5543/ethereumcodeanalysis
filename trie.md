
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>The package trie implements Merkle Patricia Tries, which is referred to herein as MPT as a data structure. This data structure is actually a Trie tree variant. MPT is a very important data structure in Ethereum for storing user accounts. Status and its changes, transaction information, and receipt information for the transaction. MPT is actually a combination of three data structures: Trie, Patricia Trie, and Merkle. The three data structures are described separately below.

## Trie Tree (quote introduction http://dongxicheng.org/structure/trietree/)
The Trie tree, also known as the dictionary tree, word search tree or prefix tree, is a multi-fork tree structure for fast retrieval. For example, the dictionary tree of English letters is a 26-fork tree, and the digital dictionary tree is a 10-fork tree.

Trie trees can take advantage of the common prefix of strings to save storage space. As shown in the following figure, the trie tree saves 6 strings with 10 nodes: tea, ten, to, in, inn, int:

![image](picture/trie_1.jpg)

In the trie tree, the common prefix for the strings in, inn, and int is "in", so you can store only one copy of "in" to save space. Of course, if there are a large number of strings in the system and these strings have no common prefix, the corresponding trie tree will consume a lot of memory, which is also a disadvantage of the trie tree.

The basic properties of the Trie tree can be summarized as:

- The root node does not contain characters, and each node contains only one character except the root node.
- From the root node to a node, the characters passing through the path are connected, which is the string corresponding to the node.
- All subnodes of each node contain different strings.

## Patricia Tries (prefix tree)
The difference between a prefix tree and a Trie tree is that the Trie tree assigns a node to each string, which degenerates the Trie tree of strings that are long but have no public nodes into an array. In Ethereum, many such nodes are constructed by hackers to cause denial of service attacks. The difference between prefix trees is that if the nodes have a common prefix, then the common prefix is ​​used, otherwise all the remaining nodes are inserted into the same node. The optimization of Patricia relative to Tire is as follows:

![Optimization of Tire to Patricia](picture/patricia_tire.png)

![image](picture/trie_2.png)

The eight Key Value pairs stored in the above figure can see the characteristics of the prefix tree.

|Key | value |
| ------------- | ---: |
|6c0a5c71ec20bq3w|5 |
|6c0a5c71ec20CX7j|27 |
|6c0a5c71781a1FXq|18 |
|6c0a5c71781a9Dog|64 |
|6c0a8f743b95zUfe|30 |
|6c0a8f743b95jx5R|2 |
|6c0a8f740d16y03G|43 |
|6c0a8f740d16vcc1|48 |

## Merkle Tree (Refer to http://blog.csdn.net/wo541075754/article/details/54632929)
Merkle Tree, also commonly referred to as Hash Tree, as the name suggests, is a tree that stores hash values. The leaves of a Merkle tree are the hash values ​​of data blocks (for example, a collection of files or files). A non-leaf node is a hash of its corresponding child node concatenated string.

![image](picture/trie_3.png)

The main function of Merkle Tree is that when I get Top Hash, this hash value represents the information summary of the whole tree. When any data in the tree changes, the value of Top Hash will change. The value of Top Hash is stored in the block header of the blockchain. The block header must be certified by the workload. This means that I can verify the block information as long as I get a block header. Please refer to that blog for more detailed information. There is a detailed introduction.


## EPT's MPT
Each block of Ethereum contains three MPT trees, respectively

- Trading tree
- Receipt tree (some data during the execution of the transaction)
- Status tree (account information, contract account and user account)

In the figure below are two block headers, where state root, tx root receipt root stores the roots of the three trees, and the second block shows when the data of account 175 changes (27 -&gt; 45). Only need to store some of the data related to this account, and the data in the old block can still be accessed normally. (This is somewhat similar to the implementation of immutable data structures in functional programming languages)
![image](picture/trie_4.png)
Detailed structure is
![world state trie](picture/worldstatetrie.png)

##黄 Mer书 Formal Definition (Appendix D. Modified Merkle Patricia Tree)

Formally, we assume that the input value J contains a collection of Key Value pairs (Key Value is a byte array):
![image](picture/trie_5.png)

When dealing with such a collection, we use the following to identify the Key and Value of the data (for any I in the J set, I0 represents Key, I1 represents Value)

![image](picture/trie_6.png)

For any particular byte, we can represent it as the corresponding nibble (nibble), where the Y set is specified in Hex-Prefix Encoding, meaning a nibble (4bit) set (the reason for using nibbles, Corresponding to the branch node structure of the branch node and the encoding flag in the key)

![image](picture/trie_7.png)

We define the TRIE function to represent the HASH value of the root of the tree (where the second parameter of the c function is the number of layers of the tree after the build is completed. The value of root is 0)

![image](picture/trie_8.png)

We also define a function n, the node function of this trie. When composing nodes, we use RLP to encode the structure. As a means of reducing storage complexity, for nodes with RLP less than 32 bytes, we store their RLP values ​​directly, and for those larger, we store their HASH nodes.
We use c to define the node composition function:

![image](picture/trie_9.png)

In a manner similar to a radix tree, a single key-value pair can be constructed when the Trie tree traverses from root to leaf. Key acquires a single nibble from each branch node (like the radix tree) by traversal accumulation. Unlike a radix tree, in the case of multiple Keys sharing the same prefix, or in the case of a single Key with a unique suffix, two optimization nodes are provided. In the case of a single key with a unique suffix, two optimization nodes are provided. Therefore, when traversing, it is possible to potentially acquire multiple nibbles from each of the other two node types, extensions, and leaves. There are three types of nodes in the Trie tree:

- ** Leaf node: ** The leaf node contains two fields, the first field is the nibble encoding of the remaining Key, and the second parameter of the nibble encoding method is true, the second The field is Value
- **Extended: ** The extended node also contains two fields, the first field is the nibble encoding of the remaining Key's part that can be shared by at least two remaining nodes, the second field Is n(J,j)
- **Branch: ** The branch node contains 17 fields whose first 16 items correspond to each of the sixteen possible nibble values ​​of the keys in which they are traversed. The 17th field stores the nodes that have ended at the current node (for example, there are three keys, respectively (abc, abd, ab). The 17th field stores the value of the ab node)

Branch nodes are only used when needed. For a Trie tree with only one non-null key value pair, there may be no branch nodes. If you use a formula to define these three nodes, the formula is as follows:
The HP function in the figure represents Hex-Prefix Encoding, which is a nibble encoding format, and RLP is a function that uses RLP for serialization.

![image](picture/trie_10.png)

Explanation of the three cases in the above figure

- If there is only one piece of data left in the KV set that currently needs to be encoded, then this data is encoded according to the first rule.
- If the KV set that currently needs to be encoded has a common prefix, then the largest common prefix is ​​extracted and processed using the second rule.
- If it is not the above two cases, then use the branch node for set splitting, because the key is encoded using HP, so the possible branch is only 16 branches of 0-15. It can be seen that the value of u is recursively defined by n, and if a node is just finished here, then the 17th element v is prepared for this case.

For how data should be stored and how it should not be stored, the Yellow Book describes the definitions that are not shown. So this is an implementation issue. We simply define a function to map J to a hash. We believe that for any J, there is only one hash value.

### Formal definition of the Yellow Book (Hex-Prefix Encoding) - hexadecimal prefix encoding
Hexadecimal prefix encoding is an efficient way to encode any number of nibbles into a byte array. It is capable of storing additional flags that, when used in the Trie tree (the only place that will be used), will disambiguate between node types.

It is defined as a function HP that maps from a series of nibbles (represented by the set Y) to a sequence of bytes (represented by set B) along with a Boolean value:

![image](picture/hp_1.png)

Therefore, the upper nibble of the first byte contains two flags; the lowest bit encodes the length of the parity bit, and the second lowest bit encodes the value of the flag. In the case of an even number of nibbles, the lower nibble of the first byte is zero, and in the case of an odd number, the first nibble. All remaining nibbles (now even) are suitable for the remaining bytes.

## Source implementation
### trie/encoding.go
Encoding.go mainly deals with the work of converting the three encoding formats in the trie tree. The three encoding formats are the following three encoding formats.

- **KEYBYTES encoding** This encoding format is a native key byte array. Most of Trie's APIs use this encoding format.
- **HEX encoding** This encoding format contains one nibble of Key for each byte, and an optional 'terminal' for the tail. The 'terminal' indicates whether the node is a leaf node or an extended node. . This node is used when the node is loaded into memory because of its convenient access.
- **COMPACT encoding** This encoding format is the Hex-Prefix Encoding mentioned in the yellow book above. This encoding format can be regarded as another version of the encoding format *HEX encoding**, which can be stored in Save disk space when the database is in use.

Simply understood as: encode the ordinary byte sequence keybytes into keybytes with t flag and odd nibble nibble flag bits.
- keybytes is normal information stored in full byte (8bit)
- hex is a format for storing information in nibble (4bit). For compact use
- In order to facilitate the key of the node of the Modified Merkle Patricia Tree in the Yellow Book, the code is hex format with the length of the even number of segments. Its first nibble nibble will store the t flag and the odd flag from high to low in the lower 2 bits. The key bytes encoded by compact are easy to store when the t mark of the hex is added and the nibble of the nibble is even (ie, the complete byte).

The code implementation mainly implements the mutual conversion of these three codes and a method of obtaining a common prefix.

Func hexToCompact(hex []byte) []byte {
Terminator := byte(0)
If hasTerm(hex) {
Terminator = 1
Hex = hex[:len(hex)-1]
}
Buf := make([]byte, len(hex)/2+1)
Buf[0] = terminator &lt;&lt; 5 // the flag byte
If len(hex)&amp;1 == 1 {
Buf[0] |= 1 &lt;&lt; 4 // odd flag
Buf[0] |= hex[0] // first nibble is contained in the first byte
Hex = hex[1:]
}
decodeNibbles(hex, buf[1:])
Return buf
}

Func compactToHex(compact []byte) []byte {
Base := keybytesToHex(compact)
Base = base[:len(base)-1]
// apply terminator flag
If base[0] &gt;= 2 { // TODO First delete the end-end flag of the keybytesToHex output, and then add back to the flag bit t of the first half byte. Operational redundancy
Base = append(base, 16)
}
// apply odd flag
Chop := 2 - base[0]&amp;1
Return base[chop:]
}

Func keybytesToHex(str []byte) []byte {
l := len(str)*2 + 1
Var nibbles = make([]byte, l)
For i, b := range str {
Nibbles[i*2] = b / 16
Nibbles[i*2+1] = b % 16
}
Nibbles[l-1] = 16
Return nibbles
}

// hexToKeybytes turns hex nibbles into key bytes.
// This can only be used for keys of even length.
Func hexToKeybytes(hex []byte) []byte {
If hasTerm(hex) {
Hex = hex[:len(hex)-1]
}
If len(hex)&amp;1 != 0 {
Panic("can't convert hex key of odd length")
}
Key := make([]byte, (len(hex)+1)/2) // TODO For a len(hex) that has been judged to be even, add 1 to the integer 2, which is invalid +1 logic.
decodeNibbles(hex, key)
Return key
}

Func decodeNibbles(nibbles []byte, bytes []byte) {
For bi, ni := 0, 0; ni &lt; len(nibbles); bi, ni = bi+1, ni+2 {
Bytes[bi] = nibbles[ni]&lt;&lt;4 | nibbles[ni+1]
}
}

// prefixLen returns the length of the common prefix of a and b.
Func prefixLen(a, b []byte) int {
Var i, length = 0, len(a)
If len(b) &lt; length {
Length = len(b)
}
For ; i &lt; length; i++ {
If a[i] != b[i] {
Break
}
}
Return i
}

// hasTerm returns whether a hex key has the terminator flag.
Func hasTerm(s []byte) bool {
Return len(s) &gt; 0 &amp;&amp; s[len(s)-1] == 16
}

### data structure
The structure of node, you can see that node is divided into 4 types, fullNode corresponds to the branch node in the Yellow Book, shortNode corresponds to the extension node and leaf node in the Yellow Book (by the type of shortNode.Val to correspond to the leaf node (valueNode) Or branch node (fullNode)

Type node interface {
Fstring(string) string
Cache() (hashNode, bool)
canUnload(cachegen, cachelimit uint16) bool
}

Type (
fullNode struct {
Children [17]node // Actual trie node data to encode/decode (needs custom encoder)
Flags nodeFlag
}
shortNode struct {
Key []byte
Val node
Flags nodeFlag
}
hashNode []byte
valueNode []byte
)

The structure of trie, root contains the current root node, db is the back-end KV storage, the structure of trie is finally stored in the form of KV to the database, and then needs to be loaded from the database when starting. originalRoot starts the hash value when loading, and the hash value can be used to recover the entire trie tree in the database. The cachegen field indicates the cache age of the current Trie tree. Each time the Commit operation is invoked, the cache age of the Trie tree is increased. The cache era will be attached to the node node. If the current cache age - cachelimit parameter is greater than the node cache age, node will be unloaded from the cache to save memory. In fact, this is the cache update LRU algorithm, if a cache is not used for a long time, then it is removed from the cache to save memory space.

// Trie is a Merkle Patricia Trie.
// The zero value is an empty trie with no database.
// Use New to create a trie that sits on top of a database.
//
// Trie is not safe for concurrent use.
Type Trie struct {
Root node
Db database
originalRoot common.Hash

// Cache generation values.
//cachegen increases by one with each commit operation.
// new nodes are tagged with the current generation and unloaded
// when their generation is older than than cachegen-cachelimit.
Cachegen, cachelimit uint16
}


###Trie tree insertion, find and delete
The initialization of the Trie tree calls the New function. The function accepts a hash value and a Database parameter. If the hash value is not null, it means that an existing Trie tree is loaded from the database, and the trea.resolveHash method is called to load the whole Trie tree, this method will be introduced later. If root is empty, then create a new Trie tree to return.

Func New(root common.Hash, db Database) (*Trie, error) {
Trie := &amp;Trie{db: db, originalRoot: root}
If (root != common.Hash{}) &amp;&amp; root != emptyRoot {
If db == nil {
Panic("trie.New: cannot use existing root without a database")
}
Rootnode, err := trie.resolveHash(root[:], nil)
If err != nil {
Return nil, err
}
Trie.root = rootnode
}
Return trie, nil
}

The insertion of the Trie tree, this is a recursive call method, starting from the root node, looking down until you find the point you can insert and insert. The parameter node is the currently inserted node, the prefix is ​​the part of the key that has been processed so far, and the key is the part of the key that has not been processed yet, the complete key = prefix + key. Value is the value that needs to be inserted. The return value bool is whether the operation changes the Trie tree (dirty), node is the root node of the subtree after the insertion is completed, and error is an error message.

- If the node type is nil (the node of a brand new Trie tree is nil), the whole tree is empty at this time, directly return shortNode{key, value, t.newFlag()}, this time the whole tree It also contains a shortNode node.
- If the current root node type is shortNode (that is, a leaf node), first calculate the common prefix. If the common prefix is ​​equal to key, then the two keys are the same, if the value is the same (dirty == false), Then return an error. Update the value of shortNode and return if there are no errors. If the common prefix does not match exactly, then the common prefix needs to be extracted to form a separate node (extended node), the extended node is connected to a branch node, and the branch node is connected to the two short nodes. First build a branch node (branch := &amp;fullNode{flags: t.newFlag()}), and then call the t.insert of the branch node's Children location to insert the remaining two short nodes. There is a small detail here, the key encoding is HEX encoding, and there is a terminal at the end. Considering that the key of our root node is abc0x16, the key of the node we inserted is ab0x16. The following branch.Children[key[matchlen]] will work fine, 0x16 just points to the 17th child of the branch node. If the length of the match is 0, then the branch node is returned directly, otherwise the shortNode node is returned as the prefix node.
- If the current node is a fullNode (that is, a branch node), then the insert method is called directly to the corresponding child node, and then the corresponding child node only wants the newly generated node.
- If the current node is a hashNode, the hashNode means that the current node has not been loaded into the memory, or is stored in the database, then first call t.resolveHash(n, prefix) to load into the memory, and then call the loaded node. Insert method to insert.


Insert code

Func (t *Trie) insert(n node, prefix, key []byte, value node) (bool, node, error) {
If len(key) == 0 {
If v, ok := n.(valueNode); ok {
Return !bytes.Equal(v, value.(valueNode)), value, nil
}
Return true, value, nil
}
Switch n := n.(type) {
Case *shortNode:
Matchlen := prefixLen(key, n.Key)
// If the whole key matches, keep this short node as is
// and only update the value.
If matchlen == len(n.Key) {
Dirty, nn, err := t.insert(n.Val, append(prefix, key[:matchlen]...), key[matchlen:], value)
If !dirty || err != nil {
Return false, n, err
}
Return true, &amp;shortNode{n.Key, nn, t.newFlag()}, nil
}
// Otherwise branch out at the index where they differ.
Branch := &amp;fullNode{flags: t.newFlag()}
Var err error
_, branch.Children[n.Key[matchlen]], err = t.insert(nil, append(prefix, n.Key[:matchlen+1]...), n.Key[matchlen+1:], n.Val)
If err != nil {
Return false, nil, err
}
_, branch.Children[key[matchlen]], err = t.insert(nil, append(prefix, key[:matchlen+1]...), key[matchlen+1:], value)
If err != nil {
Return false, nil, err
}
// Replace this shortNode with the branch if it occurs at index 0.
If matchlen == 0 {
Return true, branch, nil
}
// Otherwise, replace it with a short node leading up to the branch.
Return true, &amp;shortNode{key[:matchlen], branch, t.newFlag()}, nil

Case *fullNode:
Dirty, nn, err := t.insert(n.Children[key[0]], append(prefix, key[0]), key[1:], value)
If !dirty || err != nil {
Return false, n, err
}
n = n.copy()
N.flags = t.newFlag()
n.Children[key[0]] = nn
Return true, n, nil

Case nil:
Return true, &amp;shortNode{key, value, t.newFlag()}, nil

Case hashNode:
// We've hit a part of the trie that isn't loaded yet. Load
// the node and insert into it. This leaves all child nodes on
// the path to the value in the trie.
Rn, err := t.resolveHash(n, prefix)
If err != nil {
Return false, nil, err
}
Dirty, nn, err := t.insert(rn, prefix, key, value)
If !dirty || err != nil {
Return false, rn, err
}
Return true, nn, nil

Default:
Panic(fmt.Sprintf("%T: invalid node: %v", n, n))
}
}


The Get method of the Trie tree basically simply traverses the Trie tree to get the Key information.


Func (t *Trie) tryGet(origNode node, key []byte, pos int) (value []byte, newnode node, didResolve bool, err error) {
Switch n := (origNode).(type) {
Case nil:
Return nil, nil, false, nil
Case valueNode:
Return n, n, false, nil
Case *shortNode:
If len(key)-pos &lt; len(n.Key) || !bytes.Equal(n.Key, key[pos:pos+len(n.Key)]) {
// key not found in trie
Return nil, n, false, nil
}
Value, newnode, didResolve, err = t.tryGet(n.Val, key, pos+len(n.Key))
If err == nil &amp;&amp; didResolve {
n = n.copy()
n.Val = newnode
N.flags.gen = t.cachegen
}
Return value, n, didResolve, err
Case *fullNode:
Value, newnode, didResolve, err = t.tryGet(n.Children[key[pos]], key, pos+1)
If err == nil &amp;&amp; didResolve {
n = n.copy()
N.flags.gen = t.cachegen
n.Children[key[pos]] = newnode
}
Return value, n, didResolve, err
Case hashNode:
Child, err := t.resolveHash(n, key[:pos])
If err != nil {
Return nil, n, true, err
}
Value, newnode, _, err := t.tryGet(child, key, pos)
Return value, newnode, true, err
Default:
Panic(fmt.Sprintf("%T: invalid node: %v", origNode, origNode))
}
}

The Delete method of the Trie tree is not introduced at the moment, and the code root insertion is similar.

### Serialization and deserialization of Trie trees
Serialization mainly refers to storing the data represented by the memory into the database. Deserialization refers to loading the Trie data in the database into the data represented by the memory. The purpose of serialization is mainly to facilitate storage, reduce storage size and so on. The purpose of deserialization is to load the stored data into memory, facilitating the insertion, query, modification, etc. of the Trie tree.

Trie's serialization mainly works with the Compat Encoding and RLP encoding formats described earlier. The serialized structure is described in detail in the Yellow Book.

![image](picture/trie_8.png)
![image](picture/trie_9.png)
![image](picture/trie_10.png)

The use of the Trie tree has a more detailed reference in trie_test.go. Here I list a simple usage process. First create an empty Trie tree, then insert some data, and finally call the trie.Commit () method to serialize and get a hash value (root), which is the KEC (c (J, 0)) in the above figure or TRIE (J).

Func TestInsert(t *testing.T) {
Trie := newEmpty()
updateString(trie, "doe", "reindeer")
updateString(trie, "dog", "puppy")
updateString(trie, "do", "cat")
Root, err := trie.Commit()
}

Let's analyze the main process of Commit(). After a series of calls, the hash method of hasher.go is finally called.

Func (t *Trie) Commit() (root common.Hash, err error) {
If t.db == nil {
Panic("Commit called on trie with nil database")
}
Return t.CommitTo(t.db)
}
// CommitTo writes all nodes to the given database.
// Nodes are stored with their sha3 hash as the key.
//
// Committing flushes nodes from memory. Subsequent Get calls will
// load nodes from the trie's database. Calling code must ensure that
// the changes made to db are written back to the trie's attached
// database before using the trie.
Func (t *Trie) CommitTo(db DatabaseWriter) (root common.Hash, err error) {
Hash, cached, err := t.hashRoot(db)
If err != nil {
Return (common.Hash{}), err
}
T.root = cached
T.cachegen++
Return common.BytesToHash(hash.(hashNode)), nil
}

Func (t *Trie) hashRoot(db DatabaseWriter) (node, node, error) {
If t.root == nil {
Return hashNode(emptyRoot.Bytes()), nil, nil
}
h := newHasher(t.cachegen, t.cachelimit)
Defer returnHasherToPool(h)
Return h.hash(t.root, db, true)
}


Below we briefly introduce the hash method, the hash method mainly does two operations. One is to retain the original tree structure, and use the cache variable, the other is to calculate the hash of the original tree structure and store the hash value in the cache variable.

The main process of calculating the original hash value is to first call h.hashChildren(n, db) to find the hash values ​​of all the child nodes, and replace the original child nodes with the hash values ​​of the child nodes. This is a recursive call process that counts up from the leaves up to the root of the tree. Then call the store method to calculate the hash value of the current node, and put the hash value of the current node into the cache node, set the dirty parameter to false (the dirty value of the newly created node is true), and then return.

The return value indicates that the cache variable contains the original node node and contains the hash value of the node node. The hash variable returns the hash value of the current node (this value is actually calculated from all child nodes of node and node).

There is a small detail: when the root node calls the hash function, the force parameter is true, and the other child nodes call the force parameter to false. The purpose of the force parameter is to perform a hash calculation on c(J,i) when ||c(J,i)||&lt;32, so that the root node is hashed anyway.

// hash collapses a node down into a hash node, also returning a copy of the
// original node initialized with the computed hash to replace the original one.
Func (h *hasher) hash(n node, db DatabaseWriter, force bool) (node, node, error) {
// If we're not storing the node, just hashing, use available cached data
If hash, dirty := n.cache(); hash != nil {
If db == nil {
Return hash, n, nil
}
If n.canUnload(h.cachegen, h.cachelimit) {
// Unload the node from cache. All of its subnodes will have a lower or equal
// cache generation number.
cacheUnloadCounter.Inc(1)
Return hash, hash, nil
}
If !dirty {
Return hash, n, nil
}
}
// Trie not processed yet or needs storage, walk the children
Collapsed, cached, err := h.hashChildren(n, db)
If err != nil {
Return hashNode{}, n, err
}
Hashed, err := h.store(collapsed, db, force)
If err != nil {
Return hashNode{}, n, err
}
// Cache the hash of the node for later reuse and remove
// the dirty flag in commit mode. It's fine to assign these values ​​directly
// without copying the node first because hashChildren copies it.
cachedHash, _ := hashed.(hashNode)
Switch cn := cached.(type) {
Case *shortNode:
Cn.flags.hash = cachedHash
If db != nil {
Cn.flags.dirty = false
}
Case *fullNode:
Cn.flags.hash = cachedHash
If db != nil {
Cn.flags.dirty = false
}
}
Return hashed, cached, nil
}

The hashChildren method, which replaces all child nodes with their hashes. You can see that the cache variable takes over the complete structure of the original Trie tree, and the collapsed variable replaces the child nodes with the hash values ​​of the child nodes.

- If the current node is a shortNode, first replace the collapsed.Key from Hex Encoding with Compact Encoding, and then recursively call the hash method to calculate the child node's hash and cache, thus replacing the child node with the child node's hash value.
- If the current node is fullNode, then traverse each child node and replace the child node with the hash value of the child node,
- Otherwise this node has no children. Return directly.

Code

// hashChildren replaces the children of a node with their hashes if the encoded
// size of the child is larger than a hash, returning the collapsed node as well
// as a replacement for the original node with the child hashes cached in.
Func (h *hasher) hashChildren(original node, db DatabaseWriter) (node, node, error) {
Var err error

Switch n := original.(type) {
Case *shortNode:
// Hash the short node's child, caching the newly hashed subtree
Collapsed, cached := n.copy(), n.copy()
collapsed.Key = hexToCompact(n.Key)
cached.Key = common.CopyBytes(n.Key)

If _, ok := n.Val.(valueNode); !ok {
collapsed.Val, cached.Val, err = h.hash(n.Val, db, false)
If err != nil {
Return original, original, err
}
}
If collapsed.Val == nil {
collapsed.Val = valueNode(nil) // Ensure that nil children are encoded as empty strings.
}
Return collapsed, cached, nil

Case *fullNode:
// Hash the full node's children, caching the newly hashed subtrees
Collapsed, cached := n.copy(), n.copy()

For i := 0; i &lt; 16; i++ {
If n.Children[i] != nil {
collapsed.Children[i], cached.Children[i], err = h.hash(n.Children[i], db, false)
If err != nil {
Return original, original, err
}
} else {
collapsed.Children[i] = valueNode(nil) // Ensure that nil children are encoded as empty strings.
}
}
cached.Children[16] = n.Children[16]
If collapsed.Children[16] == nil {
collapsed.Children[16] = valueNode(nil)
}
Return collapsed, cached, nil

Default:
// Value and hash nodes don't have children so they're left as were
Return n, original, nil
}
}


Store method, if all the child nodes of a node are replaced with the hash value of the child node, then directly call the rlp.Encode method to encode the node. If the encoded value is less than 32, and the node is not the root node, then Store them directly in their parent node, otherwise call the h.sha.Write method to perform the hash calculation, then store the hash value and the encoded data in the database, and then return the hash value.

You can see that the value of each node with a value greater than 32 and the hash are stored in the database.

Func (h *hasher) store(n node, db DatabaseWriter, force bool) (node, error) {
// Don't store hashes or empty nodes.
If _, isHash := n.(hashNode); n == nil || isHash {
Return n, nil
}
// Generate the RLP encoding of the node
h.tmp.Reset()
If err := rlp.Encode(h.tmp, n); err != nil {
Panic("encode error: " + err.Error())
}

If h.tmp.Len() &lt; 32 &amp;&amp; !force {
Return n, nil // Nodes smaller than 32 bytes are stored inside their parent
}
// Larger nodes are replaced by their hash and stored in the database.
Hash, _ := n.cache()
If hash == nil {
h.sha.Reset()
h.sha.Write(h.tmp.Bytes())
Hash = hashNode(h.sha.Sum(nil))
}
If db != nil {
Return hash, db.Put(hash, h.tmp.Bytes())
}
Return hash, nil
}


Trie's deserialization process. Remember the process of creating a Trie tree before. If the parameter root's hash value is not empty, then the rootnode, err := trie.resolveHash(root[:], nil) method is called to get the rootnode node. First, the RLP encoded content of the node is obtained from the database through the hash value. Then call decodeNode to parse the content.

Func (t *Trie) resolveHash(n hashNode, prefix []byte) (node, error) {
cacheMissCounter.Inc(1)

Enc, err := t.db.Get(n)
If err != nil || enc == nil {
Return nil, &amp;MissingNodeError{NodeHash: common.BytesToHash(n), Path: prefix}
}
Dec := mustDecodeNode(n, enc, t.cachegen)
Return dec, nil
}
Func mustDecodeNode(hash, buf []byte, cachegen uint16) node {
n, err := decodeNode(hash, buf, cachegen)
If err != nil {
Panic(fmt.Sprintf("node %x: %v", hash, err))
}
Return n
}

The decodeNode method, this method determines the node to which the encoding belongs according to the length of the list of the rlp. If it is 2 fields, it is the shortNode node. If it is 17 fields, it is fullNode, and then calls the respective analytic functions.

// decodeNode parses the RLP encoding of a trie node.
Func decodeNode(hash, buf []byte, cachegen uint16) (node, error) {
If len(buf) == 0 {
Return nil, io.ErrUnexpectedEOF
}
Elems, _, err := rlp.SplitList(buf)
If err != nil {
Return nil, fmt.Errorf("decode error: %v", err)
}
Switch c, _ := rlp.CountValues(elems); c {
Case 2:
n, err := decodeShort(hash, buf, elems, cachegen)
Return n, wrapError(err, "short")
Case 17:
n, err := decodeFull(hash, buf, elems, cachegen)
Return n, wrapError(err, "full")
Default:
Return nil, fmt.Errorf("invalid number of list elements: %v", c)
}
}

The decodeShort method determines whether a leaf node or an intermediate node is determined by whether the key has a terminal symbol. If there is a terminal, then the leaf node, parse out val through the SplitString method and generate a shortNode. However, there is no terminator, then the description is the extension node, the remaining nodes are resolved by decodeRef, and then a shortNode is generated.

Func decodeShort(hash, buf, elems []byte, cachegen uint16) (node, error) {
Kbuf, rest, err := rlp.SplitString(elems)
If err != nil {
Return nil, err
}
Flag := nodeFlag{hash: hash, gen: cachegen}
Key := compactToHex(kbuf)
If hasTerm(key) {
// value node
Val, _, err := rlp.SplitString(rest)
If err != nil {
Return nil, fmt.Errorf("invalid value node: %v", err)
}
Return &amp;shortNode{key, append(valueNode{}, val...), flag}, nil
}
r, _, err := decodeRef(rest, cachegen)
If err != nil {
Return nil, wrapError(err, "val")
}
Return &amp;shortNode{key, r, flag}, nil
}

The decodeRef method parses according to the data type. If the type is list, then it may be the value of the content &lt;32, then call decodeNode to parse. If it is an empty node, it returns null. If it is a hash value, construct a hashNode to return. Note that there is no further analysis here. If you need to continue parsing the hashNode, you need to continue to call the resolveHash method. The call to the decodeShort method is complete.

Func decodeRef(buf []byte, cachegen uint16) (node, []byte, error) {
Kind, val, rest, err := rlp.Split(buf)
If err != nil {
Return nil, buf, err
}
Switch {
Case kind == rlp.List:
			// 'embedded' node reference. The encoding must be smaller
			// than a hash in order to be valid.
			if size := len(buf) - len(rest); size &gt; hashLen {
				err := fmt.Errorf("oversized embedded node (size is %d bytes, want size &lt; %d)", size, hashLen)
				return nil, buf, err
}
			n, err := decodeNode(nil, buf, cachegen)
			return n, rest, err
		case kind == rlp.String &amp;&amp; len(val) == 0:
			// empty node
			return nil, rest, nil
		case kind == rlp.String &amp;&amp; len(val) == 32:
			return append(hashNode{}, val...), rest, nil
Default:
			return nil, nil, fmt.Errorf("invalid RLP string size %d (want 0 or 32)", len(val))
}
}

decodeFull方法。根decodeShort方法的流程差不多。


	func decodeFull(hash, buf, elems []byte, cachegen uint16) (*fullNode, error) {
		n := &amp;fullNode{flags: nodeFlag{hash: hash, gen: cachegen}}
		for i := 0; i &lt; 16; i++ {
			cld, rest, err := decodeRef(elems, cachegen)
If err != nil {
				return n, wrapError(err, fmt.Sprintf("[%d]", i))
}
			n.Children[i], elems = cld, rest
}
		val, _, err := rlp.SplitString(elems)
If err != nil {
			return n, err
}
		if len(val) &gt; 0 {
			n.Children[16] = append(valueNode{}, val...)
}
		return n, nil
}


### Trie树的cache管理
Trie树的cache管理。 还记得Trie树的结构里面有两个参数， 一个是cachegen,一个是cachelimit。这两个参数就是cache控制的参数。 Trie树每一次调用Commit方法，会导致当前的cachegen增加1。

	func (t *Trie) CommitTo(db DatabaseWriter) (root common.Hash, err error) {
		hash, cached, err := t.hashRoot(db)
If err != nil {
			return (common.Hash{}), err
}
		t.root = cached
		t.cachegen++
		return common.BytesToHash(hash.(hashNode)), nil
}

然后在Trie树插入的时候，会把当前的cachegen存放到节点中。

	func (t *Trie) insert(n node, prefix, key []byte, value node) (bool, node, error) {
				....
				return true, &amp;shortNode{n.Key, nn, t.newFlag()}, nil
}

	// newFlag returns the cache flag value for a newly created node.
	func (t *Trie) newFlag() nodeFlag {
		return nodeFlag{dirty: true, gen: t.cachegen}
}

如果 trie.cachegen - node.cachegen &gt; cachelimit，就可以把节点从内存里面卸载掉。 也就是说节点经过几次Commit，都没有修改，那么就把节点从内存里面卸载，以便节约内存给其他节点使用。

卸载过程在我们的 hasher.hash方法中， 这个方法是在commit的时候调用。如果方法的canUnload方法调用返回真，那么就卸载节点，观察他的返回值，只返回了hash节点，而没有返回node节点，这样节点就没有引用，不久就会被gc清除掉。 节点被卸载掉之后，会用一个hashNode节点来表示这个节点以及其子节点。 如果后续需要使用，可以通过方法把这个节点加载到内存里面来。

	func (h *hasher) hash(n node, db DatabaseWriter, force bool) (node, node, error) {
		if hash, dirty := n.cache(); hash != nil {
			if db == nil {
				return hash, n, nil
}
			if n.canUnload(h.cachegen, h.cachelimit) {
				// Unload the node from cache. All of its subnodes will have a lower or equal
				// cache generation number.
				cacheUnloadCounter.Inc(1)
				return hash, hash, nil
}
			if !dirty {
				return hash, n, nil
}
}

canUnload方法是一个接口，不同的node调用不同的方法。

	// canUnload tells whether a node can be unloaded.
	func (n *nodeFlag) canUnload(cachegen, cachelimit uint16) bool {
		return !n.dirty &amp;&amp; cachegen-n.gen &gt;= cachelimit
}

	func (n *fullNode) canUnload(gen, limit uint16) bool  { return n.flags.canUnload(gen, limit) }
	func (n *shortNode) canUnload(gen, limit uint16) bool { return n.flags.canUnload(gen, limit) }
	func (n hashNode) canUnload(uint16, uint16) bool      { return false }
	func (n valueNode) canUnload(uint16, uint16) bool     { return false }

	func (n *fullNode) cache() (hashNode, bool)  { return n.flags.hash, n.flags.dirty }
	func (n *shortNode) cache() (hashNode, bool) { return n.flags.hash, n.flags.dirty }
	func (n hashNode) cache() (hashNode, bool)   { return nil, true }
	func (n valueNode) cache() (hashNode, bool)  { return nil, true }

### proof.go Trie树的默克尔证明
主要提供两个方法，Prove方法获取指定Key的proof证明， proof证明是从根节点到叶子节点的所有节点的hash值列表。 VerifyProof方法，接受一个roothash值和proof证明和key来验证key是否存在。

Prove方法，从根节点开始。把经过的节点的hash值一个一个存入到list中。 Then return.

	// Prove constructs a merkle proof for key. The result contains all
	// encoded nodes on the path to the value at key. The value itself is
	// also included in the last node and can be retrieved by verifying
	// the proof.
//
	// If the trie does not contain a value for key, the returned proof
	// contains all nodes of the longest existing prefix of the key
	// (at least the root node), ending with the node that proves the
	// absence of the key.
	func (t *Trie) Prove(key []byte) []rlp.RawValue {
		// Collect all nodes on the path to key.
		key = keybytesToHex(key)
		nodes := []node{}
		tn := t.root
		for len(key) &gt; 0 &amp;&amp; tn != nil {
			switch n := tn.(type) {
			case *shortNode:
				if len(key) &lt; len(n.Key) || !bytes.Equal(n.Key, key[:len(n.Key)]) {
					// The trie doesn't contain the key.
					tn = nil
} else {
					tn = n.Val
					key = key[len(n.Key):]
}
Nodes = append(nodes, n)
			case *fullNode:
				tn = n.Children[key[0]]
				key = key[1:]
Nodes = append(nodes, n)
			case hashNode:
				var err error
				tn, err = t.resolveHash(n, nil)
If err != nil {
					log.Error(fmt.Sprintf("Unhandled trie error: %v", err))
Return nil
}
Default:
				panic(fmt.Sprintf("%T: invalid node: %v", tn, tn))
}
}
		hasher := newHasher(0, 0)
		proof := make([]rlp.RawValue, 0, len(nodes))
		for i, n := range nodes {
			// Don't bother checking for errors here since hasher panics
			// if encoding doesn't work and we're not writing to any database.
			n, _, _ = hasher.hashChildren(n, nil)
			hn, _ := hasher.store(n, nil, false)
			if _, ok := hn.(hashNode); ok || i == 0 {
				// If the node's database encoding is a hash (or is the
				// root node), it becomes a proof element.
				enc, _ := rlp.EncodeToBytes(n)
				proof = append(proof, enc)
}
}
		return proof
}

VerifyProof方法，接收一个rootHash参数，key参数，和proof数组， 来一个一个验证是否能够和数据库里面的能够对应上。

	// VerifyProof checks merkle proofs. The given proof must contain the
	// value for key in a trie with the given root hash. VerifyProof
	// returns an error if the proof contains invalid trie nodes or the
	// wrong value.
	func VerifyProof(rootHash common.Hash, key []byte, proof []rlp.RawValue) (value []byte, err error) {
		key = keybytesToHex(key)
		sha := sha3.NewKeccak256()
		wantHash := rootHash.Bytes()
		for i, buf := range proof {
			sha.Reset()
			sha.Write(buf)
			if !bytes.Equal(sha.Sum(nil), wantHash) {
				return nil, fmt.Errorf("bad proof node %d: hash mismatch", i)
}
			n, err := decodeNode(wantHash, buf, 0)
If err != nil {
				return nil, fmt.Errorf("bad proof node %d: %v", i, err)
}
			keyrest, cld := get(n, key)
			switch cld := cld.(type) {
Case nil:
				if i != len(proof)-1 {
					return nil, fmt.Errorf("key mismatch at proof node %d", i)
} else {
					// The trie doesn't contain the key.
					return nil, nil
}
			case hashNode:
				key = keyrest
				wantHash = cld
			case valueNode:
				if i != len(proof)-1 {
					return nil, errors.New("additional nodes at end of proof")
}
				return cld, nil
}
}
		return nil, errors.New("unexpected end of proof")
}

	func get(tn node, key []byte) ([]byte, node) {
For {
			switch n := tn.(type) {
			case *shortNode:
				if len(key) &lt; len(n.Key) || !bytes.Equal(n.Key, key[:len(n.Key)]) {
					return nil, nil
}
				tn = n.Val
				key = key[len(n.Key):]
			case *fullNode:
				tn = n.Children[key[0]]
				key = key[1:]
			case hashNode:
				return key, n
Case nil:
				return key, nil
			case valueNode:
				return nil, n
Default:
				panic(fmt.Sprintf("%T: invalid node: %v", tn, tn))
}
}
}


### security_trie.go 加密的Trie
为了避免刻意使用很长的key导致访问时间的增加， security_trie包装了一下trie树， 所有的key都转换成keccak256算法计算的hash值。同时在数据库里面存储hash值对应的原始的key。

	type SecureTrie struct {
		trie             Trie    //原始的Trie树
		hashKeyBuf       [secureKeyLength]byte   //计算hash值的buf
		secKeyBuf        [200]byte               //hash值对应的key存储的时候的数据库前缀
		secKeyCache      map[string][]byte      //记录hash值和对应的key的映射
		secKeyCacheOwner *SecureTrie // Pointer to self, replace the key cache on mismatch
}

	func NewSecure(root common.Hash, db Database, cachelimit uint16) (*SecureTrie, error) {
		if db == nil {
			panic("NewSecure called with nil database")
}
		trie, err := New(root, db)
If err != nil {
Return nil, err
}
		trie.SetCacheLimit(cachelimit)
		return &amp;SecureTrie{trie: *trie}, nil
}

	// Get returns the value for key stored in the trie.
	// The value bytes must not be modified by the caller.
	func (t *SecureTrie) Get(key []byte) []byte {
		res, err := t.TryGet(key)
If err != nil {
			log.Error(fmt.Sprintf("Unhandled trie error: %v", err))
}
		return res
}

	// TryGet returns the value for key stored in the trie.
	// The value bytes must not be modified by the caller.
	// If a node was not found in the database, a MissingNodeError is returned.
	func (t *SecureTrie) TryGet(key []byte) ([]byte, error) {
		return t.trie.TryGet(t.hashKey(key))
}
	func (t *SecureTrie) CommitTo(db DatabaseWriter) (root common.Hash, err error) {
		if len(t.getSecKeyCache()) &gt; 0 {
			for hk, key := range t.secKeyCache {
				if err := db.Put(t.secKey([]byte(hk)), key); err != nil {
Return common.Hash{}, err
}
}
			t.secKeyCache = make(map[string][]byte)
}
		return t.trie.CommitTo(db)
}
</pre></body></html>