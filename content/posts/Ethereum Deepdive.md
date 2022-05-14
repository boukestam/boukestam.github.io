+++ 
author = "Bouke Stam" 
title = "Ethereum Deepdive" 
date = "2022-01-04" 
description = "A deepdive into the inner workings of the Ethereum protocol" 
tags = [ "ethereum", ] 
+++

This is the first post in a series on writing an Ethereum node from scratch. I want to do this because I'm the kind of person that understands things by trying to recreate it for myself. For that reason I decided to finally tackle one of the biggest mysteries to me. 

I've been working as a blockchain developer for 1.5 years now, but honestly it's inner workings are still a big mystery to me. How do nodes talk to each other and collectively come to a consensus? How does all the cryptography actually work? What happens when I store a variable?

### The goal

The goal is to learn about all the different aspects that make the Ethereum network work. The goal is **not** to create the most optimized, stable Ethereum node. I will start out creating everything in the most simple way and only optimize and handle errors where needed. Throughout the project I will dive in head-first and try to tackle the hurdles as they come.

### The plan

With this project I try to answer all these questions, and hopefully help others who are looking for the answers. This is my plan:

- use Node.js/Typescript as programming language because I am familiar with it
- try to use as few libraries as possible
- use the [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf) and [Ethereum documentation](https://github.com/ethereum) as reference
- start as simple as possible and build out from there

Some of you might question the decision to use NodeJS and Typescript for this project. But I think it will help me and others understand the code more easily since many people are familiar with it.

### Prior knowledge

I already have a vague high-level idea about how the blockchain works. This will definitely steer me in some directions so I think it's good to explain what I already know:

- the EVM (Ethereum Virtual Machine) is the processor at the core of Ethereum
- the EVM processes transactions that interact with smart contracts
- transactions can read or write from a special kind of memory called the 'state'
- transactions are grouped up in blocks
- blocks are chained together in a blockchain and depend on each other
- consensus about which chain is the correct one is reached through a network of nodes
- the nodes talk to each other to share information

I don't really have any idea how any of this works under the hood though. But looking at this we have basically two ways to start out, at the top of the list (the EVM) or the bottom (the network). I think it's best to start with the network so we can immediately start working with real data and real interactions.

### Setup

I start the project by [downloading Node.js](https://nodejs.org/en/download/). The installation is really quick and only takes a minute. After that I create a new directory and initialize my project:

```
npm install --global yarn // if you don't have it yet
yarn init
yarn add -D typescript
yarn add ts-node
yarn add -D tslib @types/node
```

I then create a new file called ```main.ts``` and put in the following code:

```typescript
console.log('Hello World!');
```

I can then run it with ```ts-node main.ts```. Now we can get started on the actual fun stuff!

### Chapters

1. [Public Key Cryptography](Public%20Key%20Cryptography.md)
2. [Node Discovery Protocol](Node%20Discovery%20Protocol.md)
3. [RLP Encoding](RLP%20Encoding.md)
4. [Kademlia Table](Kademlia%20Table.md)
5. [RLPx Transport Protocol](RLPx%20Transport%20Protocol.md)
6. [Ethereum Wire Protocol](Ethereum%20Wire%20Protocol.md)
7. [Chain synchronization](Chain%20synchronization.md)
8. [Chain storage](Chain%20storage.md)
9.  [Merkle Patricia Tree](Merkle%20Patricia%20Trie.md)
10. [Ethash PoW verification](Ethash%20PoW%20verification.md)
11. [Ethereum Virtual Machine](Ethereum%20Virtual%20Machine.md)
12. [JSON-RPC API](JSON-RPC%20API.md)
13. [Ethash Mining](Ethash%20Mining.md)