+++ 
author = "Bouke Stam" 
title = "Kademlia Table" 
date = "2022-01-14" 
description = "Sorting nodes in a Kademlia Table" 
tags = [ "ethereum", ] 
+++

In the previous part we found out how to connect to nodes, send ping and pong messages and finally get a list of neighbor nodes. Now we need to continue this process until we have enough neighbors. But how much is enough? After reading [the documentation](https://github.com/ethereum/devp2p/blob/master/discv4.md#kademlia-table) I found out that we need to use a ```Kademlia Table```.

The table is more like a tree. We start of by calculating the distance between two nodes like this:

```
distance(n₁, n₂) = keccak256(n₁) XOR keccak256(n₂)
```

This leaves us with a 256-bit number. We start of with the highest (left-most) bit. If this is a 1, we move to the left, if it's a zero, we move to the right. Then onto the next bit. We stop when we find a bucket with enough space in it. If there the bucket is full, but it can still be split further down, we do it. If we end up in a full bucket, we revalidate the nodes in there by sending a ping. If one of them doesn't respond, we replace it with the new one. We don't split further then one branch 'away' in the 1 direction. See [this presentation](https://docs.google.com/presentation/d/11qGZlPWu6vEAhA7p3qsQaQtWH7KofEC9dMeBFZ1gYeA/edit#slide=id.g1718cc2bc_0661) for a more visual explanation.

![Kademlia Table Visualisation](/images/kademlia.png)

Because our target is always a distance of 0, we don't ever need to split into the '1' direction. Therefore we don't need it to be a tree. We can just treat it like a flat array. Index 0 is the 1 branch of the first split, index 1 the second split, etc... Therefore we can do it like this:

- we have 256 buckets
- each bucket stores ```k = 16``` nodes
- bucket ```i``` stores nodes with bit ```i``` set
- we start filling from left to right
- when the current last bucket is full we split it
- if the end destination bucket is full we try to replace an inactive node

After this I made the most simple implementation I could come up with. The code is a bit too long, and hard to explain piece by piece, so you can see it [on github](https://github.com/boukestam/eth-node/blob/main/main.ts) if you are interested.

### Recursive lookup

The process of discovering nodes is called recursive lookup. We start off by getting neighbor nodes from the boot node and putting them in the table. Then we get the 3 closest nodes from the table and ask them for neighbors and put them in the table. When we have queried a node for neighbors, we mark it and don't ask it again. We keep doing this until we have queried all ```k = 16``` closest nodes.

I started of by making a class for a peer (connection). I made this class an EventEmitter so we can notify the parent controller. I put all previously written code that has to do with sending and receiving messages in here. I added a 'verified' event for when the peer receives a pong event. Then I changed the incoming neighbors handler to parse the endpoint and emit an event:

```typescript
if (packet.packetType === 0x04) { // Neighbors
  const [nodes, expiration] = data as [Buffer[][], Buffer];

  for (const node of nodes) {
    const endpoint: Endpoint = {
      id: node[3],
      ip: node[0].join('.'),
      udpPort: bufferToNumber(node[1]),
      tcpPort: bufferToNumber(node[2])
    };
    
    this.emit('neighbor', endpoint);
  }
}
```

Now we can create a server class to handle our socket and store the KademliaTable like this:

```typescript
class Server {
  socket: Socket;
  table: KademliaTable<Peer>;
}
```

We can then create a method in here to bootstrap the node finding process:

```typescript
async boot (endpoint: Endpoint) {
  const peer = new Peer(this.privateKey, this.endpoint, endpoint, this.socket);
  await this.addPeer(peer);
  
  peer.on('verified', () => {
    peer.findNode();
    peer.queried = true;
  });

  peer.ping();
}
```

When the peer receives the neighbors it will emit events for them. We can catch those events in the server and try to add the neighbors to our table. If it's added successfully, we can then send a ping message to start the verification process.

```typescript
peer.on('neighbor', (endpoint: Endpoint) => {
  if (this.table.exists(endpoint.id)) return;

  const neighbor = new Peer(this.privateKey, this.endpoint, endpoint, this.socket);
  this.addPeer(neighbor).then(added => {
    if (added) neighbor.ping();
  });
});
```

After the boot node, we need to start querying the neighbors for more nodes. According to the documentation we should only query the ```k = 16``` closest nodes, and also only once. So we can create a function that gets the closest nodes and then filters out the one's we already queried. Also we only want to do ```concurrent = 3``` number of queries at the same time:

```typescript
async findNodes (concurrent: number) {
  const closest = this.table.closest(16, (peer) => peer.verified);
  const notQueried = closest.filter(peer => !peer.queried)

  for (const peer of notQueried.slice(0, concurrent)) {
    peer.findNode();
    peer.queried = true;
  }
}
```

Finally we can create our server, boot it using the boot endpoint, and then check for new nodes every 2 seconds:

```typescript
const server = new Server(privateKey, endpoint);
server.boot(bootEndpoint);

setInterval(() => {
  server.findNodes(3);
}, 2000);
```

I ran this process multiple times and always ended up with around 80 connections. Now that we have a nicely populated list of nodes, we can move on to the next step and try to get some useful information out of them!
