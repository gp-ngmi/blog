---
title: "Certora Tutorials — Lesson 0 to 2"
date: 2023-07-22
draft: false
description: "Certora Tutorials — Lesson 0 to 2"
tags: ["Formal Verification"]
series: ["Formal Verification"]
series_order: 2
topics: ["Formal Verification"]
showAuthor: false
showAuthorsBadges : false
---


As mentioned in the previous article, Certora creates software called [Certora Prover](https://www.certora.com/#About). It allows verifying your specifications or finding bugs in your compiled smart contracts. The [Certora tutorials](https://github.com/Certora/Tutorials) is a course to start writing specifications in Certora’s spec language.

Today I will share my understanding and my opinion on lessons 0 to 2. Let’s dive in!

## Lesson 0

In this lesson, you have three resources: a [video](https://youtu.be/VGSsPIsbb6U), a [quiz](https://docs.google.com/document/d/19lLWqTrm_bzDdY9Uk-LpTSkgabk9WGtuCTPLDzSoraM/edit?usp=sharing) ([answers](https://docs.google.com/document/d/1q6N6lMopjQvCaLQVDN844qs2zFueUTuPEW7CBIjyOmQ/edit?usp=sharing)), and books ([introduction to logic](https://web.mit.edu/gleitz/www/Introduction%20to%20Logic%20-%20P.%20Suppes%20%281957%29%20WW.pdf), [discrete mathematics](http://discrete.openmathbooks.org/dmoi2/sec_propositional.html)).

### [Auditing and Formal Verification — Better together](https://www.youtube.com/watch?v=VGSsPIsbb6U)

The video presents the Certora Prover. The first part of the video presents how the Certora Prover works at an abstract level. To use the tool, you will need a smart contract (in Solidity) and your specifications (Certora’s spec language). I didn’t try to find if there is any documentation about the prover itself, but for the moment, I don’t mind; I just want to be able to write specifications.

The second part is about the specification and what is inside. Firstly, Mooly Sagiv talks about invariants. It is one of the cores of the specification. Invariants are rules that are always respected no matter the situation. When you are building a DeFi protocol, you want these invariants to remain correct. Because if not, the risk of a hack is much bigger. Here is an example: you are building a lending protocol. One of your invariants is that the TVL (Total Value Locked) of the lending pool is always higher than the borrowed allowance (if it’s not the case...).

The third part is an example of how a specification finds a bug in a contract. I will not describe the example because I have anything to add on this part. However, Mooly Sagiv said something really important about the mindset to have when you are using the Certora Prover. The biggest value of formal verification is finding bugs. It’s impossible to guarantee there is no bug inside the contract. Indeed, your specifications can be correct, but then here is my question: how can you be sure that they cover all possibilities of bugs?

### Quiz and Books

The goal of the [quiz](https://docs.google.com/document/d/19lLWqTrm_bzDdY9Uk-LpTSkgabk9WGtuCTPLDzSoraM/edit) was to test our preliminary knowledge in propositional logic and properties of systems.


# Part 1

The proportional logic part was the hardest part due to a lack of mathematics knowledge. With the help of [Introduction to Logic](https://web.mit.edu/gleitz/www/Introduction%20to%20Logic%20-%20P.%20Suppes%20%281957%29%20WW.pdf), I was able to understand the logic and managed this part. At the end, the most important thing to retain is to take your time for thinking about all the scenarios. I had a few wrong answers because I didn't take my time. So let's take an example:

> Question 10: ((p ⇒ q) ⇒ r) ⇒ (p ⇒ (q ⇒ r)) | correct answer: True in all cases

![Question 10 Example](https://cdn-images-1.medium.com/freeze/max/800/1*-O_GewGOXilZecOFszkNJQ.png)
(Source: [Image](https://cdn-images-1.medium.com/max/800/1*-O_GewGOXilZecOFszkNJQ.png))

# Part 2

It was my favorite part! I was very comfortable due to having done [ethernaut](https://ethernaut.openzeppelin.com/), [damnvulnerabledefi](https://www.damnvulnerabledefi.xyz/). However, it wasn't easy!

I had some trouble with question 18. At the beginning, I didn't find any bugs. Then I realized that in the function `transfer()`, there is no verification for `msg.sender ≠ address to`. So `msg.sender` can increase his balances with the function `buy()`. In the correction, they say that there is a bug in function `buy()` for the property P2. For this property, I'm not able to find the bug.

# Part 3

Nothing to say on this part, the questions were simple.

---

## Lesson 1

This was the first lesson where I was going to use the certora prover. The goal was to learn how to think and write high-level properties. I will make a recap of what I learned.

- A specification file (.spec) can contain one or several rules.
- A rule allows you to verify one property. You can do more than one property, but it's not efficient.
- When a rule is disproved, the certora prover will find an example such that we can debug the contract. That's why it's recommended to have some context in your rule for debugging the contract. You can add a struct variable `env` for the EVM context (msg.sender, block.number, block.timestamp). Also, you can add some variables to see what allows a violation of the rule.

![EVM Context](https://cdn-images-1.medium.com/freeze/max/800/1*khf3eFDEU5189xt5N5fwZw.png)
(Source: [Image](https://cdn-images-1.medium.com/max/800/1*khf3eFDEU5189xt5N5fwZw.png))

- The tool assumes all possible input values as the initial state. For example, it's possible to have the `msg.sender` balance > `totalBalance` as the starting state. In order to avoid that, you can add preconditions (`-> require`) for fixing this type of issues.

![Preconditions](https://cdn-images-1.medium.com/freeze/max/800/1*JMXVUTGqDDbfGYhyydQK4A.png)
(Source: [Image](https://cdn-images-1.medium.com/max/800/1*JMXVUTGqDDbfGYhyydQK4A.png))

- If you want to call every function for testing the rule, you can add `method f` as a parameter of the rule. It allows simulating the execution of all functions.

```solidity
calldata arg; // any argument
f(e, arg);
```

![Simulating Function Execution](https://cdn-images-1.medium.com/freeze/max/800/1*ykPe6q16qZU8VLdoQ7MHdg.png)
(Source: [Image](https://cdn-images-1.medium.com/max/800/1*ykPe6q16qZU8VLdoQ7MHdg.png))

## Lesson 2

This lesson was just a training on what we learned in lesson 1. I will just post some screen for Borda exercises.

Borda.spec:
I have begun by adding more variables in the rule `correctPointsIncreaseToContenders` to have a better context of the rule if a violation is discovered.

```solidity
uint256 firstPointsAfter = getPointsOfContender(e, first);
uint256 secondPointsAfter = getPointsOfContender(e, second);
uint256 thirdPointsAfter = getPointsOfContender(e, third);
```

BordaBug1.sol:
The rule `correctPointsIncreaseToContenders` is violated for this contract because all contenders receive 3 points.

![BordaBug1.sol](https://cdn-images-1.medium.com/freeze/max/800/1*604WP3NfveyQyg4iU2Yv8w.png)
(Source: [Image](https://cdn-images-1.medium.com/max/800/1*604WP3NfveyQyg4iU2Yv8w.png))

BordaBug2.sol:
The rule `onceBlackListedNotOut` is violated for this contract due to a mistake in a require().

![BordaBug2.sol](https://cdn-images-1.medium.com/freeze/max/800/1*Q6vZty7IeYZ1XtFxiNBoCA.png)
(Source: [Image](https://cdn-images-1.medium.com/max/800/1*Q6vZty7IeYZ1XtFxiNBoCA.png))

BordaBug3.sol:
The rule `contendersPointsNondecreasing` is violated for this contract due to an overflow.

![BordaBug3.sol](https://cdn-images-1.medium.com/freeze/max/800/1*1a7LqGPakAoQ00_nofdkdQ.png)
(Source: [Image](https://cdn-images-1.medium.com/max/800/1*1a7LqGPakAoQ00_nofdkdQ.png))

BordaBug4.sol:
The rule `registeredCannotChangeOnceSet` is violated for this contract due to a line that creates an inconsistency.

![BordaBug4.sol](https://cdn-images-1.medium.com/freeze/max/800/1*WuiPs6Id5Sk35wA0wsrIgQ.png)
(Source: [Image](https://cdn-images-1.medium.com/max/800/1*WuiPs6Id5Sk35wA0wsrIgQ.png))

## Conclusion

I recommend you to check [Certora Technology White Paper](https://medium.com/certora/certora-technology-white-paper-cae5ab0bdf1). It may be way better than my articles.