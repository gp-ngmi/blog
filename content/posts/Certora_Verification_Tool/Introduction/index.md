---
title: "Quick introduction to Formal Verification"
date: 2023-07-22
draft: false
description: "Introduction to formal verification"
tags: ["Formal Verification"]
series: ["Formal Verification"]
series_order: 1
topics: ["Formal Verification"]
showAuthor: false
showAuthorsBadges : false
---


When I deep dived into the blockchain ecosystem, I had a trait for demystifying the implementations of the protocol. For example, when I understood that Uniswap v2 was one of the most used DeFi protocols, I simply took a look into the smart contracts.

Then, I had an attraction to security. So, firstly, I learned the basic vector attack at [swcregistry.io](https://swcregistry.io/). In the meantime, I tried to learn how to write unit tests for smart contracts and how to use tools like [Echidna](https://github.com/crytic/echidna) or [Slither](https://github.com/crytic/slither). During my learning session, I watched this video [Martin Lundfall: “Smart contracts as inductive systems”](https://www.youtube.com/watch?v=WbL8U-nyhJE). This sparked my interest in formal verification.

This article will be just an introduction to the concept of formal verification. Then I will present a bunch of tools for writing specifications. So let’s dive in!

## Concept of Formal Verification

Formal verification uses exhaustive mathematical methods for proving or disproving the correctness of a design (hardware or software systems) through formal specification (requirements). To be more precise, on one hand, you have your source code. On the other hand, you have a bunch of specifications (properties). The formal verification will be processed by transforming your source code into an abstract mathematical model. In order to prove mathematically that your code is respecting your specification. That’s a powerful feature for minimizing the risk of errors or bugs.

The counterpart of this method is the complexity of the implementation of this system. You will need a tool that allows transforming your code into an abstract mathematical model. Who can say that there is no error during the transformation?

The other counterpart is the specification. It takes a lot of time to make correct properties. If the specifications are incorrect or not worthwhile, then the formal verification is useless.

I didn’t make some research about where formal verification is used outside the blockchain. But I know that NASA and the aeronautics sector use the formal method for the pertinence of aiding design assurance.

## Tools

This part will be quick, I will just release some resources without explanation:

- Understand formal verification: [https://ethereumorgwebsitedev01-emmanuelawosikaethereumor06662.gtsb.io/en/developers/docs/smart-contracts/formal-verification](https://ethereumorgwebsitedev01-emmanuelawosikaethereumor06662.gtsb.io/en/developers/docs/smart-contracts/formal-verification)
- [Coq](https://coq.inria.fr/) and [K](https://kframework.org/) are formal verification tools for a lot of languages.
- [Act](https://fv.ethereum.org/2021/08/31/act-0.1/), [Certora Prover](https://demo.certora.com/) are specification languages for Solidity.

## Conclusion

That was a very quick introduction to formal verification. It was my first article; this is the beginning of a series on the Certora Prover. See you soon!
