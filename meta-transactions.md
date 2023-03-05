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

**The Imports**

The **`ERC20.sol`** import is to inherit the base ERC-20 contract implementation from OpenZeppelin, which we have learned a lot about in previous tracks.

**`ECDSA` ** stands for **`Elliptic Curve Digital Signature Algorithm`** - it is the signatures algorithm used by Ethereum, and the OpenZeppelin library for `ECDSA.sol` contains some helper functions used for digital signatures in Solidity.

**The Functions**

The ERC-20 contract is quite self-explanatory, as all it does is let you mint an arbitrary amount of free tokens.

For `TokenSender`, there are two functions here. Let's first look at the helper function - `getHash` - which takes in the `sender` address, `amount` of tokens, `recipient` address, and `tokenContract` address, and returns the `keccak256` hash of them packed together. **`abi.encodePacked` converts all the specified values in bytes leaving no padding** in between and passes it to **keccak256 which is a hashing function used by Ethereum**. This is a `pure` function so we will also be using this client-side through Javascript to avoid dealing with keccak hashing and packed encodes in Javascript which can be a bit annoying.

The `transfer` function is the interesting one, which takes in the above four parameters, and a signature. It calculates the hash using the `getHash` helper. After that **the message hash is converted to a `Ethereum Signed Message Hash` according to the** [**EIP-191**](https://eips.ethereum.org/EIPS/eip-191) **.** Calling this function converts the `messageHash` into this format `"\x19Ethereum Signed Message:\n" + len(message) + message)`. It is important to abide by the standard for interoperability.

After doing so you call `recover` method in which you pass the `signature` which is nothing but your `Ethereum Signed Message` signed with the sender's private key, you compare it with the `Ethereum Signed Message` - `signedMessageHash` you generated to recover the public key which should be the address of the sender.

If the signer address is the same as the `sender` address that was passed in, then we transfer the ERC-20 tokens from `sender` to `recipient`.

#### Writing the Test

So lets try to do that, In your `test` directory create a new file named **`metatxn-test.js`** and lets have some fun with code

```javascript
const { expect } = require("chai");
const { BigNumber } = require("ethers");
const { arrayify, parseEther } = require("ethers/lib/utils");
const { ethers } = require("hardhat");

describe("MetaTokenTransfer", function () {
  it("Should let user transfer tokens through a relayer", async function () {
    // Deploy the contracts
    const RandomTokenFactory = await ethers.getContractFactory("RandomToken");
    const randomTokenContract = await RandomTokenFactory.deploy();
    await randomTokenContract.deployed();

    const MetaTokenSenderFactory = await ethers.getContractFactory(
      "TokenSender"
    );
    const tokenSenderContract = await MetaTokenSenderFactory.deploy();
    await tokenSenderContract.deployed();

    // Get three addresses, treat one as the user address
    // one as the relayer address, and one as a recipient address
    const [_, userAddress, relayerAddress, recipientAddress] =
      await ethers.getSigners();

    // Mint 10,000 tokens to user address (for testing)
    const tenThousandTokensWithDecimals = parseEther("10000");
    const userTokenContractInstance = randomTokenContract.connect(userAddress);
    const mintTxn = await userTokenContractInstance.freeMint(
      tenThousandTokensWithDecimals
    );
    await mintTxn.wait();

    // Have user infinite approve the token sender contract for transferring 'RandomToken'
    const approveTxn = await userTokenContractInstance.approve(
      tokenSenderContract.address,
      BigNumber.from(
        // This is uint256's max value (2^256 - 1) in hex
        // Fun Fact: There are 64 f's in here.
        // In hexadecimal, each digit can represent 4 bits
        // f is the largest digit in hexadecimal (1111 in binary)
        // 4 + 4 = 8 i.e. two hex digits = 1 byte
        // 64 digits = 32 bytes
        // 32 bytes = 256 bits = uint256
        "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"
      )
    );
    await approveTxn.wait();

    // Have user sign message to transfer 10 tokens to recipient
    const transferAmountOfTokens = parseEther("10");
    const messageHash = await tokenSenderContract.getHash(
      userAddress.address,
      transferAmountOfTokens,
      recipientAddress.address,
      randomTokenContract.address
    );
    const signature = await userAddress.signMessage(arrayify(messageHash));

    // Have the relayer execute the transaction on behalf of the user
    const relayerSenderContractInstance =
      tokenSenderContract.connect(relayerAddress);
    const metaTxn = await relayerSenderContractInstance.transfer(
      userAddress.address,
      transferAmountOfTokens,
      recipientAddress.address,
      randomTokenContract.address,
      signature
    );
    await metaTxn.wait();

    // Check the user's balance decreased, and recipient got 10 tokens
    const userBalance = await randomTokenContract.balanceOf(
      userAddress.address
    );
    const recipientBalance = await randomTokenContract.balanceOf(
      recipientAddress.address
    );

    expect(userBalance.lt(tenThousandTokensWithDecimals)).to.be.true;
    expect(recipientBalance.gt(BigNumber.from(0))).to.be.true;
  });
});

```

#### Running the Test

To run this test, open up your terminal pointing to the root of the directory for this level and execute this command:

```sh
npx hardhat test
```

### 🔓 Security Vulnerability

Can you guess what the problem is with the code we just wrote though? 🤔

Since the signature contains the information necessary, the relayer could keep sending the signature to the contract over and over, thereby continuously transferring tokens out of the `sender`'s account into the `recipient`'s account.

While this may not seem like a big deal in this specific example, what if this contract was responsible for dealing with money on mainnet? If the same signature could be reused over and over, the user would lose all their tokens!

Instead, the transaction should only be executed when the user explicitly provides a second signature (while staying within the rules of the smart contract, of course).

This attack is called **Signature Replay** - because, well you guessed it, you're replaying a signature.

### 🌟 Solving for Signature Replay

For even simpler contracts, you could resolve this by having some (nested) mappings in your contract. But there are 4 variables to keep track of here per transfer - `sender`, `amount`, `recipient`, and `tokenContract`. Creating a nested mapping this deep can be quite expensive in Solidity.

Also, that would be different for each 'kind' of a smart contract - as you're not always dealing with the same use case. A more general-purpose solution for this is to create a single mapping from the hash of the parameters to a boolean value, where `true` indicates that this meta-transaction has already been executed, and `false` indicates it hasn't.

Something like `mapping(bytes32 => bool)`.

This also has a problem though. With the current set of parameters, if Alice sent 10 tokens to Bob, it would go through the first time, and the mapping would be updated to reflect that. However, what if Alice genuinely wants to send 10 more tokens to Bob a second time?

Since digital signatures are deterministic, i.e. the same input will give the same output for the same set of keys, that means Alice would never be able to send Bob 10 tokens again!

To avoid this, we introduce a fifth parameter, the `nonce`.

The `nonce` is just a random number value, and can be selected by the user, the contract, be randomly generated, it doesn't matter - as long as the user's signature includes that nonce. Since the exact same transaction but with a different nonce would produce a different signature, the above problem is solved!

#### Smart Contract Changes

We need to update the smart contract's code in three places.

* **First**, we will add a `mapping(bytes32 => bool)` to keep track of which signatures have already been executed.
* **Second**, we will update our helper function `getHash` to take in a `nonce` parameter and include that in the hash.
* **Third**, we will update our `transfer` function to also take in the `nonce` and pass it onto `getHash` when verifying the signature. It will also update the mapping after the signature is verified.

**Here's the updated code:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract RandomToken is ERC20 {
    constructor() ERC20("", "") {}

    function freeMint(uint amount) public {
        _mint(msg.sender, amount);
    }
}

contract TokenSender {

    using ECDSA for bytes32;

    // New mapping
    mapping(bytes32 => bool) executed;

    // Add the nonce parameter here
    function transfer(address sender, uint amount, address recipient, address tokenContract, uint nonce, bytes memory signature) public {
        // Pass ahead the nonce
        bytes32 messageHash = getHash(sender, amount, recipient, tokenContract, nonce);
        bytes32 signedMessageHash = messageHash.toEthSignedMessageHash();

        // Require that this signature hasn't already been executed
        require(!executed[signedMessageHash], "Already executed!");

        address signer = signedMessageHash.recover(signature);

        require(signer == sender, "Signature does not come from sender");

        // Mark this signature as having been executed now
        executed[signedMessageHash] = true;
        bool sent = ERC20(tokenContract).transferFrom(sender, recipient, amount);
        require(sent, "Transfer failed");
    }

    // Add the nonce parameter here
    function getHash(address sender, uint amount, address recipient, address tokenContract, uint nonce) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(sender, amount, recipient, tokenContract, nonce));
    }
}
```

**Here are the updated tests:**

```javascript
const { expect } = require("chai");
const { BigNumber } = require("ethers");
const { arrayify, parseEther } = require("ethers/lib/utils");
const { ethers } = require("hardhat");

describe("MetaTokenTransfer", function () {
  it("Should let user transfer tokens through a relayer with different nonces", async function () {
    // Deploy the contracts
    const RandomTokenFactory = await ethers.getContractFactory("RandomToken");
    const randomTokenContract = await RandomTokenFactory.deploy();
    await randomTokenContract.deployed();

    const MetaTokenSenderFactory = await ethers.getContractFactory(
      "TokenSender"
    );
    const tokenSenderContract = await MetaTokenSenderFactory.deploy();
    await tokenSenderContract.deployed();

    // Get three addresses, treat one as the user address
    // one as the relayer address, and one as a recipient address
    const [_, userAddress, relayerAddress, recipientAddress] =
      await ethers.getSigners();

    // Mint 10,000 tokens to user address (for testing)
    const tenThousandTokensWithDecimals = parseEther("10000");
    const userTokenContractInstance = randomTokenContract.connect(userAddress);
    const mintTxn = await userTokenContractInstance.freeMint(
      tenThousandTokensWithDecimals
    );
    await mintTxn.wait();

    // Have user infinite approve the token sender contract for transferring 'RandomToken'
    const approveTxn = await userTokenContractInstance.approve(
      tokenSenderContract.address,
      BigNumber.from(
        // This is uint256's max value (2^256 - 1) in hex
        "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"
      )
    );
    await approveTxn.wait();

    // Have user sign message to transfer 10 tokens to recipient
    let nonce = 1;

    const transferAmountOfTokens = parseEther("10");
    const messageHash = await tokenSenderContract.getHash(
      userAddress.address,
      transferAmountOfTokens,
      recipientAddress.address,
      randomTokenContract.address,
      nonce
    );
    const signature = await userAddress.signMessage(arrayify(messageHash));

    // Have the relayer execute the transaction on behalf of the user
    const relayerSenderContractInstance =
      tokenSenderContract.connect(relayerAddress);
    const metaTxn = await relayerSenderContractInstance.transfer(
      userAddress.address,
      transferAmountOfTokens,
      recipientAddress.address,
      randomTokenContract.address,
      nonce,
      signature
    );
    await metaTxn.wait();

    // Check the user's balance decreased, and recipient got 10 tokens
    let userBalance = await randomTokenContract.balanceOf(userAddress.address);
    let recipientBalance = await randomTokenContract.balanceOf(
      recipientAddress.address
    );

    expect(userBalance.eq(parseEther("9990"))).to.be.true;
    expect(recipientBalance.eq(parseEther("10"))).to.be.true;

    // Increment the nonce
    nonce++;

    // Have user sign a second message, with a different nonce, to transfer 10 more tokens
    const messageHash2 = await tokenSenderContract.getHash(
      userAddress.address,
      transferAmountOfTokens,
      recipientAddress.address,
      randomTokenContract.address,
      nonce
    );
    const signature2 = await userAddress.signMessage(arrayify(messageHash2));
    // Have the relayer execute the transaction on behalf of the user
    const metaTxn2 = await relayerSenderContractInstance.transfer(
      userAddress.address,
      transferAmountOfTokens,
      recipientAddress.address,
      randomTokenContract.address,
      nonce,
      signature2
    );
    await metaTxn2.wait();

    // Check the user's balance decreased, and recipient got 10 tokens
    userBalance = await randomTokenContract.balanceOf(userAddress.address);
    recipientBalance = await randomTokenContract.balanceOf(
      recipientAddress.address
    );

    expect(userBalance.eq(parseEther("9980"))).to.be.true;
    expect(recipientBalance.eq(parseEther("20"))).to.be.true;
  });

  it("Should not let signature replay happen", async function () {
    // Deploy the contracts
    const RandomTokenFactory = await ethers.getContractFactory("RandomToken");
    const randomTokenContract = await RandomTokenFactory.deploy();
    await randomTokenContract.deployed();

    const MetaTokenSenderFactory = await ethers.getContractFactory(
      "TokenSender"
    );
    const tokenSenderContract = await MetaTokenSenderFactory.deploy();
    await tokenSenderContract.deployed();

    // Get three addresses, treat one as the user address
    // one as the relayer address, and one as a recipient address
    const [_, userAddress, relayerAddress, recipientAddress] =
      await ethers.getSigners();

    // Mint 10,000 tokens to user address (for testing)
    const tenThousandTokensWithDecimals = parseEther("10000");
    const userTokenContractInstance = randomTokenContract.connect(userAddress);
    const mintTxn = await userTokenContractInstance.freeMint(
      tenThousandTokensWithDecimals
    );
    await mintTxn.wait();

    // Have user infinite approve the token sender contract for transferring 'RandomToken'
    const approveTxn = await userTokenContractInstance.approve(
      tokenSenderContract.address,
      BigNumber.from(
        // This is uint256's max value (2^256 - 1) in hex
        "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"
      )
    );
    await approveTxn.wait();

    // Have user sign message to transfer 10 tokens to recipient
    let nonce = 1;

    const transferAmountOfTokens = parseEther("10");
    const messageHash = await tokenSenderContract.getHash(
      userAddress.address,
      transferAmountOfTokens,
      recipientAddress.address,
      randomTokenContract.address,
      nonce
    );
    const signature = await userAddress.signMessage(arrayify(messageHash));

    // Have the relayer execute the transaction on behalf of the user
    const relayerSenderContractInstance =
      tokenSenderContract.connect(relayerAddress);
    const metaTxn = await relayerSenderContractInstance.transfer(
      userAddress.address,
      transferAmountOfTokens,
      recipientAddress.address,
      randomTokenContract.address,
      nonce,
      signature
    );
    await metaTxn.wait();

    // Have the relayer attempt to execute the same transaction again with the same signature
    // This time, we expect the transaction to be reverted because the signature has already been used.
    expect(
      relayerSenderContractInstance.transfer(
        userAddress.address,
        transferAmountOfTokens,
        recipientAddress.address,
        randomTokenContract.address,
        nonce,
        signature
      )
    ).to.be.revertedWith("Already executed!");
  });
});

```

The first test here has the user sign two distinct signatures with two distinct nonces, and the relayer executes them both.&#x20;

In the second test, however, the relayer attempts to execute the same signature twice. The second time the relayer tries to use the same signature, we expect the transaction to revert.

If you run **`npx hardhat test`** here and all tests succeed, that means that the second transaction with the replay attack was reverted.
