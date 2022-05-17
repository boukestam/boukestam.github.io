+++ 
author = "Bouke Stam" 
title = "RLPx Transport Protocol" 
date = "2022-01-19" 
description = "Explanation of the RLPx Transport Protocol" 
tags = [ "ethereum", ] 
+++

I think the easiest way to start is by just trying to connect to another node and seeing what data we get sent. We'll pick it up from there. 

Through the Ethereum documentation I manage to find [a document](https://github.com/ethereum/devp2p/blob/master/rlpx.md) that explains the process of connecting to another node. The protocol is called the **RLPx Transport protocol**. It's TCP-based and starts out a handshake. After the handshake we send and receive a hello message that communicates some information about the version and capabilities. A capability is for example the 'eth' protocol. This is the protocol that communicates new transactions or blocks for example. The flow looks like this:

1. send auth message
2. receive ack message
3. send hello message
4. receive hello message
5. check capabilities (for example ethereum version)
6. decide to keep or destroy connection based on capabilities

The first step in the handshake is to send an ```auth``` message. The ```auth``` message looks like this:

```
auth = auth-size || enc-auth-body
auth-size = size of enc-auth-body, encoded as a big-endian 16-bit integer
auth-vsn = 4
auth-body = [sig, initiator-pubk, initiator-nonce, auth-vsn, ...]
enc-auth-body = ecies.encrypt(recipient-pubk, auth-body || auth-padding, auth-size)
auth-padding = arbitrary data
```

Apparently the ```auth``` message consists of the size of the message and then an encrypted body, got that part. The size is a big-endian 16-bit integer. The body consists of the following parts:

- signature (ECDSA signature)
- public key (our public key, 64 bytes long)
- nonce (random 32 bytes)
- version number (needs to be 4)

Apparently the signature is a signed message using standard ECDSA with P256 Curve. It's calculated like this:

```
shared-secret = SSK(initiator-privkey, receiver-pubkey)
signature := Sign(ecdhe-random-key, shared-secret ^ init_nonce) 

Where SSK(initiator-privkey, receiver-pubkey) is a symmetric shared secret key as given by ECDH.
```

The nonce is just a random 32 byte sequence that we can decide ourselves. The ECIES encryption is new to me, so let's look that up.

### ECIES

A quick google search tells us that ECIES stands for Elliptic Curve Integrated Encryption Scheme. I tried multiple libraries, but unfortunately none of them give the correct output. I tried writing it by myself but there were multiple things that were just not documented well and almost impossible to find out by myself. I then found the [ethereumjs-devp2p repository](https://github.com/ethereumjs/ethereumjs-devp2p/blob/master/src/rlpx/ecies.ts) that has a nice implementation that I could slightly modify and use in my project. The implementation is too big to put in this post, but if you're interested you can find it [here](https://github.com/boukestam/eth-node/blob/main/ecies.ts). 

### Auth message

I started of by creating a new type of peer class called 'RLPxPeer'. In this constructor I initialize the ECIES class we just added:

```typescript
this.eceis = new ECIES(privateKey, initiatorEndpoint.id, receiverEndpoint.id);
```

Then we can create a new TCP socket and connect it to our node's TCP port. When it's connected we can create and send the auth message:

```typescript
this.socket = new net.Socket();

this.socket.connect(receiverEndpoint.tcpPort, receiverEndpoint.ip, () => {
  const auth = this.eceis.createAuthEIP8();
  this.socket.write(auth);
});
```

### Ack message

After sending the auth message, this is what we get back:

```
Data <Buffer 01 df 04 b6 07 65 53 85 47 c4 ee 0e 8f 20 16 72 ff 2c c6 48 2b b4 aa 33 65 42 28 23 e0 99 d6 10 61 60 58 df 38 bf 9b 64 d6 f6 f7 43 5e 6c cf 62 91 a4 ... 431 more bytes>
Data <Buffer dc e6 43 88 10 e3 5e 1c 77 92 95 55 f8 5c 04 a7 2f cf 70 23 5d dd 55 80 2f e2 a7 10 28 08 3d 0a 42 12 59 a6 ed 95 c7 c4 3d 14 f7 1e d6 b5 25 1e b7 e8 ... 158 more bytes>
```

According to the documentation, data is streamed to us. That means that we can't just see every piece of data coming in as being it's own message. We have to store and combine all incoming data in a buffer and then parse that buffer and extract all messages. Every time we get some data from the socket we append it to a buffer:

```typescript
this.buffer = Buffer.concat([this.buffer, data]);
```

Then we loop through the buffer and try to parse it:

```typescript
while (this.buffer.length > 0) {
  const size = this.parse();
  if (size === 0) break;
  
  this.buffer = this.buffer.slice(size);
}
```

In the parse function we check if we are waiting for the ack message. If we are, then we parse it and send a hello message:

```typescript
if (this.state === 'auth') {
  const ackSizeBuffer = this.buffer.slice(0, 2);
  const ackSize = bufferToInt(ackSizeBuffer);

  this.eceis.parseAckEIP8(this.buffer.slice(0, ackSize + 2));

  this.state = 'header';
  process.nextTick(() => this.sendHello());

  return ackSize + 2;
}
```

### Hello message

After the authentication handshake every message is encoded using a frame:

```
frame = header-ciphertext || header-mac || frame-ciphertext || frame-mac
header-ciphertext = aes(aes-secret, header)
header = frame-size || header-data || header-padding
header-data = [capability-id, context-id]
capability-id = integer, always zero
context-id = integer, always zero
header-padding = zero-fill header to 16-byte boundary
frame-ciphertext = aes(aes-secret, frame-data || frame-padding)
frame-padding = zero-fill frame-data to 16-byte boundary

// hello message
frame-data = msg-id || msg-data
frame-size = length of frame-data, encoded as a 24bit big-endian integer

// all other messages
frame-data = msg-id || snappyCompress(msg-data)
frame-size = length of frame-data encoded as a 24bit big-endian integer
```

Thankfully this handled by the ECIES implementation, because this is very complicated. A message is described by the message code, which indicates the kind of message, and the data. To send a message we just create and send the header and the body:

```typescript
send (code: number, data: Buffer, compress: boolean) {
  if (this.closed) return console.error('Socket already closed');

  const msg = Buffer.concat([rlp.encode(code), compress ? compressSync(data) : data]);

  const header = this.eceis.createHeader(msg.length);
  this.socket.write(header);

  const body = this.eceis.createBody(msg);
  this.socket.write(body);
}
```

As you can see, we also compress the data based on a parameter, the compression uses the [snappy library](https://www.npmjs.com/package/snappy). We can use this function to send a hello message. The hello message should have the following components:

- `protocolVersion` the version of the "p2p" capability, **5**.
- `clientId` Specifies the client software identity, as a human-readable string (e.g. "Ethereum(++)/1.0.0").
- `capabilities` is the list of supported capabilities and their versions: `[[cap1, capVersion1], [cap2, capVersion2], ...]`.
- `listenPort` specifies the port that the client is listening on (on the interface that the present connection traverses). If 0 it indicates the client is not listening.
- `nodeId` is the secp256k1 public key corresponding to the node's private key.

Using this we can our hello message like this:

```typescript
sendHello () {
  this.send(0x00, rlp.encode([
    intToBuffer(5),
    Buffer.from('eth-node/v0.1', 'ascii'),
    [
      [Buffer.from('eth', 'ascii'), intToBuffer(66)]
    ],
    intToBuffer(this.peer.initiatorEndpoint.tcpPort),
    this.peer.initiatorEndpoint.id
  ]), false);
}
```

As you can see, we call our node the 'eth-node/v.0.1'. And we only support the latest version of the ethereum protocol (version 66). I then also wrote some code to handle the parsing of the response hello message. When we see that the peer doesn't support ```eth 66```, we send a disconnect message.

A lot of nodes don't respond or respond with a ```Too many peers``` disconnect message. But after a few attempts we finally make a connection and immediately get a lot of messages:

```
Unhandled code 0x13
Unhandled code 0x18
Unhandled code 0x18
```

These messages are not from the eth protocol, and we will handle them in the next post.