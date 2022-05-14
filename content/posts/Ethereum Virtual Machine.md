+++ 
author = "Bouke Stam" 
title = "Ethereum Virtual Machine" 
date = "2022-02-14" 
description = "What is the EVM and how does it work?" 
tags = [ "ethereum", ] 
+++



### Intrinsic gas

Every transaction has an up-front gas fee, also called intrinsic gas. This amount needs to be paid upfront and is calculated like this:

- 21000 by default
- for each byte in the data, 4 if it's zero, 16 if it's non-zero
- 32000 if this transaction creates a contract (after Homestead update, block 1,150,000)

The Berlin update also introduced a new transaction type, that can specify the accounts and storage keys that the transaction plans to access. This introduces one more rule for the intrinsic gas cost:

- 2400 per account and 1900 per storage key to be accessed (only for transaction type 0x01)

### Substate

While executing a transaction, we need to keep track of several variables related to the output and effects of the transaction. The collection of these variables is called the substate and consists of:

- self destruct set
- log series
- touched accounts
- refund balance

Additionaly, in the Berlin update (block 12,244,000) there were some changes to account for loading in accounts and storage keys. See [EIP-2929](https://eips.ethereum.org/EIPS/eip-2929) and [EIP-2930](https://eips.ethereum.org/EIPS/eip-2930) for more details on this. Two new items were added to the substate to keep track of this:

- accessed account addresses
- accessed storage keys

### Machine state

- remaining gas
- program counter
- memory
- stack

### Storage


### Execution

The execution of the transactions starts with increasing the nonce by 1 and decreasing the balance by the up-front gas. 
This is irrevocable, even if the transaction fails. 

### Arithmetic

two's complement

### Contract creation

The address of the new account is calculated by combining the sender and the nonce. This account is then initialised by setting the nonce to 1, the balance to the value and the storage and code to empty. Then the transaction data is executed and the result of this execution is the contracts code, this is set in the account in an action called the code deposit. This also costs some more gas (200 for every byte in the code).