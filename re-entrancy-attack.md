---
description: 🤯 The bug which cost $60 million dollars - Re-entrancy
---

# Re-entrancy Attack

Re-Entrancy is one of the oldest security vulnerabilities that was discovered in smart contracts. It is the exact vulnerability that caused the infamous 'DAO Hack' of 2016. Over 3.6 million ETH was stolen in the hack, which today is worth billions of dollars. 🤯

At the time, the DAO contained 15% of all Ethereum on the network as Ethereum was relatively new. The failure was having a negative impact on the Ethereum network, and Vitalik Buterin proposed a software fork where the attacker would never be able to transfer out his ETH. Some people agreed, some did not. This was a highly controversial event, and one which still is full of controversy.

At the end, it led to Ethereum being forked into two - Ethereum Classic, and the Ethereum we know today. Ethereum Classic's blockchain is the exact same as Ethereum up until the fork, but then proceeded as if the hack did happen and the attacker still controls the stolen funds. Today's Ethereum implemented the blacklist and it's as if that attack never happened. 🤔

This is a simplified version of that story, and the entire dynamic was quite complex. Everyone was stuck between a rock and a hard place. [You can read more about this story here to know what happened in more detail](https://www.coindesk.com/learn/2016/06/25/understanding-the-dao-attack/)

### 👀 What is Re-Entrancy?

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

Re-Entrancy is the vulnerability in which if `Contract A` calls a function in `Contract B`, `Contract B` can then call back into `Contract A` while `Contract A` is still processing.

This can lead to some serious vulnerabilities in Smart contracts, often creating the possibility of draining funds from a contract.

Let's understand how this works with the example shown in the above diagram. Let's say `Contract A` has some function - call it `f()` that does 3 things:

* Checks the balance of ETH deposited into `Contract A` by `Contract B`
* Sends the ETH back to `Contract B`
* Updates the balance of `Contract B` to 0

Since the balance gets updated after the ETH has been sent, `Contract B` can do some tricky stuff here. If `Contract B` was to create a `fallback()` or `receive()` function in it's contract, which would execute when it received ETH, it could call `f()` in `Contract A` again.

Since `Contract A` hasn't yet updated the balance of `Contract B` to be 0 at that point, it would send ETH to `Contract B` again - and herein lies the exploit, and `Contract B` could keep doing this until `Contract A` was completely out of ETH.

### ⚒️ Contract

We will create a couple of smart contracts, `GoodContract` and `BadContract` to demonstrate this behaviour. `BadContract` will be able to drain all the ETH out from `GoodContract`.

> Note All of these commands should work smoothly . If you are on windows and face Errors Like **`Cannot read properties of null (reading 'pickAlgorithm')`** Try Clearing the NPM cache using **`npm cache clear --force`**.

**Setup a Hardhat project**

```shell
mkdir re-entrancy
cd re-entrancy
npm init --yes
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# bootstrap the hardhat project
npx hardhat
```
