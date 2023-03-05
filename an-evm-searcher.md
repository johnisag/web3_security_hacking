---
description: Building an MEV Searcher
---

# An EVM Searcher

In the last lesson, we understood what MEV is, what Flashbots are, and some use cases of Flashbots. In this level we will learn how to mint an NFT using Flashbots. This is going to be a very simple use case designed to teach you how to use Flashbots, not make a profit.&#x20;

Finding opportunities where you can make profit using MEV is a hard problem and are typically not public information. Every Searcher is trying to do their best, and if they tell you exactly what strategies they're using, they are shooting themselves in the foot.

> NOTE: Similar to the arbitrage/flash loan level - you will almost never find an open-source MEV strategy that is actually profitable, because once it's out in the open, there is always someone willing to run it for cheaper driving down the profit margin to zero. MEV is a field which requires a lot of ingenuity to be profitable and keeping the strategy secret. You will find, however, a lot of open-source MEV bots that can help build your understanding on what kind of strategies make sense - even if that specific one is no longer profitable.

This tutorial is just meant to show you how you use Flashbots to send transactions in the first place, the rest is up to you!

### ⚒️ Contract

Note All of these commands should work smoothly . If you are on windows and face Errors Like **`Cannot read properties of null (reading 'pickAlgorithm')`** Try Clearing the NPM cache using **`npm cache clear --force`**.

**Setup a Hardhat project**

```shell
mkdir flashbots
cd flashbots
npm init --yes
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# bootstrap the hardhat project
npx hardhat

# This installs the Flashbots provider which enables eth_sendBundle, 
# the OpenZeppelin contracts, and 
# dotenv to keep our environment variables safe.
npm install @flashbots/ethers-provider-bundle @openzeppelin/contracts dotenv
```

Make sure you select **`Create a Javascript Project`**

Create a new file inside the **`tx-origin/contracts`**directory and call it **`FakeNFT.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";

contract FakeNFT is ERC721 {
    uint256 tokenId = 1;
    uint256 constant price = 0.01 ether;

    constructor() ERC721("FAKE", "FAKE") {}

    function mint() public payable {
        require(msg.value == price, "Ether sent is incorrect");
        _mint(msg.sender, tokenId);
        tokenId += 1;
    }
}
```

Update **hardhat.config.js** file to use the <mark style="color:purple;">**`goerli`**</mark>network

```javascript
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config({ path: ".env" });

const QUICKNODE_RPC_URL = process.env.QUICKNODE_RPC_URL;
const PRIVATE_KEY = process.env.PRIVATE_KEY;

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  solidity: "0.8.17",
  networks: {
    goerli: {
      url: QUICKNODE_RPC_URL,
      accounts: [PRIVATE_KEY],
    },
  },
};
```

Create a `.env` file in the `flashbot`folder and add the following lines

```python
QUICKNODE_RPC_URL="QUICKNODE_RPC_URL"
PRIVATE_KEY="YOUR-PRIVATE-KEY"
QUICKNODE_WS_URL="QUICKNODE_WS_URL"
```

**Steps:**

1. **Create an account** in  [Quicknode](https://www.quicknode.com/?utm\_source=learnweb3\&utm\_campaign=generic\&utm\_content=sign-up\&utm\_medium=learnweb3)&#x20;
2. **`Create an endpoint`** on  [Quicknode](https://www.quicknode.com/?utm\_source=learnweb3\&utm\_campaign=generic\&utm\_content=sign-up\&utm\_medium=learnweb3)&#x20;
   * Select <mark style="color:orange;">**`Ethereum`**</mark>, and then&#x20;
   * Select the <mark style="color:purple;">**`Goerli`**</mark>network (testnet)
   * `Click`` `**`Continue` ** in the bottom right and then click on **`Create Endpoint`**
3. Copy the link given to you in **`HTTP Provider`** and add it to the **`.env`** file for **`QUICKNODE_RPC_URL`**
4. Copy the link given to you in **`WSS Provider`** and add it to the **`.env`** file for **`QUICKNODE_WS_URL`**
5. To get some Goerli ether try out [this faucet](https://goerlifaucet.com/)

Create a new file under **`./scripts` ** folder and name it **`flashbots.js`**&#x20;

```javascript
const {
  FlashbotsBundleProvider,
} = require("@flashbots/ethers-provider-bundle");
const { BigNumber } = require("ethers");
const { ethers } = require("hardhat");
require("dotenv").config({ path: ".env" });

async function main() {
  // Deploy FakeNFT Contract
  const fakeNFT = await ethers.getContractFactory("FakeNFT");
  const FakeNFT = await fakeNFT.deploy();
  await FakeNFT.deployed();

  console.log("Address of Fake NFT Contract:", FakeNFT.address);

  // Create a Quicknode WebSocket Provider
  const provider = new ethers.providers.WebSocketProvider(
    process.env.QUICKNODE_WS_URL,
    "goerli"
  );

  // Wrap your private key in the ethers Wallet class
  const signer = new ethers.Wallet(process.env.PRIVATE_KEY, provider);

  // Create a Flashbots Provider which will forward the request to the relayer
  // Which will further send it to the flashbot miner
  const flashbotsProvider = await FlashbotsBundleProvider.create(
    provider,
    signer,
    // URL for the flashbots relayer
    "https://relay-goerli.flashbots.net",
    "goerli"
  );

  provider.on("block", async (blockNumber) => {
    console.log("Block Number: ", blockNumber);
    // Send a bundle of transactions to the flashbot relayer
    const bundleResponse = await flashbotsProvider.sendBundle(
      [
        {
          transaction: {
            // ChainId for the Goerli network
            chainId: 5,
            // EIP-1559
            type: 2,
            // Value of 1 FakeNFT
            value: ethers.utils.parseEther("0.01"),
            // Address of the FakeNFT
            to: FakeNFT.address,
            // In the data field, we pass the function selctor of the mint function
            data: FakeNFT.interface.getSighash("mint()"),
            // Max Gas Fes you are willing to pay
            maxFeePerGas: BigNumber.from(10).pow(9).mul(3),
            // Max Priority gas fees you are willing to pay
            maxPriorityFeePerGas: BigNumber.from(10).pow(9).mul(2),
          },
          signer: signer,
        },
      ],
      blockNumber + 1
    );

    // If an error is present, log it
    if ("error" in bundleResponse) {
      console.log(bundleResponse.error.message);
    }
  });
}

main();
```

In the initial lines of code, we deployed the `FakeNFT` contract which we wrote.

After that we created an Quicknode WebSocket Provider, a signer and a Flashbots provider.

**Note the reason why we created a WebSocket provider this time is because we want to create a socket to listen to every new block that comes in `Goerli` network.**&#x20;

HTTP Providers, as we had been using previously, work on a request-response model, where a client sends a request to a server, and the server responds back. In the case of WebSockets, however, the client opens a connection with the WebSocket server once, and then the server continuously sends them updates as long as the connection remains open. Therefore the client does not need to send requests again and again.

The reason to do that is that all miners in `Goerli` network are not flashbot miners. This means for some blocks it might happen that the bundle of transactions you send dont get included.

As a reason, we listen for each block and send a request in each block so that when the coinbase miner(miner of the current block) is a flashbots miner, our transaction gets included.

```javascript
// Create a Quicknode WebSocket Provider
const provider = new ethers.providers.WebSocketProvider(
    process.env.QUICKNODE_WS_URL,
    "goerli"
  );
  
  // Wrap your private key in the ethers Wallet class
  const signer = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
  
  // Create a Flashbots Provider which will forward the request to the relayer
  // Which will further send it to the flashbot miner
  const flashbotsProvider = await FlashbotsBundleProvider.create(
    provider,
    signer,
    // URL for the goerli flashbots relayer
    "https://relay-goerli.flashbots.net",
    "goerli"
  );
```

After initializing the providers and signers, we use our provider to listen for the `block` event. Every time a `block` event is called, we print the block number and send a bundle of transactions to mint the NFT. Note the bundle we are sending may or may not get included in the current block depending on whether the coinbase miner is a flashbot miner or not.

Now to create the transaction object, we specify the `chainId` which is `5` for Goerli, `type` which is `2` because we will use the `Post-London Upgrade` gas model which is `EIP-1559`. To refresh your memory on how this gas model works, check out the `Gas` module in Sophomore.

We specify `value` which is `0.01` because that's the amount for minting 1 NFT and the `to` address which is the address of `FakeNFT` contract.

Now for `data` we need to specify the function selector which is the first four bytes of the Keccak-256 (SHA-3) hash of the name and the arguments of the function This will determine which function are we trying to call, in our case, it will be the mint function.

Then we specify the `maxFeePerGas` and `maxPriorityFeePerGas` to be `3 GWEI` and `2 GWEI` respectively. Note the values I got here are from looking at the transactions which were mined previously in the network and what `Gas Fees` were they using.

also, `1 GWEI = 10*WEI = 10*10^8 = 10^9`

We want the transaction to be mined in the next block, so we add 1 to the current blocknumber and send this bundle of transactions.

After sending the bundle, we get a `bundleResponse` on which we check if there was an error or not, if yes we log it.

Now note, getting a response doesn't guarantee that our bundle will get included in the next block or not. To check if it will get included in the next block or not you can use `bundleResponse.wait()` but for the sake of this tutorial, we will just wait patiently for a few blocks and observe.

```javascript
provider.on("block", async (blockNumber) => {
    console.log("Block Number: ", blockNumber);
    // Send a bundle of transactions to the flashbot relayer
    const bundleResponse = await flashbotsProvider.sendBundle(
      [
        {
          transaction: {
            // ChainId for the Goerli network
            chainId: 5,
            // EIP-1559
            type: 2,
            // Value of 1 FakeNFT
            value: ethers.utils.parseEther("0.01"),
            // Address of the FakeNFT
            to: FakeNFT.address,
            // In the data field, we pass the function selctor of the mint function
            data: FakeNFT.interface.getSighash("mint()"),
            // Max Gas Fees you are willing to pay
            // we need to increase this, 05/3/23 * 10
            maxFeePerGas: BigNumber.from(10).pow(9).mul(3).mul(10),
            // Max Priority gas fees you are willing to pay
            maxPriorityFeePerGas: BigNumber.from(10).pow(9).mul(2),
          },
          signer: signer,
        },
      ],
      blockNumber + 1
    );
  
    // If an error is present, log it
    if ("error" in bundleResponse) {
      console.log(bundleResponse.error.message);
    }
  });
```

Now to run this code, in your terminal pointing to the root directory execute the following command:

```sh
npx hardhat run scripts/flashbots.js --network goerli
```

After an address is printed on your terminal, go to [Goerli Etherscan](https://goerli.etherscan.io/) and keep refreshing the page till you see `Mint` transaction appear(Note it takes some time for it to appear cause the flashbot miner has to be the coinbase miner for our bundle to be included in the block)

![Image](https://i.imgur.com/sVwacVp.png)

![Image](https://i.imgur.com/Aawg5gK.png)

Boom 🤯 We now learned how to use flashbots to mint a NFT but you can do so much more 👀
