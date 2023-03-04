---
description: 🆙 Upgradeable Smart Contracts and Proxy Patterns
---

# Upgradable Smart Contracts

**We know that smart contracts on Ethereum are immutable**, as the code is immutable and cannot be changed once it is deployed.&#x20;

But writing perfect code the first time around is hard, and as humans we are all prone to making mistakes. Sometimes even contracts which have been audited turn out to have bugs that cost them millions.

### How does it work?

To upgrade our contracts we use something called the `Proxy Pattern`. The word `Proxy` might sound familiar to you because it is not a web3-native word.

![Image](https://i.imgur.com/llJGnTF.png)

Essentially how this pattern works is that a contract is split into two contracts - `Proxy Contract` and the `Implementation` contract.

**The `Proxy Contract` is responsible for managing the state of the contract** which involves persistent storage whereas **`Implementation Contract` is responsible for executing the logic and doesn't store any persistent state**.&#x20;

User calls the `Proxy Contract` which further does a `delegatecall` to the `Implementation Contract` so that it can implement the logic.&#x20;

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

<mark style="color:purple;">This pattern becomes interesting when</mark> <mark style="color:purple;"></mark><mark style="color:purple;">`Implementation Contract`</mark> <mark style="color:purple;"></mark><mark style="color:purple;">can be replaced which means the logic which is executed can be replaced by another version of the</mark> <mark style="color:purple;"></mark><mark style="color:purple;">`Implementation Contract`</mark> <mark style="color:purple;"></mark><mark style="color:purple;">without affecting the state of the contract which is stored in the proxy.</mark>

There are mainly **three ways in which we can replace/upgrade** the `Implementation Contract`:

1. **Diamond** Implementation
2. **Transparent** Implementation
3. **UUPS** Implementation

We will however only focus on Transparent and UUPS because they are the most commonly used ones.

_To upgrade the **`Implementation Contract`** you will have to use some method like **`upgradeTo(address)`** which will essentially change the address of the **`Implementation Contract`** from the old one to the new one._

<mark style="color:blue;">But the important part lies in</mark> <mark style="color:blue;"></mark><mark style="color:blue;">**where should we keep the**</mark><mark style="color:blue;">** **</mark><mark style="color:blue;">**`upgradeTo(address)`**</mark> <mark style="color:blue;"></mark><mark style="color:blue;">function, we have two choices:</mark>

* **keep it in the `Proxy Contract`** which is essentially how **`Transparent Proxy Pattern`** works,or&#x20;
* or **keep it in the `Implementation Contract`** which is how the **UUPS contract** works.

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

**Another important thing to note about this `Proxy Pattern` is that the constructor of the `Implementation Contract` is never executed.**

_When deploying a new smart contract, the code inside the constructor is not a part of the contract's runtime bytecode because it is only needed during the deployment phase and runs only once._&#x20;

Now because when `Implementation Contract` was deployed it was initially not connected to the `Proxy Contract` as a reason any state change that would have happened in the constructor is now not there in the `Proxy Contract` which is used to maintain the overall state.

As a reason `Proxy Contracts` are unaware of the existence of constructors. Therefore, instead of having a constructor, we use something called an `initializer` function which is called by the `Proxy Contract` once the `Implementation Contract` is connected to it. This function does exactly what a constructor is supposed to do but is now included in the runtime bytecode as it behaves like a regular function and is callable by the `Proxy Contract`.

Using OpenZeppelin contracts, you can use their `Initialize.sol` contract which makes sure that your `initialize` function is executed only once just like a contructor

```solidity
// contracts/MyContract.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract MyContract is Initializable {
    function initialize(
        address arg1,
        uint256 arg2,
        bytes memory arg3
    ) public payable initializer {
        // "constructor" code...
    }
}
```

Above given code is from [Openzeppelin's documentation](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies#the-constructor-caveat) and provides an example of how the `initializer` modifier ensures that the `initialize` function can only be called once. This modifier comes from the `Initializable Contract`

### Transparent Proxy Pattern

The Transparent Proxy Pattern is a simple way to separate responsibilities between `Proxy` and `Implementation` contracts. In this case, the `upgradeTo` function is part of the `Proxy` contract, and the `Implementation` can be upgraded by calling `upgradeTo` on the proxy thereby changing where future function calls are delegated to.

There are some caveats though. There might be a case where the `Proxy Contract` and `Implementation Contract` have a function with the same name and arguments. Imagine if `Proxy Contract` has a `owner()` function and so does `Implementation Contract`. In Transparent Proxy contracts, this problem is dealt by the `Proxy` contract which decides whether a call from the user will execute within the `Proxy` contract itself or the `Implementation Contract` based on the `msg.sender` global variable

So if the `msg.sender` is the admin of the proxy then the proxy will not delegate the call and will try to execute the call if it understands it. If it's not the admin address, the proxy will delegate the call to the `Implementation Contract` even if the matches one of the proxy's functions.

### Issues with Transparent Proxy Pattern

As we know that the address of the `owner` will have to be stored in the storage and using storage is one of the most inefficient and costly steps in interacting with a smart contract every time the user calls the proxy, the proxy checks whether the user is the admin or not which adds unnecessary gas costs to majority of the transactions taking place.

### UUPS Proxy Pattern

The UUPS Proxy Pattern is another way to separate responsibilities between `Proxy` and `Implementation` contracts. In this case, the `upgradeTo` function is also part of the `Implementation` contract, and is called using a `delegatecall` through the Proxy by the owner.

In UUPS whether its the admin or the user, all the calls are sent to the `Implementation Contract` The advantage of this is that every time a call is made we will not have to access the storage to check if the user who started the call is an admin or not which improved efficiency and costs.&#x20;

Also because its the `Implementation Contract` you can customize the function according to your need by adding things like `Timelock`, `Access Control` etc with every new `Implementation` that comes up which couldn't have been done in the `Transparent Proxy Pattern`

### Issues with UUPS Proxy Pattern

The issue with this is now because the `upgradeTo` function exists on the side of the `Implementation contract` developer has to worry about the implementation of this function which may sometimes be complicated and because more code has been added, it increases the possibility of attacks.&#x20;

This function also needs to be in all the versions of `Implementation Contract` which are upgraded which introduces a risk if maybe the developer forgets to add this function and then the contract can no longer be upgraded.

### ⚒️ Contract

Note All of these commands should work smoothly . If you are on windows and face Errors Like **`Cannot read properties of null (reading 'pickAlgorithm')`** Try Clearing the NPM cache using **`npm cache clear --force`**.

**Setup a Hardhat project + open zeppelin deps**

```shell
mkdir upgradable-contracts
cd upgradable-contracts
npm init --yes
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# bootstrap the hardhat project
npx hardhat

# add open zeppelin deps
npm install @openzeppelin/contracts-upgradeable @openzeppelin/hardhat-upgrades
```

This installs the OpenZeppelin upgradeable contracts library and their Hardhat plugin for upgradeable contracts.

Make sure you select **`Create a Javascript Project`**

Replace the code in your **`hardhat.config.js`** with the following code to be able to use these libraries:

```javascript
require("@openzeppelin/hardhat-upgrades");
require("@nomicfoundation/hardhat-toolbox");

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  solidity: "0.8.17",
};

```

Create a new file inside the **`upgradable-contracts/contracts`** directory and call it **`LW3NFT.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "@openzeppelin/contracts-upgradeable/token/ERC721/ERC721Upgradeable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract LW3NFT is
    Initializable,
    ERC721Upgradeable,
    UUPSUpgradeable,
    OwnableUpgradeable
{
    // Note how we created an initialize function and then added the
    // initializer modifier which ensure that the
    // initialize function is only called once
    function initialize() public initializer {
        // Note how instead of using the ERC721() constructor, we have to manually initialize it
        // Same goes for the Ownable contract where we have to manually initialize it
        __ERC721_init("LW3NFT", "LW3NFT");
        __Ownable_init();
        _mint(msg.sender, 1);
    }

    function _authorizeUpgrade(address newImplementation)
        internal
        override
        onlyOwner
    {}
}
```

If you look at all the contracts which `LW3NFT` is importing, you will realize why they are important. First being the `Initializable` contract from Openzeppelin which provides us with the `initializer` modifier which ensures that the `initialize` function is only called once. The `initialize` function is needed because we cant have a contructor in the `Implementation Contract` which in this case is the `LW3NFT` contract

It imports `ERC721Upgradeable` and `OwnableUpgradeable` because the original `ERC721` and `Ownable` contracts have a constructor which cant be used with proxy contracts.

Lastly we have the `UUPSUpgradeable Contract` which provides us with the `upgradeTo(address)` function which has to be put on the `Implementation Contract` in case of a `UUPS` proxy pattern.

After the declaration of the contract, we have the `initialize` function with the `initializer` modifier which we get from the `Initializable` contract. The `initializer` modifier ensures the `initialize` function can only be called once. Also note that the new way in which we are initializing `ERC721` and `Ownable` contract. This is the standard way of initializing upgradeable contracts and you can look at the function [here](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/token/ERC721/ERC721Upgradeable.sol#L45). After that we just mint using the usual mint function.

```solidity
function initialize() public initializer  {
    __ERC721_init("LW3NFT", "LW3NFT");
    __Ownable_init();
    _mint(msg.sender, 1);
}
```

Another interesting function which we dont see in the normal `ERC721` contract is the `_authorizeUpgrade` which is a function which needs to be implemented by the developer when they import the `UUPSUpgradeable Contract` from Openzeppelin, it can be found [here](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/proxy/utils/UUPSUpgradeable.sol#L100).&#x20;

Now why this function has to be overwritten is intresting because it gives us the ability to add authorization on who can actually upgrade the given contract, it can be changed according to requirements but in our case we just added an `onlyOwner` modifier.

```solidity
function _authorizeUpgrade(address newImplementation) internal override onlyOwner {
}
```

Create a new file inside the **`upgradable-contracts/contracts`** directory and call it **`LW3NFT2.sol`**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "./LW3NFT.sol";

contract LW3NFT2 is LW3NFT {
    function test() public pure returns (string memory) {
        return "upgraded";
    }
}
```

This smart contract is much easier because it is just inheriting `LW3NFT` contract and then adding a new function called `test` which just returns a string `upgraded`.

Pretty easy right? 🤯

Wow 🙌, okay we are done with writing the `Implementation Contract`, do we now need to write the `Proxy Contract` as well?

Good news is nope, we dont need to write the `Proxy Contract` because `Openzeppelin` deploys and connects a `Proxy Contract` automatically when we use their library to deploy the `Implementation Contract`.

#### Writing the Test

So lets try to do that, In your `test` directory create a new file named `proxy-test.js` and lets have some fun with code

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");
const hre = require("hardhat");

describe("ERC721 Upgradeable", function () {
  it("Should deploy an upgradeable ERC721 Contract", async function () {
    const LW3NFT = await ethers.getContractFactory("LW3NFT");
    const LW3NFT2 = await ethers.getContractFactory("LW3NFT2");

    // Deploy LW3NFT as a UUPS Proxy Contract
    let proxyContract = await hre.upgrades.deployProxy(LW3NFT, {
      kind: "uups",
    });
    const [owner] = await ethers.getSigners();
    const ownerOfToken1 = await proxyContract.ownerOf(1);

    expect(ownerOfToken1).to.equal(owner.address);

    // Deploy LW3NFT2 as an upgrade to LW3NFT
    proxyContract = await hre.upgrades.upgradeProxy(proxyContract, LW3NFT2);
    // Verify it has been upgraded
    expect(await proxyContract.test()).to.equal("upgraded");
  });
});

```

We first get the `LW3NFT` and `LW3NFT2` instance using the `getContractFactory` function which is common to all the levels we have been teaching till now. After that the most important line comes in which is:

```javascript
let proxyContract = await hre.upgrades.deployProxy(LW3NFT, {
  kind: "uups",
})
```

This function comes from the `@openzeppelin/hardhat-upgrades` library that you installed, It essentially uses the upgrades class to call the `deployProxy` function and specifies the kind as `uups`. When the function is called it deploys the `Proxy Contract`, `LW3NFT Contract` and connects them both. More info about this can be found [here](https://docs.openzeppelin.com/upgrades-plugins/1.x/api-hardhat-upgrades).

Note that the `initialize` function can be named anything else, its just that `deployProxy` by default calls the function with name `initialize` for the initializer but you can modify it by changing the defaults 😇

After deploying, we test that the contract actually gets deployed by calling the `ownerOf` function for Token ID 1 and checking if the NFT was indeed minted.

Now the next part comes in where we want to deploy `LW3NFT2` which is the upgraded contract for `LW3NFT`.

For that we execute the `upgradeProxy` method again from the `@openzeppelin/hardhat-upgrades` library which upgrades and replaces `LW3NFT` with `LW3NFT2` without changing the state of the system

```solidity
proxyContract = await hre.upgrades.upgradeProxy(proxyContract, LW3NFT2);
```

To test if it was actually replaced we call the `test()` function, and ensured that it returned `"upgraded"` even though that function wasn't present in the original `LW3NFT` contract.

#### Running the Test

To run this test, open up your terminal pointing to the root of the directory for this level and execute this command:

```sh
npx hardhat test
```

### Readings

* `Timelock` was mentioned in the given article, to learn more about it you can read the [following article](https://blog.openzeppelin.com/protect-your-users-with-smart-contract-timelocks/)
* `Access Control` was also mentioned and you can read about it more [here](https://docs.openzeppelin.com/contracts/3.x/access-control)

### References

* [OpenZeppelin Docs](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies)
* [OpenZeppelin Youtube](https://www.youtube.com/watch?v=kWUDTZhxKZI)
