+++ 
author = "Bouke Stam" 
title = "Node Discovery Protocol" 
date = "2022-01-05" 
description = "How the Ethereum clients discover other nodes" 
tags = [ "ethereum", ] 
+++

The first step is to find some nodes that we can connect to. This process is defined as the [Node Discovery Protocol](https://github.com/ethereum/devp2p/blob/master/discv4.md). The process goes like this:

1. Connect to a node with UDP
2. Send ping
3. Receive pong
4. Ask for nodes close to me
5. Receive neighbor nodes
6. Go back to step 1 for all neighbors until we have enough nodes

To start this process we need to know at least one node that we can connect with. These starting nodes are called ```bootstrap nodes```. There is a list of them in the [go-ethereum repository](https://github.com/ethereum/go-ethereum/blob/master/params/bootnodes.go). The address for a node in that repo looks this:

```
"enode://d860a01f9722d78051619d1e2351aba3f43f943f6f00718d1b9baa4101932a1f5011f16bb2b1bb35db20d6fe28fa0bf09636d26a87d31de9ec6203eeedb1f666@18.138.108.67:30303"
```

The long hex string is the ID of the node. The part after the @ symbol contains the ip address and port. More information about the enode url format can be found [here](https://eth.wiki/en/fundamentals/enode-url-format). According to the Discovery Protocol documentation all nodes have these 4 attributes:

```
[ip, udp-port, tcp-port, node-id]
```

So let's create an interface for that:

```typescript
interface Endpoint {
  id: string;
  ip: string;
  udpPort: number;
  tcpPort: number;
}
```

Let's also write a function to parse the url into this format:

```typescript
function parseEnode (url: string): Endpoint {
  const [id, endpoint] = url.replace('enode://', '').split('@');
  const [ip, port] = endpoint.split(':');
  return {
    id: id,
    ip: ip,
    udpPort: parseInt(port),
    tcpPort: parseInt(port)
  };
}
```

### Sending a ping message

All messages that we want to send have to be wrapped according to the [Wire Protocol](https://github.com/ethereum/devp2p/blob/master/discv4.md#wire-protocol):

```
packet = packet-header || packet-data
packet-header = hash || signature || packet-type
hash = keccak256(signature || packet-type || packet-data)
signature = sign(packet-type || packet-data)
```

Looking at this, there are two things we need to look up: keccak256 and the sign function. Keccak256 is the hashing function used by Ethereum, it's in the same family as SHA hashing algorithms. We can put any amount of data in, and out comes a unique byte sequence. After some searching through the documentation, the signature turns out to be a ECDSA signature using a secp256k1 curve. This is all starting to look way too complicated, so this is where I need to bring in some libaries.

Even though my goal is to use as few libraries as possible, at this point I'm getting into math that is so difficult that it would probably kill the whole project right here if I try to take it on. At some point I might come back to this to try and figure it out for myself, but for now we just use a library to keep going.

I added the [secp256k1](https://www.npmjs.com/package/secp256k1) and [keccak256](https://www.npmjs.com/package/keccak256) packages to the project and this is the function I wrote:

```typescript
function encodePacket (privateKey: Buffer, type: number, packetData: Buffer): Buffer {
  const packetType = Buffer.from([type]);

  const sig = secp256k1.ecdsaSign(
    keccak256(Buffer.concat([packetType, packetData])), 
    privateKey
  );
  const signature = Buffer.concat([ sig.signature, Buffer.from([sig.recid])]);

  const hash = keccak256(signature, packetType, packetData);
  const packetHeader = Buffer.concat([hash, signature, packetType]);

  return Buffer.concat([packetHeader, packetData]);
}
```

Now that we have that ready, let's take look at the ping packet:

```
packet-data = [version, from, to, expiration, enr-seq ...]
version = 4
from = [sender-ip, sender-udp-port, sender-tcp-port]
to = [recipient-ip, recipient-udp-port, 0]
```

At first I was confused by the square brackets [], but I quickly found out that they refer to RLP encoding. I went head first into this, and it's not too difficult, but enough to put it in a [separate article](RLP Encoding.md). To result of this is that we now have two more functions: rlpEncode and rlpDecode. We also need a function to encode the endpoint as something that we can put in the rlpEncoder:

```typescript
function encodeEndpoint (endpoint: Endpoint): Buffer[] {
  return [
    ip.toBuffer(endpoint.ip),
    Buffer.from([endpoint.udpPort]),
    Buffer.from([endpoint.tcpPort])
  ];
}
```

Then we can create the expiration timestamp and RLP encode the version number, endpoints and timestamp to create our ping message.

```typescript
const expiration = Buffer.allocUnsafe(4);
expiration.writeUInt32BE(Date.now() / 1000 + 60); // Timeout after 60 seconds

const pingData = rlpEncode([
  Buffer.from([0x04]),
  encodeEndpoint(initiatorEndpoint),
  encodeEndpoint(receiverEndpoint), 
  expiration
]);

const ping = encodePacket(initiatorPrivate, 0x01, pingData);
```

We can then send the ping message to the node using a UDP socket:

```typescript
const client = udp.createSocket('udp4');

client.on('message', function(msg, info) {
  printBytes('Data from server', msg);
});

client.send(ping, receiverEndpoint.udpPort, receiverEndpoint.ip, function(error) {
  if(error) client.close();
  else console.log('Ping sent');
});
```

When we now run the program, this is what we get:

```
Ping sent
Data from server 150 <Buffer 01 2b 50 04 6e df d2 3b bc 37 f2 dc 69 39 75 0b 1e f7 fa 76 4a 80 80 ab 77 69 85 eb b8 e8 95 cf 60 6d 99 90 de 0c 5e c9 23 59 d0 a0 18 de 83 a2 63 15 ... 100 more bytes>
Data from server 130 <Buffer 3c fc 00 b7 4f e6 9f b7 d8 a2 88 24 5a 1d 53 be 6f 16 d1 f7 fa 1f 5d ab e1 1c 67 1b 5e 13 1d 6d 91 e6 95 f2 a8 4d a6 ce 9b 86 47 be 8e f3 61 32 a1 7c ... 80 more bytes>
```

### Receiving messages

After our ping message, the node sends two messages back to us. Since we haven't made any way to decode messages, we have no idea what they are... Presumably one of them is the pong packet that we want to receive. So let's build a function to decode messages.

```typescript
function decodePacket (data: Buffer) {
  const hash = keccak256(data.slice(32));
  const givenHash = data.slice(0, 32);
  if (!hash.equals(givenHash)) throw new Error('Invalid hash');

  const typeAndData = data.slice(97);
  const packetType = typeAndData[0];
  const packetData = typeAndData.slice(1);

  const sighash = keccak256(typeAndData);
  const signature = data.slice(32, 96);
  const recoverId = data[96];
  const publicKey = secp256k1.ecdsaRecover(signature, recoverId, sighash);

  return {packetType, packetData, hash, publicKey};
}
```

If we decode the messages and print out the type we get this:

```
Ping sent
Package type 2 // pong
Package type 1 // ping
```

Apparently the node first replies with a pong message and then also sends us a ping. Looking at the documentation this makes sense:

> If no communication with the sender has occurred within the last 12h, a ping should be sent in addition to pong in order to receive an endpoint proof.

### Sending a pong message

We need to respond to the ping message in order to be verified by the node and be able to get the neighbors. The pong message is very simple:

```
packet-data = [to, ping-hash, expiration, enr-seq, ...]
```

The ```ping-hash``` is the hash of the ping message we are responding to. By checking the packet type we can respond to the ping messages with a pong message:

```typescript
if (packet.packetType === 0x01) {
  // Ping

  await send(
    encodePacket(initiatorPrivate, 0x02, rlpEncode([
      encodeEndpoint(receiverEndpoint),
      packet.hash,
      encodeExpiration()
    ]))
  );
}
```

### Asking for neighbor nodes

As you can see I also wrote a helper method to make sending the message much easier. After the pong message, we can send the FindNode message:

```
packet-data = [target, expiration, ...]
```

In the documentation it says that the target is supposed to be the 65-byte secp256k1 public key. But after getting no response and debugging for an hour, I finally found out it's supposed to be the 64-byte ID (the public key without 0x04). So this is the final version:

```typescript
await send(
  encodePacket(initiatorPrivate, 0x03, rlpEncode([
    initiatorId,
    encodeExpiration()
  ]))
);
```

We can get the response like this:

``` typescript
if (packet.packetType === 0x04) {
  // Neighbors

  const [nodes, expiration] = data as [Buffer[][], Buffer];

  for (const endpoint of nodes) {
    console.log(`IP ${endpoint[0].join('.')} UDP ${bufferToNumber(endpoint[1])} TCP ${bufferToNumber(endpoint[2])}`);
  }
}
```

The output is this:

```
IP 13.229.120.227 UDP 30311 TCP 30311
IP 162.55.93.53 UDP 32631 TCP 30303
IP 136.243.49.136 UDP 30303 TCP 30303
IP 54.199.181.188 UDP 30303 TCP 30303
IP 108.52.155.208 UDP 30303 TCP 30303
IP 65.108.7.112 UDP 30303 TCP 30303
IP 45.32.37.192 UDP 30303 TCP 30303
IP 35.181.104.81 UDP 30303 TCP 30303
IP 116.203.189.155 UDP 30303 TCP 30303
IP 65.21.89.101 UDP 21219 TCP 21219
IP 161.97.151.215 UDP 28657 TCP 28657
IP 217.79.250.90 UDP 30311 TCP 30311
IP 3.250.36.7 UDP 30311 TCP 30311
IP 135.181.205.56 UDP 30303 TCP 30303
IP 212.102.36.241 UDP 5050 TCP 5050
IP 144.91.112.41 UDP 30303 TCP 30303
```

We can now use these nodes to continue looking for more nodes until we have enough. But what is enough and how will we store these nodes? These questions will be answered in the next part in the series.

### Improvements

- Periodically clean up unresponsive nodes