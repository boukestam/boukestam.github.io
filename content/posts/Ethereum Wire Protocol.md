+++ 
author = "Bouke Stam" 
title = "Ethereum Wire Protocol" 
date = "2022-01-24" 
description = "Explanation of the Ethereum Wire Protocol" 
tags = [ "ethereum", ] 
+++

The [ethereum wire protocol](https://github.com/ethereum/devp2p/blob/master/caps/eth.md) is the build on top of the RLPx protocol. 
It's messages are offset by 0x10 (right after the RLPx builtin messages).
There are 3 main functionalities in the protocol:

- transaction exchange
- block propagation
- chain synchronization

We will start by focusing on the transaction exchange and block propagation.
These parts relay new transactions and blocks troughout the network.
By doing this first, we can already be a helpful node while we are synchronizing (downloading) the chain.

### Status message

The protocol starts off with both peers sending a status messages (0x00) to the other peer.
The status message contains:

- protocol version (latest is 66)
- chain id (e.g. 1 for mainnet, 4 for rinkeby testnet)
- total difficulty of our chain
- hash of latest block of our chain
- hash of the genesis block
- hash of the current fork
- blocknumber of the next fork

Since we don't have not downloaded any blocks yet, our current block is the [genesis block](https://etherscan.io/block/0).
The current fork at the time of writing is 'Arrow Glacier'.
The code to send the status message looks like this:

```typescript
this.send(peer, 0x00, rlp.encode([
  intToBuffer(66),  // protocol version
  intToBuffer(1),   // chain id
  intToBuffer(17179869184), // genesis total difficulty
  Buffer.from('d4e56740f876aef8c010b86a40d5f56745a118d0906a34e69aec8c0db1cb8fa3', 'hex'), // genesis hash
  Buffer.from('d4e56740f876aef8c010b86a40d5f56745a118d0906a34e69aec8c0db1cb8fa3', 'hex'), // genesis hash
  [
    Buffer.from('20c327fc', 'hex'), // arrowGlacier hash
    Buffer.alloc(0) // unknown block for merge fork
  ]
]));
```

When we receive their status message, we can determine if the other peer uses the same protocol version, chain id and fork.
If they don't, we immediately disconnect:

```typescript
if (code === 0x00) { // status
  const [version, networkId, totalDifficulty, blockHash, genesis, forkId] = body;
        
  if (bufferToInt(version) !== 66 || bufferToInt(networkId) !== 1) {
    peer.disconnect(0x10);
    return;
  }
}
```

If they do, we set their status to verified.

### Transaction pool

Every node is expected to keep track of a pool of pending transactions. 
Right after the status message is received, each node sends the hashes of it's pooled transactions to the other node.
Whenever new transaction hashes are received, we need to 'forward' them to all nodes that don't know about it.
Also, we need to retrieve the actual transaction information.
To know which nodes know about which transactions, we also keep a record of all exchanged transaction hashes for each node.

Let's start of by creating the data structures for keeping the pool of hashes and the hashes per peer:

```typescript
const transactionHashes = new Set<string>();
const hashesByPeer = new Map<string, Set<string>>();
```

Then we need to handle the incoming transaction hashes message by extracting the hashes and converting them to strings to use in the data structures:

```typescript
if (code === 0x08) { // new pooled transaction hashes
  const hashes = body as Buffer[];
  const hashStrings = body.map(buffer => buffer.toString('hex'));

  this.broadcast(peer, this.transactionHashes, hashStrings, hashes);
}
```

In the broadcast function we need to add the hash to the pool and then send the message to all peers that don't already know of this hash:

```typescript
broadcast (from: RLPxPeer, set: Set<string>, hashStrings: string[], bodies: any[]) {
  const peerPool = this.hashesByPeer.get(from.idString());

  for (const hash of hashStrings) {
    set.add(hash);
    peerPool.add(hash);
  }

  for (const broadcastPeer of this.peers) {
    if (broadcastPeer === from) continue;

    const broadcastPeerPool = this.hashesByPeer.get(broadcastPeer.idString());
    const broadcastHashes = bodies.filter((v, i) => broadcastPeerPool.has(hashStrings[i]));

    this.send(broadcastPeer, 0x08, rlp.encode(broadcastHashes));

    for (const hash of hashStrings) broadcastPeerPool.add(hash);
  }
}
```

We also need to send all pooled transactions whenever a new connection is made. 
According to the documentation, the maximum amount of hashes per message is 4096.
So we need to split the hashes into chunks and send the chunks one by one:

```typescript
const hashes = this.transactionHashes.values();
let chunk = [];

for (const hash of hashes) {
  chunk.push(Buffer.from(hash, 'hex'));

  if (chunk.length === 4096) {
    this.send(peer, 0x08, rlp.encode(chunk));
    chunk = [];
  }
}

if (chunk.length > 0) this.send(peer, 0x08, rlp.encode(chunk));
```

### Block propagation

Block propagation is basically the same as transaction exchange, but with blocks.
When a new block is found, it is propagated throughout the network using the 'New Block Hashes' message (0x01).
When we receive new blocks, we need to send it to all peers that don't know of this block yet.
We can reuse the broadcast method for this:

```typescript
if (code === 0x01) { // new block hashes
  const blocks = body as [Buffer, Buffer][];
  const hashStrings = blocks.map(([hash, _]) => hash.toString('hex'));

  this.broadcast(peer, this.blockHashes, hashStrings, blocks);
}
```

### Improvements

- Clean up or limit the transaction pool size so it doesn't get too big