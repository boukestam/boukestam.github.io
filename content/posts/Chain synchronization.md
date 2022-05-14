+++ 
author = "Bouke Stam" 
title = "Chain synchronization" 
date = "2022-02-02" 
description = "Explanation of the synchronization process to download the blockchain" 
tags = [ "ethereum", ] 
+++

Chain synchronization is the process of downloading blocks from other peers in the network.
After the status message is sent, each node checks how long the other's chain is.
If one node has a shorter chain than the other, it starts downloading the missing blocks.

### What is a block?

An ethereum block is basically a container that holds a bundle of transactions that is to be executed.
The number of transactions is limited by the block gas limit. Which is currently around 30,000,000.

Blocks are mined by miners by doing a very difficult calculation. The miner that solves this the first can claim the block.
The difficulty of the calculation increases or decreases based on the amount of mining power.
This makes sure that blocks will not be mined too quickly or too slowly. Ethereum has a target block time of 13 seconds, this means that on average, every 13 seconds a new block is created. The difficulty is adjusted to maintain this.

When a block is mined, the miner get's a reward, this is called the block reward and is currently around 2 ETH.
The address of the miner that the reward is 'sent to' is called the 'coinbase'.
When someone mines the same block, but is just too late and is no longer the first person anymore, this block will be added as an 'uncle block'. The uncles also receive a bit of reward. This makes sure that mining is not a winner takes all game, where the biggest miner will quickly become the fastest and keep snowballing this keep being the first one to mine the block.

- parent hash
- ommers
- coinbase
- number
- difficulty
- total difficulty (sum of all blocks in chain)
- gas limit
- gas used
- time (UTC)
- logs bloom
- proof of work

The exact definition can be found [in the documentation](https://github.com/ethereum/devp2p/blob/master/caps/eth.md#block-encoding-and-validity).

### Synchronization process

The synchronization process works like this:

1. download block header
2. verify proof of work
3. download block body (transactions)
4. execute transactions

The first block in the chain is called the genesis block. It's block number is 0.
The genesis block is hardcoded and contains 8893 ETH transactions to wallets that participated in the presale.
The presale was an event held in 2014 to raise funds for the development of the Ethereum network.

The chain synchronization therefore starts at block number 1, since we already defined the genesis block.

### Downloading blocks

We can request block headers from other peers in the network.
To do this, we have to send a message with code `0x03` and specify the block number to start at and the number of headers we want.
The response to this message always contains a maximum of 2048 headers.

```typescript
getBlockHeaders (peer: RLPxPeer, start: number, count: number): Promise<Buffer[][]> {
  return this.request(peer, 0x03, [
    intToBuffer(start),
    intToBuffer(count),
    intToBuffer(0),
    intToBuffer(0)
  ]);
}
```

Once we have the headers, we can send another request to get the block bodies, these contain the transactions and receipts.
This requests has the code `0x05` and needs the hashes of the blocks that we want to get.
The hashes are a keccak256 hash of the rlp encoded header data.
The result of this message has a maximum size of 2MB.

```typescript
getBlockBodies (peer: RLPxPeer, hashes: Buffer[]): Promise<any[]> {
  return this.request(peer, 0x05, hashes);
}
```

With these two methods we can now simply do the following loop to download all the blocks:

1. set blockNumber to 1
2. download 2048 block headers starting from blockNumber
3. download the corresponding block bodies (possibly in multiple requests)
4. increase the blockNumber by the amount of downloaded blocks
5. go back to step 2

### Next steps

Now that we know how to download all the blocks, we need to process them as follows:

- verify header validity (parent, difficulty, gas limit, time)
- verify the proof of work
- verify transactions merkle root
- store the block
- execute the block, and modify the state trie accordingly

This requires a lot of additional knowledge, which is covered in the following posts:

- [Chain storage](Chain%20storage.md)
- [Merkle Patricia Tree](Merkle%20Patricia%20Trie.md)
- [Ethash PoW verification](Ethash%20PoW%20verification.md)
- [Ethereum Virtual Machine](Ethereum%20Virtual%20Machine.md)