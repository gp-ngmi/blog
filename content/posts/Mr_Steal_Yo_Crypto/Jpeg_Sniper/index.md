---
title: "Mr Steal Yo Crypto — Jpeg Sniper"
date: 2023-07-22
draft: false
description: "CTF - 1"
series: ["CTF"]
series_order: 1
tags: ["CTF"]
topics: ["CTF"]
showAuthor: false
showAuthorsBadges : false
---

This is the #1 challenge of [Mr Steal Yo Crypto](https://mrstealyocrypto.xyz/index.html). I will try to explain how to find the vulnerability.

> A set of challenges to learn offensive security of smart contracts. Featuring interesting challenges loosely (or directly) inspired by real world exploits.

Created by [@0xToshii](https://twitter.com/0xToshii)

I used the foundry version for this CTF: [mr-steal-yo-crypto-ctf-foundry](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement)

![Mr Steal Yo Crypto](https://cdn-images-1.medium.com/max/800/1*67w9ffBxLP4AMvoxHowZ2w.jpeg)

### Jpeg Sniper

> Hopegs the NFT marketplace is launching the hyped NFT collection BOOTY soon.
> They have a wrapper contract: FlatLaunchpeg, which handles the public sale mint for the collection.
> Your task is to bypass their safeguards and max mint the entire collection in a single tx.

[See the contracts](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/tree/implement/src/jpeg-sniper)

[Complete the challenge](https://github.com/0xToshii/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/1-jpeg-sniper.sol)

### Review of the contracts

**LaunchpegErrors.sol**

This contract contains some declared error types. It will be more interesting when we will see where they are implemented.

**BaseLaunchpegNFT.sol**

This is the core of the NFT marketplace. We will use the function `_mintForUser()` for minting the NFT through `FlatLaunchpeg.sol`.

Here are some interesting notes:
- We can see the modifier `isEOA()`, which is almost useless because it is very easy to bypass this modifier. We know that when a contract is deployed, the code size of the contract is equal to 0, so if we do our things inside the constructor, we can bypass the modifier. Now we need to see where the modifier is used.
- There can be a possibility of reentrancy through `_refundIfOver()`. If we send some ether, the contract sends it back to us (this is a free mint). We will see if this vulnerability will be useful or not.

```csharp
/// @dev Verifies that enough funds have been sent by the sender and refunds the extra tokens if any
/// @param _price The price paid by the sender for minting NFTs
function _refundIfOver(uint256 _price) internal {
    if (msg.value < _price) {
        revert Launchpeg__NotEnoughFunds(msg.value);
    }
    if (msg.value > _price) {
        (bool success, ) = msg.sender.call{value: msg.value - _price}("");
        if (!success) {
            revert Launchpeg__TransferFailed();
        }
    }
}
```

**FlatLaunchpeg.sol**

This contract implements `BaseLaunchpegNFT.sol` and adds some details.
We will only focus on `publicSaleMint()` because we will always be in `Phase.PublicSale` and have no problem with the modifier `atPhase()`. If we want to mint the NFTs, the first step is to call the `publicSaleMint()` function. As we said a little bit above, the modifier `atPhase()` will not give us any problem, and we can easily bypass the modifier `isEOA()`.

```csharp
/// @notice Mint NFTs during the public sale
/// @param _quantity Quantity of NFTs to mint
function publicSaleMint(uint256 _quantity)
    external
    payable
    isEOA
    atPhase(Phase.PublicSale)
{
    if (numberMinted(msg.sender) + _quantity > maxPerAddressDuringMint) {
        revert Launchpeg__CanNotMintThisMany();
    }
    if (totalSupply() + _quantity > collectionSize) {
        revert Launchpeg__MaxSupplyReached();
    }
    uint256 total = salePrice * _quantity;

    _mintForUser(msg.sender, _quantity);
    _refundIfOver(total);
}
```

Here are my notes:
- `maxPerAddressDuringMint` is set at five in the test file. So we will need to find a solution to pass this requirement.
- The second condition is pretty obvious, so there is no trick for this one.
- Then we mint `_quantity` of NFTs, and we are refunded if we send some ether.

Now we have all the information for exploiting the marketplace.

### Exploit the vulnerability

I didn’t make this challenge alone. I did it with [https://twitter.com/mis4nthr0pic](https://twitter.com/mis4nthr0pic) (in his discord server) and some other guys.

At the beginning, I thought that we should use the possibility of reentrancy for minting all the NFTs like this:

1. Inside the constructor, we call `publicSaleMint()` and send some ether.
2. The `FlatLaunchpeg` contract mints our NFTs and sends our ethers.
3. When we go back to the receive function of our contract, we transfer the NFTs and call `publicSaleMint()` again. We are transferring the NFTs, which allows us to pass the first condition. `numberMinted(msg.sender)` will always be lower than 5.

However, it doesn't work at all. I never deep dive into the issue, but basically the `receive()` function doesn't work when you are deploying a contract. Only when the smart contract is fully deployed, you are able to use reentrancy, but then the modifier `isEOA()` blocks us.

So [https://twitter.com/mis4nthr0pic](https://twitter.com/mis4nthr0pic) suggested an easier path:

1. Deploy a smart contract where we do a for loop.
2. Inside the loop, we are creating a new contract that will mint 5 NFTs and send them to the attacker's address.
3. Then we are minting the remaining NFTs, and that's it.

So that's what we did, and it worked. You can have a look [here](https://github.com/gp-ngmi/mr-steal-yo-crypto-ctf-foundry/blob/implement/test/1-jpeg-sniper.sol).

### Acknowledgement

Thank you [https://stermi.xyz/](https://stermi.xyz/) for inspiring me to write articles on CTF ^^.