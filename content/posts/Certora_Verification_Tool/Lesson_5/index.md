---
title: "Certora Tutorials — Lesson 5"
date: 2023-07-22
draft: false
description: "Certora Tutorials — Lesson 5"
tags: ["Formal Verification"]
series: ["Formal Verification"]
series_order: 5
topics: ["Formal Verification"]
showAuthor: false
showAuthorsBadges : false
---

![Certora Tutorials](https://cdn-images-1.medium.com/freeze/max/800/1*oIZUl8R5UvbwCoi2CMACeg.jpeg)

How are you doing? I hope pretty well! Today we will discuss lesson 5.

## Additional Useful Flags

Follow the instructions on [ScriptExercise2](https://github.com/Certora/Tutorials/blob/master/05.Lesson_GettingFamiliarWithCVT/ScriptExercise2):

We are only using two flags on this part. Personally, the most useful flag is `--method`, as it allows you to avoid rerunning all your rules when you are debugging the specification files. About the flag `--send_only`, it can be useful if you don't want to analyze the result on your terminal.

Have a brief look at the entry on [CLI Options](https://certora.atlassian.net/wiki/spaces/CPD/pages/7340043/Certora+Prover+CLI+Options) in the documentation:

After a brief review of the documentation, here is a list of the flags that I think I will commonly use:

- [https://certora.atlassian.net/wiki/spaces/CPD/pages/7340043/Certora+Prover+CLI+Options#--debug](https://certora.atlassian.net/wiki/spaces/CPD/pages/7340043/Certora+Prover+CLI+Options#--debug): Of course, it could be interesting when I don't understand the error of the Certora Verification Tool.
- [https://certora.atlassian.net/wiki/spaces/CPD/pages/7340043/Certora+Prover+CLI+Options#--msg](https://certora.atlassian.net/wiki/spaces/CPD/pages/7340043/Certora+Prover+CLI+Options#--msg): Must be present in order to keep a trace of all your running tests and the modifications between them.
- `--loop`: We will have a lesson on this flag, and I’m not surprised. Indeed, loops in smart contracts can cause a lot of problems (gas limit, etc.).
- `--rule_sanity`: Again, we will have a lesson on this flag. This may be the most useful flag of the CLI options. We should not forget that if the rules are each vacuous, then they are all ineffective. A vacuous rule is when we can’t check the assertion at the end of the rule.

Have a look at the shell scripts lying in this directory:

I was able to globally interpret the shell scripts. These examples are very interesting for the future. When I checked the shell scripts, I noticed two options that were unknown to me: `--settings` and `-copyLoopUnroll`. Unfortunately, there is no information about them in the documentation. If you know anything related to these two parameters, please contact me.

## How Certora Prover Works:

[![Certora Prover Works](https://img.youtube.com/vi/c5ViO3Dpfqs/0.jpg)](https://www.youtube.com/watch?v=c5ViO3Dpfqs)

I have nothing to add to what Chandrakana Nandi said. The video enables a global comprehension of the Certora Verification tool. I will just make some little comments. My biggest problem is that we are not allowed to see the implementations of the tool. This leads me to a blurred area on whether I’m doing some formal verification. For example, I would love to look at the differences between runtime verification and them. However, we don’t need to master the implementation when we are using the tool. This is a personal problem. Also, I have never heard of TAC ([Three-address code](https://en.wikipedia.org/wiki/Three-address_code)). It seems pretty interesting; I will just post this link ([https://docs.certora.com/en/latest/docs/confluence/assert-splitting.html](https://docs.certora.com/en/latest/docs/confluence/assert-splitting.html)) if you want to deep dive into it. Finally, I figured out why lesson 3 is essential. Lesson 3 allows us to perceive plenty of things on how the CVT is working. Now I think that I can reuse this knowledge in some other areas.

## Conclusion

This lesson allowed us to be eased with the CLI and have a global comprehension of what is going on behind the scenes. I recommend you to check [Certora Technology White Paper](https://medium.com/certora/certora-technology-white-paper-cae5ab0bdf1). It may be way better than my articles.