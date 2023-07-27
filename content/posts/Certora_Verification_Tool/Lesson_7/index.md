---
title: "Certora Tutorials — Lesson 7"
date: 2023-07-22
draft: false
description: "Certora Tutorials — Lesson 7"
tags: ["Formal Verification"]
series: ["Formal Verification"]
series_order: 7
topics: ["Formal Verification"]
showAuthor: false
showAuthorsBadges : false
---


![Certora Tutorials](https://cdn-images-1.medium.com/freeze/max/800/1*bawS9h2_UU11x5Xcg_AMnw.jpeg)

Bonjour ! Comment allez-vous ? The lesson of the day will once again include a lot of theory. Don’t worry, it’s interesting. Let’s start !

## Inductive Reasoning

**Pre-conditions and post-conditions**

- **Blog [post](https://medium.com/@mlbors/preconditions-and-postconditions-5913fc0fcdaf) by Mátyás Lancelot Bors**

In this post, Mátyás Lancelot Bors offers a definition for pre-conditions and post-conditions.

> A *precondition* is a condition, or a predicate, that must be true before a method runs for it to work. In other words, the method tells clients, “this is what I expect from you”. So, the method we are calling is expecting something to be in place before or at the point of the method being called. The operation is not guaranteed to perform as it should unless the *precondition* has been met.

> A *postcondition* is a condition, or a predicate, that can be guaranteed after a method is finished. In other words, the method tells clients, “this is what I promise to do for you”. If the operation is correct and the *precondition(s)* met, then the *postcondition* is guaranteed to be true.

It’s quite basic, I will complete these definitions for the CVT. A post-condition will confirm that the final assertion is true and so the property is verified. A pre-condition will create the environment for verifying a rule. If we have `require(number != 0)`, automatically, the value of number will be different from 0. We will not have a test with number = 0. It can be problematic because if someone managed to put `number = 0`, we didn’t test this use case.

- **A [lesson](https://web.mit.edu/6.031/www/fa17/classes/06-specifications/) on specifications in MIT University**

![Slide 3](https://cdn-images-1.medium.com/freeze/max/800/1*H7JHewHMFOI1wZj3uaP97w.png)

During your youth, during math class, you probably faced with mathematical induction. For proving that a property is true, then you need to prove it at initialization (n=0). Furthermore, for property(n), then we have property(n+1).

![Slide 4](https://cdn-images-1.medium.com/freeze/max/800/1*gK4OBsGuVVhaFPLrXy1Izg.png)

We will see a concrete example:

![Slide 6](https://cdn-images-1.medium.com/freeze/max/800/1*Vq35-Uv2d1JCVd8zKSoblQ.png)

After a quick check at the scheme, it’s obvious that the ball will never go to D if it starts at player A. But how do we prove that?

![Slide 9](https://cdn-images-1.medium.com/freeze/max/800/1*Ff5Bw7uanXYw8rQehPlqiQ.png)

James Wilcox exposes a suggestion for proving by induction, the initialization is good. However, there is nothing correct in the development because it does not cover all the cases that are possible. I have circled in red the area of the second bullet point. In our case, it’s possible to have X(m) = B, and so X(m+1) = D. So this proof is wrong. In the next slide, we will expand our unreachable area (so we hardened the property) with the intention of creating a good proof.

![Slide 14](https://cdn-images-1.medium.com/freeze/max/800/1*fV3PE_3aBPeWqE5yaAUqpQ.png)

If we centered what we see just above on a smart contract, we can imagine the players like valid states and arrows as state transitions (like we viewed in lesson 6). The goal of induction reasoning is to prove that no matter the current valid state and the state transition, your property will always be true.

During the video, James Wilcox presents a picture that abstracts what we see. We have a rectangle representing literally all the states of the system. Those include over-approximation. In the system state space, we can imagine the total of balances = 10, and the balance of John = 100. In reality, this is merely impossible, but this belongs to the state of the system. The initial state is represented in purple, it’s the state after the call of constructor() or initialize(). The blue represents the reachable states determined by the logic inside the smart contracts. In red, the possible state that we don’t want to be accessible.

![Slide 15](https://cdn-images-1.medium.com/freeze/max/800/1*1hSMMHkqdLogMCN3X4q44Q.png)

Now we will add in green, something called invariants. The next part of the article is about them. Basically, this is an over-approximation of the reachable states to be guaranteed that all the states inside invariants respect properties. These are the high-level properties.

TR: transition state

![Slide 16](https://cdn-images-1.medium.com/freeze/max/800/1*iLd2HzcaQVtlWoxicM5aXw.png)

When we are using the Certora Verification Tool, the tool reverts a counterexample for a rule or an invariant, also called CTI. I will let the next slide

 complete the rest.

**Invariants and their implementation in CVL**

An invariant is something that is always true and won’t change. An invariant is a combined *precondition* and *postcondition*. It has to be valid before and after a call to a method.

- They verify the state of the system at a single point in time.
- In the CVT, we will use invariants for applying inductive reasoning.

Syntax of an invariant:

```cpp
invariant inv_number(uint256 a, uint256 b)
a > b // your assertion
```

You don’t need to put the invariant into the rules. The CVT will automatically call the invariant at the beginning and the end of the rule. The invariant is also checked on the constructor.

```java
rule invariant_init(method f, env e, uint256 a, uint256 b)
{
    call constructor();
    assert inv_number(a, b); //invisible inside a real rule
}

rule invariant_body(method f, env e, uint256 a, uint256 b)
{
    require inv_number(a, b); //invisible inside a real rule
    f(e, args);
    a = getA();
    b = getB();
    assert inv_number(a, b); //invisible inside a real rule
}
```

We will deep dive into the implementation of invariants in the next lesson.

## Conclusion

The inductive reasoning and invariants are the core concepts of high-level properties. Personally, I have a better understanding of the test area concept.