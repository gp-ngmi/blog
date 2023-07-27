---
title: "Certora Tutorials — Lesson 3"
date: 2023-07-22
draft: false
description: "Certora Tutorials — Lesson 3"
tags: ["Formal Verification"]
series: ["Formal Verification"]
series_order: 3
topics: ["Formal Verification"]
showAuthor: false
showAuthorsBadges : false
---

![Certora Tutorials — Lesson 3](https://cdn-images-1.medium.com/freeze/max/800/1*NbZ1WL8-Cw92i07JQ-mIxw.jpeg)

Hello everyone ! Today will be a quick recap about what I learned. This lesson is not about the certora verification language (CVL). The subject is SMT — Satisfiability Modulo Theories, one of the cores of the certora prover.

> The role of the SMT is to decide whether a property, expressed as a set of logic expressions, can be proved or disproved. - Lesson 3

I recommend having done Lesson 0 (more precisely [introduction to logic](http://web.mit.edu/gleitz/www/Introduction%20to%20Logic%20-%20P.%20Suppes%20(1957)%20WW.pdf)). We will only use [Z3 Playground](https://jfmc.github.io/z3-play/) for the exercise and this tab:

![Z3 Playground](https://cdn-images-1.medium.com/freeze/max/800/1*gEb3egCIGIQlGWQ_n-Ps3A.png)

> Image source: [Secureum quiz](https://docs.google.com/document/d/19lLWqTrm_bzDdY9Uk-LpTSkgabk9WGtuCTPLDzSoraM/edit)

Firstly, I will add a few comments on this [video](http://Constraint%20Solving%20for%20Program%20Analysis). The presentation is very clear, and I don't have a lot to say, even if the video lasts an hour:

- A verification is done by reducing the program and the desired property into a formula (propositional logic) understandable by an SMT (SAT Modulo Theory).
- When it's reduced to a formula, the resolution is done only by boolean formulas. Afterward, with some commands, you can have an example if the formula is satisfiable.

With the presentation of Prof. Mooly Sagiv, we can now do the exercises without any difficulties. Let’s start with the first one:

![Logic Puzzle 1](https://cdn-images-1.medium.com/freeze/max/800/1*pd6yH8cjgS_IkmDQiNBZPQ.png)

We implement the image into Z3 input format, and we add the command `get-model`. It provides a correct example of the puzzle:

```
(declare-const a Int)
(declare-const b Int)
(declare-const c Int)
(assert (= (+ a a ) 10))
(assert (= (+ (* a b) b) 12))
(assert (= (- (* a b) (* c a)) a))
(check-sat)
(get-model)
```

The number of the red triangle (variable c) is:

```
sat // =satisfiable -> it means there is a solution
(
  (define-fun a () Int  5)
  (define-fun c () Int  1)
  (define-fun b () Int  2)
)
```

Now we can go to the other puzzle. I have the result that the exercise wants, but not sure if this is what they wanted:

![Logic Puzzle 2](https://cdn-images-1.medium.com/freeze/max/800/1*6XuclWTAvRZbMQWLLnqwHw.png)

Here is my implementation:

```
(declare-const p Bool)
(declare-const q Bool)
(define-fun equivalent () Bool
  (= (=> (and p q) p) (=> p (and p q))))
(assert (not equivalent))
(check-sat)
```

Result:

```
unsat // **equivalent()** is *valid* precisely when **not //equivalent** is not satisfiable (unsatisfiable)
```

Let me explain, if we have `unsat`, it means that there is no solution (no example) where `(p ⇒q ) ⇔ ((p ⋀ q)⇔ p)` is False. **To conclude, the property is proved**.

## Conclusion

Through this lesson, we are now able to have an image of how the abstract mathematical contract (created by certora prover) is verified via the specification. It’s through the SMT solver represented by `assert()` inside a rule. I recommend you to check [Certora Technology White Paper](https://medium.com/certora/certora-technology-white-paper-cae5ab0bdf1). It may be way better than my articles.