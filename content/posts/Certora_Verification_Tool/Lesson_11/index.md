---
title: "Certora Tutorials — Lesson 11"
date: 2023-07-22
draft: false
description: "Certora Tutorials — Lesson 11"
tags: ["Formal Verification"]
series: ["Formal Verification"]
series_order: 10
topics: ["Formal Verification"]
showAuthor: false
showAuthorsBadges : false
---


![Certora Tutorials](https://cdn-images-1.medium.com/freeze/max/800/1*HnSErp18uxfhXvmqeHIWkA.jpeg)

Hi, it’s me again. Today, we are here for our last lesson! It will be on how to handle loops in the smart contract with the Certora verification tool.

In most programming languages, loops are accompanied by possible computational problems. Certora identifies two main problems. The first one is the significant increase of commands execution in the programs depending on your conditions for the iterations, it can considerably affect your programs. The second one is the [halting problem](https://en.wikipedia.org/wiki/Halting_problem). In brief, if you have a Turing machine, loops can break the rules of determinism. For these problems, the EVM chooses to establish the concept of gas to avoid infinite computation.

As we know, the Certora tool generates symbolic execution from smart contracts. With the presence of a loop, the number of equations can substantially increase, which will make it more difficult for the execution inside the Prover. This is why Certora created useful flags for handling this part more easily.

## Documentation

One of the approximations applied by the Certora Prover is **loop unrolling**. Loops in the contract are replaced by multiple copies of their bodies. The default number of copies is 1.

> From [https://docs.certora.com/en/latest/docs/prover/approx/loops.html?highlight=loop](https://docs.certora.com/en/latest/docs/prover/approx/loops.html?highlight=loop)

Example:

```solidity
/// @notice: `slow_copy(n)` always returns `n`
function slow_copy(uint n) returns uint {
    uint j = 0;
    for (uint i = 0; i < n; i++)
        j++;
    return j;
}
```

Inside the prover, it will be:

```solidity
function slow_copy_unrolled(uint n) returns uint {
    uint j = 0;
    
    uint i = 0;
    if (i < n) {
        j++;
        i++;
        if (i < n) {
            j++;
            i++;
        }
    }
    
    return j;
}
```

We have three modes for handling the loop:

- `loop_iter n`: It will simply duplicate `n` times the code inside the loop.
- The default: **pessimistic mode** will enable the Prover to execute too many times a loop.
- `optimistic_loop`: This time, the Prover will ignore any example with too many loops. If the loop should be executed three times, it will copy the code inside the loop three times.

It may not be globally understanding, so let’s do the exercise!

## Summary Exercise

When we want to verify a smart contract containing some loops, we need to set the correct flag. Otherwise, the pessimistic loop will be activated and most of the time it causes failure of rules.

If you have a dynamic loop, the use of `optimistic_loop` alone is correct. The flag wants an "n" for supposing the number of times the rule is unrolled. Per default `n` will be equal to 1. What you can do is adding the flag `--loop_iter n`. Then, the flag `--optimistic_loop` will make sure that the loop will not iterate more than it should, and `n` will be equal to "n". If you want to use the flag `--loop_iter n` alone, the Prover will automatically find an iteration "n" + 1 (so it’s useless).

If you have a constant iteration loop, depending on `--loop_iteration n`, the rule can pass or not. Adding the flag `--optimistic_loop` does not resolve all the problems. If the `--loop_iter n` is too low, then `--optimistic_loop` transforms the rule into a vacuous rule.

## Conclusion

You need to be clever with the use of loops flags. On a parametric loop, it is often enough to check a few iterations to verify the entire rule (or at least give a very good coverage). Personally, it wasn’t completely clear, so I will go over it again.