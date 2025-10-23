---
title: "Aaron Swartz"
date: 2022-05-15
description: "Co-founder Reddit"
tags: ["strategy", "lifestyle"]
draft: true
---

{{< lead >}}
Internet's Own Boy
{{< /lead >}}

There are many problems where computer scientists employ interesting algorithms to solve them. One such problem which is often seen in real life is the problem of [Optimal Stopping](https://en.wikipedia.org/wiki/Optimal_stopping).

Also known as [The Secretary problem](https://en.wikipedia.org/wiki/Secretary_problem) (GFG article on the same- [here](https://www.geeksforgeeks.org/secretary-problem-optimal-stopping-problem/)), say you’re interviewing a group of applicants for a position, how do you maximize the chances of hiring the single best applicant in the pool? (Once a candidate is rejected, they are gone forever and cannot be recalled.)

Interestingly the dating variant of the problem asks how many people should you date before truly finding the `one` or deciding to settle down with? Tricky question but surprisingly probability has an answer for this- \\(\dfrac{1}{e} \\) or approximately 37%.

- Let's say you want to date \\( n\\) people (idk how you'd do that though)
- First date \\( \dfrac{n}{e}\\) without commiting to anyone of them, (here e is the base of the natural logarithm)
- And then ask the rest, one by one
- Now select the person outranking the candidates you've seen so far
- Thus maximizing the probability of going out with the best person
- Marry that person, 'cause they are the best match for you (according to probability)

By the way, do check out [The 37% Rule](https://cs.nyu.edu/~davise/Verses/ThirtySeven.html) verse by [Ernest Davis](https://cs.nyu.edu/~davise/). Man's got an enitre page of his amazing verses here- [Verses for the Information Age](https://cs.nyu.edu/~davise/Verses/) (found this awesome piece of literature accidentally)

Won't be going into details of this thing but you can check out [this](https://www.randomservices.org/random/urn/Secretary.html) for a proof, or [this paper](https://www2.math.upenn.edu/~ted/210F10/References/Secretary.pdf) for a more detailed historical outlook to the problem.

---
