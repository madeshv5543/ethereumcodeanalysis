
<!-- saved from url=(0051)https://translate.googleusercontent.com/translate_f -->
<html><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8"></head><body><pre>RLPx Encryption (RLPx encryption)

The discovery node discovery protocol introduced earlier, because the data carried is not very important, basically the plaintext transmission.

Each node will open two identical ports, one for UDP port and one for TCP discovery, which is used to carry service data. The port number of the UDP port and the TCP port are the same. In this way, as long as the port is discovered through UDP, it is equivalent to using TCP to connect to the corresponding port.

The RLPx protocol defines the encryption process for TCP links.

RLPx uses (Perfect Forward Secrecy), in simple terms. The two sides of the link generate a random private key, and the public key is obtained by a random private key. The two parties then exchange their respective public keys so that both parties can generate a shared key (shared-secret) by their own random private key and the other party's public key. Subsequent communications use this shared key as the key to the symmetric encryption algorithm. In this way. If one day's private key is leaked, it will only affect the security of the message after the leak. It is safe for the previous communication (because the communication key is randomly generated and disappears after use).


## Forward Security (cited from Wikipedia)
Forward security or forward secrecy (English: Forward Secrecy, abbreviation: FS), sometimes referred to as Perfect Forward Secrecy (PFS), is the security of communication protocols in cryptography. Attributes refer to long-term use of master key leaks that do not cause past session key leaks. [2] Forward security protects communications that were conducted in the past without the threat of passwords or keys being exposed in the future. [3] If the system has forward security, it can guarantee that if the password or key is accidentally leaked at a certain moment, the communication that has been carried out in the past is still safe and will not be affected, even if the system is actively attacked. in this way.

### Diffie-Hellman Key Exchange
Diffie-Hellman key exchange (D-H) is a security protocol. It allows the parties to create a key over an insecure channel without any prior information from the other party. This key can be used as a symmetric key to encrypt the communication content in subsequent communications. The concept of public key exchange was first proposed by Ralph C. Merkle, and this key exchange method was developed by Bailey Whitfield Diffie and Martin Edward Hellman. First published in 1976. Martin Herman once argued that this key exchange method should be called Diffie–Hellman–Merkle key exchange.

- Synonyms for Diffie-Hellman key exchange include:
- Diffie-Hellman key agreement
- Diffie-Hellman key creation
- Index key exchange
- Diffie-Herman Agreement

Although the Diffie-Hellman key exchange itself is an anonymous (non-authenticated) key exchange protocol, it is the basis of many authentication protocols and is used to provide a complete pre-transmission mode in the transport layer security protocol. To safety.

#### Description
Defi Herman exchanges a message over the common channel to create a shared secret that can be used for secure communication over the common channel.
The following explains its process (including the mathematical part of the algorithm):
![image](picture/rlpx_1.png)

The simplest, the earliest proposed protocol uses an integer modulo n multiplicative group of prime numbers p and its original root g. The algorithm is shown below, green for non-secret information, red for bold for secret information:
![image](picture/rlpx_2.png)
![image](picture/rlpx_3.png)

## p2p/rlpx.go source code interpretation
This file implements the link protocol of RLPx.

The general process of link contact is as follows:

1. doEncHandshake() This method is used to complete the process of exchanging keys and creating an encrypted channel. If it fails, the link is closed.
2. doProtoHandshake() This method is used to negotiate between protocol features, such as the protocol version of both parties, whether to support the Snappy encryption method.


After the link has been processed twice, it is built up. Because TCP is a streaming protocol. All RLPx protocols define the way of framing. All data can be understood as one after another rlpxFrame. Rlpx reads and writes are handled by the rlpxFrameRW object.

### doEncHandshake
The originator of the link is called the initiator. The passive recipient of the link is called a receiver. The process of processing in these two modes is different. After completing the handshake. Generated a sec. It can be understood as a key that gets symmetric encryption. Then created a newRLPXFrameRW frame reader. Complete the process of creating an encrypted channel.


Func (t *rlpx) doEncHandshake(prv *ecdsa.PrivateKey, dial *discover.Node) (discover.NodeID, error) {
Var (
Sec secrets
Err error
)
If dial == nil {
Sec, err = receiverEncHandshake(t.fd, prv, nil)
} else {
Sec, err = initiatorEncHandshake(t.fd, prv, dial.ID, nil)
}
If err != nil {
Return discover.NodeID{}, err
}
t.wmu.Lock()
T.rw = newRLPXFrameRW(t.fd, sec)
t.wmu.Unlock()
Return sec.RemoteID, nil
}

The initiatorEncHandshake first looks at the operation of the originator of the link. First created authMsg with makeAuthMsg. Then send it to the peer through the network. Then read the response from the peer through readHandshakeMsg. Finally, the secrets are created by calling secrets.

// initiatorEncHandshake negotiates a session token on conn.
// it should be called on the dialing side of the connection.
//
// prv is the local client's private key.
Func initiatorEncHandshake(conn io.ReadWriter, prv *ecdsa.PrivateKey, remoteID discover.NodeID, token []byte) (s secrets, err error) {
h := &amp;encHandshake{initiator: true, remoteID: remoteID}
authMsg, err := h.makeAuthMsg(prv, token)
If err != nil {
Return s, err
}
authPacket, err := sealEIP8(authMsg, h)
If err != nil {
Return s, err
}
If _, err = conn.Write(authPacket); err != nil {
Return s, err
}

authRespMsg := new(authRespV4)
authRespPacket, err := readHandshakeMsg(authRespMsg, encAuthRespLen, prv, conn)
If err != nil {
Return s, err
}
If err := h.handleAuthResp(authRespMsg); err != nil {
Return s, err
}
Return h.secrets(authPacket, authRespPacket)
}

makeAuthMsg. This method creates the handshake message of the initiator. First, the public key of the peer can be obtained through the ID of the peer. So the public key of the peer is known to the person who initiated the connection. But for the connected person, the public key of the peer should be unknown.

// makeAuthMsg creates the initiator handshake message.
Func (h *encHandshake) makeAuthMsg(prv *ecdsa.PrivateKey, token []byte) (*authMsgV4, error) {
Rpub, err := h.remoteID.Pubkey()
If err != nil {
Return nil, fmt.Errorf("bad remoteID: %v", err)
}
h.remotePub = ecies.ImportECDSAPublic(rpub)
// Generate random initiator nonce.
// Generate a random initial value to avoid replay attacks? Or to avoid guessing keys through multiple connections?
h.initNonce = make([]byte, shaLen)
If _, err := rand.Read(h.initNonce); err != nil {
Return nil, err
}
// Generate random keypair to for ECDH.
/ / Generate a random private key
h.randomPrivKey, err = ecies.GenerateKey(rand.Reader, crypto.S256(), nil)
If err != nil {
Return nil, err
}

// Sign known message: static-shared-secret ^ nonce
// This place should be a static shared secret directly. A shared secret generated using your own private key and the other's public key.
Token, err = h.staticSharedSecret(prv)
If err != nil {
Return nil, err
}
//Here I understand using a shared secret to encrypt this initNonce.
Signed := xor(token, h.initNonce)
// Encrypt this information with a random private key.
Signature, err := crypto.Sign(signed, h.randomPrivKey.ExportECDSA())
If err != nil {
Return nil, err
}

Msg := new(authMsgV4)
Copy(msg.Signature[:], signature)
//This tells the other party the public key of the initiator. In this way, the other party can generate a static shared secret using its own private key and this public key.
Copy(msg.InitiatorPubkey[:], crypto.FromECDSAPub(&amp;prv.PublicKey)[1:])
Copy(msg.Nonce[:], h.initNonce)
msg.Version = 4
Return msg, nil
}

// staticSharedSecret returns the static shared secret, the result
// of key agreement between the local and remote static node key.
Func (h *encHandshake) staticSharedSecret(prv *ecdsa.PrivateKey) ([]byte, error) {
Return ecies.ImportECDSA(prv).GenerateShared(h.remotePub, sskLen, sskLen)
}

The sealEIP8 method, which is a grouping method, encodes rml for msg. Fill in some data. The data is then encrypted using the other party's public key. This means that only the other party's private key can decrypt this information.

Func sealEIP8(msg interface{}, h *encHandshake) ([]byte, error) {
Buf := new(bytes.Buffer)
If err := rlp.Encode(buf, msg); err != nil {
Return nil, err
}
// pad with random amount of data. the amount needs to be at least 100 bytes to make
// the message distinguishable from pre-EIP-8 handshakes.
Pad := padSpace[:mrand.Intn(len(padSpace)-100)+100]
buf.Write(pad)
Prefix := make([]byte, 2)
binary.BigEndian.PutUint16(prefix, uint16(buf.Len()+eciesOverhead))

Enc, err := ecies.Encrypt(rand.Reader, h.remotePub, buf.Bytes(), nil, prefix)
Return append(prefix, enc...), err
}

The readHandshakeMsg method will be called from two places. One is in the initiatorEncHandshake. One is in receiverEncHandshake. This method is relatively simple. First try decoding in one format. If not, change the other one. It should be a compatibility setting. Basically, it uses its own private key for decoding and then calls rlp to decode into a structure. The description of the structure is the following authRespV4, the most important of which is the random public key of the opposite end. Both parties can get the same shared secret through their private key and the random public key of the peer. And this shared secret is not available to third parties.


// RLPx v4 handshake response (defined in EIP-8).
Type authRespV4 struct {
RandomPubkey [pubLen]byte
Nonce [shaLen]byte
Version uint

// Ignore additional fields (forward-compatibility)
Rest []rlp.RawValue `rlp:"tail"`
}


Func readHandshakeMsg(msg plainDecoder, plainSize int, prv *ecdsa.PrivateKey, r io.Reader) ([]byte, error) {
Buf := make([]byte, plainSize)
If _, err := io.ReadFull(r, buf); err != nil {
Return buf, err
}
// Attempt decoding pre-EIP-8 "plain" format.
Key := ecies.ImportECDSA(prv)
If dec, err := key.Decrypt(rand.Reader, buf, nil, nil); err == nil {
msg.decodePlain(dec)
Return buf, nil
}
// Could be EIP-8 format, try that.
Prefix := buf[:2]
Size := binary.BigEndian.Uint16(prefix)
If size &lt; uint16(plainSize) {
Return buf, fmt.Errorf("size underflow, need at least %d bytes", plainSize)
}
Buf = append(buf, make([]byte, size-uint16(plainSize)+2)...)
If _, err := io.ReadFull(r, buf[plainSize:]); err != nil {
Return buf, err
}
Dec, err := key.Decrypt(rand.Reader, buf[2:], nil, prefix)
If err != nil {
Return buf, err
}
// Can't use rlp.DecodeBytes here because it rejects
// trailing data (forward-compatibility).
s := rlp.NewStream(bytes.NewReader(dec), 0)
Return buf, s.Decode(msg)
}



The handleAuthResp method is very simple.

Func (h *encHandshake) handleAuthResp(msg *authRespV4) (err error) {
h.respNonce = msg.Nonce[:]
h.remoteRandomPub, err = importPublicKey(msg.RandomPubkey[:])
Return err
}

Finally, the secrets function, which is called after the handshake is completed. It generates a shared secret through its own random private key and the peer's public key. This shared secret is instantaneous (only exists in the current link). So when one day the private key is cracked. The previous news is still safe.

// secrets is called after the handshake is completed.
// It extracts the connection secrets from the handshake values.
Func (h *encHandshake) secrets(auth, authResp []byte) (secrets, error) {
ecdheSecret, err := h.randomPrivKey.GenerateShared(h.remoteRandomPub, sskLen, sskLen)
If err != nil {
Return secrets{}, err
}

// derive base secrets from ephemeral key agreement
sharedSecret := crypto.Keccak256(ecdheSecret, crypto.Keccak256(h.respNonce, h.initNonce))
aesSecret := crypto.Keccak256(ecdheSecret, sharedSecret)
// In fact, this MAC protects the shared secret of ecdheSecret. Three values ​​of respNonce and initNonce
s := secrets{
RemoteID: h.remoteID,
AES: aesSecret,
MAC: crypto.Keccak256(ecdheSecret, aesSecret),
}

// setup sha3 instances for the MACs
Mac1 := sha3.NewKeccak256()
mac1.Write(xor(s.MAC, h.respNonce))
mac1.Write(auth)
Mac2 := sha3.NewKeccak256()
mac2.Write(xor(s.MAC, h.initNonce))
mac2.Write(authResp)
//Each packet received will check if its MAC value meets the calculated result. If the instructions are not met, there is a problem.
If h.initiator {
s.EgressMAC, s.IngressMAC = mac1, mac2
} else {
s.EgressMAC, s.IngressMAC = mac2, mac1
}

Return s, nil
}


The receiverEncHandshake function is roughly the same as the initiatorEncHandshake. But the order is a bit different.

// receiverEncHandshake negotiates a session token on conn.
// it should be called on the listening side of the connection.
//
// prv is the local client's private key.
// token is the token from a previous session with this node.
Func receiverEncHandshake(conn io.ReadWriter, prv *ecdsa.PrivateKey, token []byte) (s secrets, err error) {
authMsg := new(authMsgV4)
authPacket, err := readHandshakeMsg(authMsg, encAuthMsgLen, prv, conn)
If err != nil {
Return s, err
}
h := new(encHandshake)
If err := h.handleAuthMsg(authMsg, prv); err != nil {
Return s, err
}

authRespMsg, err := h.makeAuthResp()
If err != nil {
Return s, err
}
Var authRespPacket []byte
If authMsg.gotPlain {
authRespPacket, err = authRespMsg.sealPlain(h)
} else {
authRespPacket, err = sealEIP8(authRespMsg, h)
}
If err != nil {
Return s, err
}
If _, err = conn.Write(authRespPacket); err != nil {
Return s, err
}
Return h.secrets(authPacket, authRespPacket)
}

### doProtocolHandshake
This method is relatively simple, and the encrypted channel has been created. We saw that it was only agreed to use Snappy encryption and then quit.

// doEncHandshake runs the protocol handshake using authenticated
// messages. the protocol handshake is the next authenticated message
// and also verifies whether the encryption handshake 'worked' and the
// remote side actually provided the right public key.
Func (t *rlpx) doProtoHandshake(our *protoHandshake) (their *protoHandshake, err error) {
// Writing our handshake happens concurrently, we prefer
// returning the handshake read error. If the remote side
// disconnects us early with a valid reason, we should return it
// as the error so it can be tracked elsewhere.
Werr := make(chan error, 1)
Go func() { werr &lt;- Send(t.rw, handshakeMsg, our) }()
If their, err = readProtocolHandshake(t.rw, our); err != nil {
&lt;-werr // make sure the write terminates too
Return nil, err
}
If err := &lt;-werr; err != nil {
Return nil, fmt.Errorf("write error: %v", err)
}
// If the protocol version supports Snappy encoding, upgrade immediately
T.rw.snappy = their.Version &gt;= snappyProtocolVersion

Return their, nil
}


### rlpxFrameRW Data Framing
Data framing is mainly done by the rlpxFrameRW class.


// rlpxFrameRW implements a simplified version of RLPx framing.
// chunked messages are not supported and all headers are equal to
// zeroHeader.
//
// rlpxFrameRW is not safe for concurrent use from multiple goroutines.
Type rlpxFrameRW struct {
Conn io.ReadWriter
Enc cipher.Stream
Dec cipher.Stream

macCipher cipher.Block
egressMAC hash.Hash
ingressMAC hash.Hash

Snappy bool
}

We are after completing two handshakes. This object was created by calling the newRLPXFrameRW method.

T.rw = newRLPXFrameRW(t.fd, sec)

Then provide the ReadMsg and WriteMsg methods. These two methods directly call ReadMsg and WriteMsg of rlpxFrameRW


Func (t *rlpx) ReadMsg() (Msg, error) {
t.rmu.Lock()
Defer t.rmu.Unlock()
t.fd.SetReadDeadline(time.Now().Add(frameReadTimeout))
Return t.rw.ReadMsg()
}
Func (t *rlpx) WriteMsg(msg Msg) error {
t.wmu.Lock()
Defer t.wmu.Unlock()
t.fd.SetWriteDeadline(time.Now().Add(frameWriteTimeout))
Return t.rw.WriteMsg(msg)
}

WriteMsg

Func (rw *rlpxFrameRW) WriteMsg(msg Msg) error {
Ptype, _ := rlp.EncodeToBytes(msg.Code)

// if snappy is enabled, compress message now
If rw.snappy {
If msg.Size &gt; maxUint24 {
Return errPlainMessageTooLarge
}
Payload, _ := ioutil.ReadAll(msg.Payload)
Payload = snappy.Encode(nil, payload)

msg.Payload = bytes.NewReader(payload)
msg.Size = uint32(len(payload))
}
// write header
Headbuf := make([]byte, 32)
Fsize := uint32(len(ptype)) + msg.Size
If fsize &gt; maxUint24 {
Return errors.New("message size overflows uint24")
}
putInt24(fsize, headbuf) // TODO: check overflow
Copy(headbuf[3:], zeroHeader)
rw.enc.XORKeyStream(headbuf[:16], headbuf[:16]) // first half is now encrypted

// write header MAC
Copy(headbuf[16:], updateMAC(rw.egressMAC, rw.macCipher, headbuf[:16]))
If _, err := rw.conn.Write(headbuf); err != nil {
Return err
}

// write encrypted frame, updating the egress MAC hash with
// the data written to conn.
Tee := cipher.StreamWriter{S: rw.enc, W: io.MultiWriter(rw.conn, rw.egressMAC)}
If _, err := tee.Write(ptype); err != nil {
Return err
}
If _, err := io.Copy(tee, msg.Payload); err != nil {
Return err
}
If padding := fsize % 16; padding &gt; 0 {
If _, err := tee.Write(zero16[:16-padding]); err != nil {
Return err
}
}

// write frame MAC. egress MAC hash is up to date because
// frame content was written to it as well.
Fmacseed := rw.egressMAC.Sum(nil)
Mac := updateMAC(rw.egressMAC, rw.macCipher, fmacseed)
_, err := rw.conn.Write(mac)
Return err
}

ReadMsg

Func (rw *rlpxFrameRW) ReadMsg() (msg Msg, err error) {
// read the header
Headbuf := make([]byte, 32)
If _, err := io.ReadFull(rw.conn, headbuf); err != nil {
Return msg, err
}
// verify header mac
shouldMAC := updateMAC(rw.ingressMAC, rw.macCipher, headbuf[:16])
If !hmac.Equal(shouldMAC, headbuf[16:]) {
Return msg, errors.New("bad header MAC")
}
rw.dec.XORKeyStream(headbuf[:16], headbuf[:16]) // first half is now decrypted
Fsize := readInt24(headbuf)
// ignore protocol type for now

// read the frame content
Var rsize = fsize // frame size rounded up to 16 byte boundary
If padding := fsize % 16; padding &gt; 0 {
Rsize += 16 - padding
}
Framebuf := make([]byte, rsize)
If _, err := io.ReadFull(rw.conn, framebuf); err != nil {
Return msg, err
}

// read and validate frame MAC. we can re-use headbuf for that.
rw.ingressMAC.Write(framebuf)
Fmacseed := rw.ingressMAC.Sum(nil)
If _, err := io.ReadFull(rw.conn, headbuf[:16]); err != nil {
Return msg, err
}
shouldMAC = updateMAC(rw.ingressMAC, rw.macCipher, fmacseed)
If !hmac.Equal(shouldMAC, headbuf[:16]) {
Return msg, errors.New("bad frame MAC")
}

// decrypt frame content
rw.dec.XORKeyStream(framebuf, framebuf)

// decode message code
Content := bytes.NewReader(framebuf[:fsize])
If err := rlp.Decode(content, &amp;msg.Code); err != nil {
Return msg, err
}
msg.Size = uint32(content.Len())
msg.Payload = content

// if snappy is enabled, verify and decompress message
If rw.snappy {
Payload, err := ioutil.ReadAll(msg.Payload)
If err != nil {
Return msg, err
}
Size, err := snappy.DecodedLen(payload)
If err != nil {
Return msg, err
}
If size &gt; int(maxUint24) {
Return msg, errPlainMessageTooLarge
}
Payload, err = snappy.Decode(nil, payload)
If err != nil {
Return msg, err
}
msg.Size, msg.Payload = uint32(size), bytes.NewReader(payload)
}
Return msg, nil
}

Frame structure

Normal = not chunked
Chunked-0 = First frame of a multi-frame packet
Chunked-n = Subsequent frames for multi-frame packet
|| is concatenate
^ is xor

Single-frame packet:
Header || header-mac || frame || frame-mac

Multi-frame packet:
Header || header-mac || frame-0 ||
[ header || header-mac || frame-n || ... || ]
Header || header-mac || frame-last || frame-mac

Header: frame-size || header-data || padding
Frame-size: 3-byte integer size of frame, big endian encoded (excludes padding)
Header-data:
Normal: rlp.list(protocol-type[, context-id])
Chunked-0: rlp.list(protocol-type, context-id, total-packet-size)
Chunked-n: rlp.list(protocol-type, context-id)
Values:
Protocol-type: &lt; 2**16
Context-id: &lt; 2**16 (optional for normal frames)
Total-packet-size: &lt; 2**32
Padding: zero-fill to 16-byte boundary

Header-mac: right128 of egress-mac.update(aes(mac-secret,egress-mac) ^ header-ciphertext).digest

Frame:
Normal: rlp(packet-type) [|| rlp(packet-data)] || padding
Chunked-0: rlp(packet-type) || rlp(packet-data...)
Chunked-n: rlp(...packet-data) || padding
Padding: zero-fill to 16-byte boundary (only necessary for last frame)

Frame-mac: right128 of egress-mac.update(aes(mac-secret,egress-mac) ^ right128(egress-mac.update(frame-ciphertext).digest))

Egress-mac: h256, continuously updated with egress-bytes*
Ingress-mac: h256, continuously updated with ingress-bytes*


I am not very familiar with the encryption and decryption algorithm, so the analysis here is not very thorough. For the time being, I just analyzed the rough process. There are still many details that have not been confirmed.</pre></body></html>