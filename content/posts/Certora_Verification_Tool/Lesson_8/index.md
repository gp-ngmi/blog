---
title: "Certora Tutorials — Lesson 8"
date: 2023-07-22
draft: false
description: "Certora Tutorials — Lesson 8"
tags: ["Formal Verification"]
series: ["Formal Verification"]
series_order: 8
topics: ["Formal Verification"]
showAuthor: false
showAuthorsBadges : false
---


![Certora Tutorials](https://cdn-images-1.medium.com/freeze/max/800/1*r2vrJ9-U3N3CpAeg467CuQ.jpeg)

How it’s going? Last lesson we had a theoretical course on inductive reasoning and invariants. This time we will deep dive into how to implement invariants properly. Let’s start!

## New Concepts Associated With Invariants

**Pre-conditions Vs. Valid States**

In the previous lesson (lesson 7), we mentioned the over-approximating state of CVT for reaching full coverage. This can bring some false negatives due to infeasible states that can never occur at runtime. To address this, we use pre-conditions to restrict the set of reachable states.

However, we need to be cautious when manipulating pre-conditions. Partial coverage inside a rule can be successfully verified, but it might hide a real bug. Partial coverage occurs when the `require()` statements are too strong and lead to under-approximation. In such cases, we might consider redefining the specification of the rule.

**requireInvariants**

To avoid mistakes, the concept of `requireInvariant` was introduced. This allows us to require an invariant. As we have seen in the previous lesson, invariants cover more than just the reachable states. Normally, we will not have any problem with the coverage area.

```cpp
invariant exampleInvariant(uint arg1, address arg2, ...)
exp

rule exampleRule(bytes32 arg1, uint arg2, address arg3, ...)
{
    requireInvariant exampleInvariant(arg2, arg3);

    ...

    assert exp2;
}
```

**Preserved Blocks**

Another concept we want to introduce is the preserved block for invariants. The preserved block is a component that allows us to insert a handful of pre-conditions after the post-constructor assert and before calling an arbitrary function, right next to the assumption of the invariant’s expression.

This concept appears useful when an invariant needs to rely on other invariants or when an invariant depends on the environment of the EVM (i.e., the block, msg.value, etc.).

**General Preserved Block**

A general preserved block applies the same pre-conditions for each function. The first curly brackets contain all the possible general preserved blocks or function-specific preserved blocks. A general preserved block begins with `preserved` followed by curly brackets:

```cpp
invariant example(address user, bytes32 id)
exp
{
    preserved with (env e2) // if you want to define an env variable
    {
        // otherwise you just need to write preserved
        {
            ...
        }
    }
}
```

**Preserved Block For A Specific Function**

Function-specific preserved blocks have the same logic as general preserved blocks, but they are specific for a single method (i.e., one specific function).

```cpp
invariant example(env e)
exp
{
    preserved transfer(address recipient, uint256 amount) with (env e3) {
        require amount > 0;
    }
}
```

**General And Function-Specific Preserved Blocks Together**

If you have both, then the method with a function-specific preserved block will not apply the generic preserved block.

```cpp
invariant example(bytes32 hashId, env e)
exp
{
    preserved with (env e2)
    {
        require hashId == 100;
    }

    preserved transfer(address recipient, uint256 amount) with (env e3) {
        require amount > 0;
    }
}
```

## Conclusion

Working with invariants is not as simple as I thought. I will need to check some real examples in [https://www.certora.com/dashboards/aave-dashboard/](https://www.certora.com/dashboards/aave-dashboard/) for a better understanding.