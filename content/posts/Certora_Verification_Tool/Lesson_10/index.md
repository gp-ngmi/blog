---
title: "Certora Tutorials — Lesson 10"
date: 2023-07-22
draft: false
description: "Certora Tutorials — Lesson 10"
tags: ["Formal Verification"]
series: ["Formal Verification"]
series_order: 9
topics: ["Formal Verification"]
showAuthor: false
showAuthorsBadges : false
---


![Certora Tutorials](https://cdn-images-1.medium.com/freeze/max/800/1*4J8poHENIrtuVSFquWfmjQ.jpeg)

What’s up? We are almost at the end of the lessons. The subject of the day will be quick. Indeed, we will see a phenomenon that can be recurring when writing rules, but the core concept is simple. Let’s start!

## Vacuous Rules

When the Certora tool verifies a rule, CVT will follow the script inside the rule. Let’s create a situation where we call a function, and this one reverts every time regarding the parameters given. Then the test of the rule is useless, and we weren’t able to test the final assertion. For the CVT, there is nothing to signal because there is no assertion returning false. This is what we call a vacuous rule; it’s when a rule creates a false proof of the properties. Don’t be afraid; there are three methods for checking if a rule is vacuous.

### Methods

**Vacuity Check Rule**

At the beginning of your specification file, you can create the MethodsVacuityCheck rule that will call every function. Theoretically, almost all the tests created by the CVT should trigger the assertion false (depending on the logic of the function). If it’s not the case, then you should deep dive into the function and see why the function reverts to conclude if the function is vacuous or not.

```java
rule MethodsVacuityCheck(method f)
{
    env e; calldata arg args;
    f(e, args);
    assert false, "this method should have a non-reverting path";
}
```

### Adding `assert false`

This type of test is manual. Basically, you will add the line `assert false` at the end of your rule. We want that every path created by the CVT triggered the assertion false. It will confirm that the tests passed your original last assertion, indicating that the rule is not vacuous.

> By adding an `assert false` to the last line of a rule and expecting it to fail, we're asking the question - "Is there a path (set of values) that reached our original last assert and passed it?". When we fail the newly added assert, the answer to our question is "Yes - there is a set of values that reached and passed our last original assert". If all paths were to revert before reaching our `assert false`, this assert will have never been checked in the first place.
> [https://github.com/Certora/Tutorials/tree/master/10.Lesson_VacuousRules#adding-assert-false](https://github.com/Certora/Tutorials/tree/master/10.Lesson_VacuousRules#adding-assert-false)

### `--rule_sanity` Flag

I don’t want to say anything wrong, so here is the description of the flag from the Certora lesson.

> When running a verification with the `--rule_sanity` flag, the tool runs the rule twice - once as written, and again changing the original asserts to requires and adding an `assert false` at the end. It combines the results and indicates whether the `assert false` did not fail, meaning the rule is empty.
> [https://github.com/Certora/Tutorials/tree/master/10.Lesson_VacuousRules#--rule_sanity-flag](https://github.com/Certora/Tutorials/tree/master/10.Lesson_VacuousRules#--rule_sanity-flag)

### Recap

Both of the last vacuity checking methods are good; here is a recap:

- You need to modify the spec file when adding manually the assertion false and firing two runs (with and without assertion false).
- `--rule_sanity` mashes all the data together and saves the need to alter the code and run the tool multiple times. However, as this is still an evolving feature, its indicator can still be confusing.

## Other tips

This part will be like bullet points that I noticed during my exploration of the document that is not contained in this lesson.

- **Revert conditions**: There can be another way to handle revert. We can append to a function `@withrevert`, this modifier will allow you to check if the call of the function reverts or not. You will have a boolean called `lastReverted`, it will be true if the function reverts and false otherwise.

```java
rule insertRevertConditions(uint key, uint value)
{
    env e;
    insert @withrevert(e, key, value); //revert if value == 0
    bool succeeded = !lastReverted;

    assert value != 0 => succeeded;
}
```

- **Environment `e`**: In the `env e` of a rule, you can see if a function reverts or not with `e.msg.value`. A revert happens if `value = 1`, `0` otherwise.

## Conclusion

This lesson was quick but it is a foundation when you are verifying your rules. Recently, I have found a new article about the CVT: [Certora Technology White Paper](https://medium.com/certora/certora-technology-white-paper-cae5ab0bdf1). It describes very well the CVT and the core of formal verification.