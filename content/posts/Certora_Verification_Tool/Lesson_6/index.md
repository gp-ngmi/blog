---
title: "Certora Tutorials — Lesson 6"
date: 2023-07-22
draft: false
description: "Certora Tutorials — Lesson 6"
tags: ["Formal Verification"]
series: ["Formal Verification"]
series_order: 6
topics: ["Formal Verification"]
showAuthor: false
showAuthorsBadges : false
---


![Certora Tutorials](https://cdn-images-1.medium.com/freeze/max/800/1*-eX0XyLCKvDtPBvuGg-z8w.jpeg)

Everything is clear for the moment? Good because today will be a theoretical lesson on the mindset of how analyzing a bunch of smart contracts. This lesson can be helpful not only in the formal verification sector but also for auditors. Let's start ^^

## A Guide For Thinking About Properties

First of all, I recommend you to have a quick check on the [introduction](https://github.com/Certora/Tutorials/blob/master/06.Lesson_ThinkingProperties/README.md) of the lesson. I will try to recap with my words.

Before you begin writing rules, you need to understand the system that you want to test. Getting familiar with the system is your first priority; you will be able to detect some system requirements. The others will come with the iterative process. During the verification process, new properties can appear, and these will have to be implemented into your specification file. This first part of understanding is essential because the goal of the specification file is to push the system to its limit. You need to expand the scenarios to analyze if the system handles it or not.

## Categorizing Properties

This [video](https://www.youtube.com/watch?v=f3K-68k7vig) will be of assistance to get the point on these [slides](https://github.com/Certora/Tutorials/blob/master/06.Lesson_ThinkingProperties/Categorizing_Properties.pdf). As we said earlier, we need to understand the system of the smart contracts, and after that, we can create properties. Certora suggests different points of properties with the intention of writing a good specification file. In my opinion, these five types of properties are very connected with the way to think of the system as a state machine.

- **Valid States**: The system needs to establish a set of valid states. There is only one valid state at a time. Depending on the state (not allowed, pending, allowed), the system is enabled to realize some actions.

- **State Transitions**: Related to the first type of properties. State transitions control the transition between two states. Firstly, it will inspect if the new valid state respects the correct order of the transition between the valid states. Secondly, it will control if the conditions are fulfilled for the transition of valid states. In our case, we can't go from pending to not allowed, and we need to wait for 10 new blocks before pending -> allowed.

- **Variable Transitions**: This type is regarding the evolution of variables. A variable can be modified throughout the life cycle of the system. It might change in various ways or unidirectional (a variable that counts the number of mints of an NFT).

- **High-Level Properties**: This type has lesson 9 just for him. It covers the basic properties of the whole system. In our model, if you mint an NFT -> you will have an NFT in your wallet.

- **Unit Tests**: This will focus on an individual function or a few of them. With the purpose of verifying that they perform as desired. Within our example, the function `getTotalMints()` should return the total mints of NFT.

## Conclusion

This lesson allows you to understand how to write specifications. This is the most sensitive part when you are working with CVT. Once again, with weak properties, the formal verification is worthless.