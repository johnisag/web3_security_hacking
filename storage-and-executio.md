# Storage and Executio

We have been writing smart contracts over the last few tracks, and briefly mentioned that Ethereum smart contracts run within this thing called the Ethereum Virtual Machine (EVM).

We also briefly mentioned in passing that EVM is capable of running certain OPCODES, and deals with data present either in the stack or heap. If you have a formal computer science background, that may have made sense to you, but for everyone else, what does this actually mean? 🤔

In this level, we will dig deeper into the EVM execution engine and how data is stored, manipulated, and ran throughout the course of a transaction.

### 🧠 Recap

Let's recap a few things we taught in earlier tracks before moving ahead.

Recall that Ethereum works as a transaction-based state machine. Starting at some state `s1`, a transaction manipulates certain data to shift the world state to some state `s2`.

<figure><img src=".gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

To group things together, transactions are packed together in blocks. Generally speaking, each block changes the world state from state `s1` to `s2`, and the conversion is calculated based on the state changes made by every transaction within the block.

When we think of these state changes, Ethereum can be thought of as a state chain.

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

But, what is this world state? 🤨

### 🗺️ World State

The World State in Ethereum is a mapping between addresses and account states. Each address on Ethereum has it's own state, this could be a user account (EOA) or a smart contract.

<figure><img src=".gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

Each block essentially manipulates multiple account states, thereby manipulating the overall world state of Ethereum.

### 📒 Account State

Alright, so the world state is comprised of various account states. What is an account state?

<figure><img src=".gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

The account state contains a few common things, like the nonce and the balance (in ETH). Additionally, smart contracts also contain a storage hash and a code hash. The two hashes act as references to a separate state tree, which store state variables and the bytecode of the smart contract respectively.

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

Recall that there are two types of accounts in Ethereum. Externally owned accounts (e.g. Coinbase Wallets, Metamask Wallets, etc.) and Smart Contract Accounts.

EOA's are controlled by private keys, and do not have any EVM code. Contract accounts on the other hand contain EVM code and are controlled by the code itself, and do not have private keys associated with them.

### 💵 Types of Transactions

There are two types of transactions on Ethereum mainly. Those which create new contracts, and those which just send messages.

Sending messages here implies making a transaction that either transfers ETH, or calls functions on a smart contract. They are just different types of messages that can be sent by an EOA.

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

When a contract creation transaction is made, a new account is added to the world state. The transaction carries with it the bytecode of the contract to be created and the initializing code (i.e. constructor calls).

<figure><img src=".gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

On the other hand, for all other transactions, i.e. message calls, the account state of an existing account is modified following the transaction.

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

### ✉️ Messages

Messages in Ethereum are passed between two accounts. They consist primarily of two things - `data` and `value`.

`data` is a set of bytes, that indicate the type of transaction that needs to take place (transfer ETH, mint an NFT, vote in a DAO, etc) and `value` is the Ether value that is transfered along with the transaction.

Transactions made by EOA's send a mesasge to the recipient account. Contract accounts can also send messages to accounts through the EVM code.



