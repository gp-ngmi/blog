---
title: "Certora Tutorials — Lesson 4"
date: 2023-07-22
draft: false
description: "Certora Tutorials — Lesson 4"
tags: ["Formal Verification"]
series: ["Formal Verification"]
series_order: 4
topics: ["Formal Verification"]
showAuthor: false
showAuthorsBadges : false
---

![Certora](https://cdn-images-1.medium.com/freeze/max/800/1*_XqBoXnORiTa4fq9_FP-kw.jpeg)

Hello everyone, after a pretty long pause. It’s time to go back in order to finalize the project. This lesson is pretty wide, it will cover some theoretical course and the syntax CVT (Certora Verification tool).

## How to Think of Systems as State Machines:

Here are the resources:
- Watch the video from 48:58 to 1:02:45 — [Link](https://youtu.be/cQntMUMQyRw?t=2938)
- [https://youtu.be/8FWfmvj3HYw](https://youtu.be/8FWfmvj3HYw)
- [https://www.cs.cornell.edu/courses/cs211/2006sp/Lectures/L26-MoreGraphs/state_mach.html](https://www.cs.cornell.edu/courses/cs211/2006sp/Lectures/L26-MoreGraphs/state_mach.html)

After reviewing these videos and articles, I understand the core concept of state machines. Correct me if I’m wrong, but a blockchain is monolithic. A block is composed of transactions that are executed one at a time. If we abstract the system, a blockchain “is a state machine”. Every protocol built on a blockchain is constrained to think of a state machine system. Otherwise, there will be some problems.

Abstracting the protocol into a state machine system allows you to have, in theory, a full knowledge of all the possibilities of your protocol (smart contract). This is why there are a lot of hacks in DeFi; it’s because the team doesn’t really control what they built. Understanding the implementation of a protocol using the concept of state machine will facilitate the understanding and identification of vulnerabilities in a system.

To conclude, it strengthens my belief in the importance of documentation and the conception of a protocol before even writing a line of code.

## Declaring Methods, Creating Definitions, and Using CVL Functions

**Methods**:
As we have seen in Lesson 1, instead of creating a rule for each function, we can put the method `f` in the parameter of the rule and then put `f(e, arg)` inside the rule. It will call every function of the contract in the rule. To make it more readable for others, you can create a method block at the beginning of the document and list every callable function you want. You can also describe if the function is `env` or `envfree` at the end of the function declaration. In CVT, `env` means the calling context (sender, value, timestamp, block number, etc.) is not always important for a function’s operation like a pure or view function. The methods component makes it easier to understand what function we will call inside the specification file.

Example:
```md
methods{
    // getFunds implementation does not require any context to get successfully executed
    getFunds(address) returns (uint256) envfree
    // deposit's implementation uses msg.sender, info that's encapsulated in the environment
    deposit(uint256)
}
```

**CVL Functions**:
You are able to create functions to not duplicate code. For example:
```md
function abs_value_difference(uint256 x, uint256 y) returns uint256 {
    if (x < y) {
        return y - x;
    } else {
        return x - y;
    }
}
```
This function can be callable in a rule or in another CVL function.

**Definitions**:
Definitions are macros that we can declare at the top-level of a specification and are in scope inside every rule, function, and other definitions. They can be used as constants or as verifying expressions.

Example:
```md
definition MAX_UINT256() returns uint256 = 0xffffffffffffffffffffffffffffffff;
definition is_even(uint256 x) returns bool = exists y. 2 * y == x;

rule my_rule(uint256 x) {
    require is_even(x) && x <= MAX_UINT256();
}
```

**Ghost Function**:
A ghost function is useful for keeping track of the state of the contract with the help of axioms and hooks.

**Types and Type Handling**:
For information about the different CVL types available to use, their distinction from EVM types, casting operations, and useful macros, refer to the documentation: [https://certora.atlassian.net/wiki/spaces/CPD/pages/7340101/Types](https://certora.atlassian.net/wiki/spaces/CPD/pages/7340101/Types).

Except for the predefined type `env`, the fields of `env` can be very useful for verifying some elements of the EVM on the system. For example, we can use `env.msg.sender` and `env.tx.origin` to check if it's a user or a contract that is calling a function.

## Conclusion:

We have finished Lesson 4, and we have deep-dived a little bit more than usual. However, now I understand how powerful CVT can be. For more in-depth information, I recommend you check the [Certora Technology White Paper](https://medium.com/certora/certora-technology-white-paper-cae5ab0bdf1).