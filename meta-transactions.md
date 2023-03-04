---
description: 🤨 Metatransactions and Signature Replay
---

# Meta transactions

There are times when you want your dApp users to have a gas-less experience, or perhaps make a transaction without actually putting something on the chain. These types of transactions are called meta-transactions, and in this level, we will dive deep into how to design meta-transactions and also how they can be exploited if not designed carefully.

For those of you who have used OpenSea, ever noticed how OpenSea lets you make listings of your NFT for free? No matter what price you want to sell your NFT at, somehow it never charges gas beyond the initial NFT Approval transaction? The answer, Meta transactions.

Meta transactions are also commonly used for gas-less transaction experiences, for example asking the users to sign a message to claim an NFT, instead of paying gas to send a transaction to claim an NFT.

There are other use cases as well, for example letting users pay for gas fees with any token, or even fiat without needing to convert to crypto. Multisig wallets also only ask the last signer to pay gas for the transaction being made from the multisig wallet, and the other users just sign some messages.

### 👀 How does it work?

**Meta transactions are just a fancy word for transactions where a third party (a relayer) pays the gas for the transaction on behalf of the user.**&#x20;

The user just needs to sign messages (and not send a transaction) that contain information about the transaction they want to execute and hand it to the relayer. Relayers are then responsible for creating valid transactions using this data and paying for gas themselves.

**Relayers** could be, for example, code in your dApp to let your users experience gas-less transactions, or it could be a third-party company that you pay in fiat to execute transactions on Ethereum, etc.

At this level, we will do two things - learn about meta transactions and how to build a simple smart contract that supports meta transactions, and also learn about a security bug in the first contract we build and how to fix it. Kill two birds with one stone 😎.

### 🔥 Digital Signatures

In the previous IPFS levels, we have explained what hashing is. Before moving on, we need to understand a second fundamental concept of cryptography - digital signatures.

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

<mark style="color:purple;">**A digital signature is a mathematical way to verify the authenticity of a message.**</mark>** Given some data, a private key can be used to sign that data and produce a signed message.**

Then, someone else who believes they have the original input data can compare it to the signed message to retrieve the public key which signed the message. Since the public key of the sender must have been known beforehand, they can be assured that the data has not been manipulated and its integrity is upheld.

So in the example above, Alice signs the message `Hello Bob!` using her private key and sends it to Bob. Bob already knows that the content is `Hello Bob!` and now wants to verify that the message was indeed sent from Alice, so he tries to retrieve the public key from the signed message. If the public key is Alice's public key then the message was not tampered with and was securely transmitted.

**Use Cases of Digital Signatures**

Digital signatures are commonly used when wanting to communicate securely and ensuring that the contents of the messages were not manipulated during communication.&#x20;

You can send the original plaintext message, along with a signed message from a known sender, and the receiver can verify the integrity of the message by deriving the public key of the signer by comparing the plaintext message and the signed message.&#x20;

If the public key of the signer matches the public key of the expected sender, the message was transmitted securely!

**Digital Signatures on Ethereum**

In the case of Ethereum, user wallets can sign messages, and then the original input data can be verified with the signed message to retrieve the address of the user who signed the message. Signature verification can be done both off-chain as well as within Solidity. Here in this case the address of the user is the public key.

### 🧠 Understanding the concepts

Let's say we want to build a dApp where users can transfer tokens to other addresses without needing to pay gas, except just once for approval. Since the user themself will not be sending the transaction, we will be covering the gas for them 🤑.

This design is quite similar to how OpenSea works, where once you pay gas to provide approval over an ERC-721 or ERC-1155 collection, listings and sales on OpenSea are free for the seller. OpenSea only asks the sellers to sign a transaction when putting up a listing, and when a buyer comes along, the seller's signature is submitted alongside the buyer's transaction thereby transferring the NFT to the buyer and transferring the ETH to the seller - and the buyer pays all the gas for this to happen.

In our case, we will just have a relayer letting you transfer ERC-20 tokens to other addresses after initial approval 🎉.

Let's think about what all information is needed to make this happen. The smart contract needs to know everything about the token transfer, so we at least need to incorporate 4 things into the signed message:

* `sender` address
* `recipient` address
* `amount` of tokens to transfer
* `tokenContract` address of the ERC-20 smart contract

The flow will look something like this:

* User first approves the `TokenSender` contract for infinite token transfers (using ERC20 `approve` function)
* User signs a message containing the above information
* Relayer calls the smart contract and passes along the signed message, and pays for gas
* Smart contract verifies the signature and decodes the message data, and transfers the tokens from sender to recipient

Thankfully, `ethers.js` comes with a function called `signMessage` which lets us sign messages easily.&#x20;

However, to avoid producing arbitrary length messages, we will first hash the required data and then sign it.&#x20;

In the smart contract as well, we will first hash the given arguments, and then try to verify the signature against the hash.

We will code this up as a Hardhat test, so first, we need to write the smart contract.

### ⚒️ Contract

Note All of these commands should work smoothly . If you are on windows and face Errors Like **`Cannot read properties of null (reading 'pickAlgorithm')`** Try Clearing the NPM cache using **`npm cache clear --force`**.

**Setup a Hardhat project**

```shell
mkdir meta-transactions
cd meta-transactions
npm init --yes
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# bootstrap the hardhat project
npx hardhat

# openzepelin deps
npm install @openzeppelin/contracts
```

Make sure you select **`Create a Javascript Project`**

We will create two smart contracts.&#x20;

* The first is a super simple ERC-20 implementation, and the second,&#x20;
* our `TokenSender` contract.&#x20;

For simplicity, we will create them both in the same `.sol` file.

Create a new file inside the **`meta-transactions/contracts`**directory and call it **`MetaTokenSender.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract RandomToken is ERC20 {
    constructor() ERC20("", "") {}

    function freeMint(uint256 amount) public {
        _mint(msg.sender, amount);
    }
}

contract TokenSender {
    using ECDSA for bytes32;

    function transfer(
        address sender,
        uint256 amount,
        address recipient,
        address tokenContract,
        bytes memory signature
    ) public {
        // Calculate the hash of all the requisite values
        bytes32 messageHash = getHash(sender, amount, recipient, tokenContract);
        // Convert it to a signed message hash
        bytes32 signedMessageHash = messageHash.toEthSignedMessageHash();

        // Extract the original signer address
        address signer = signedMessageHash.recover(signature);

        // Make sure signer is the person on whose behalf we're executing the transaction
        require(signer == sender, "Signature does not come from sender");

        // Transfer tokens from sender(signer) to recipient
        bool sent = ERC20(tokenContract).transferFrom(
            sender,
            recipient,
            amount
        );
        require(sent, "Transfer failed");
    }

    // Helper function to calculate the keccak256 hash
    function getHash(
        address sender,
        uint256 amount,
        address recipient,
        address tokenContract
    ) public pure returns (bytes32) {
        return
            keccak256(
                abi.encodePacked(sender, amount, recipient, tokenContract)
            );
    }
}
```

**``**
