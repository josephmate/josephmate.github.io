---
layout: post
title: Determining If A Boolean Expression Can Be False
date: 2013-02-06 08:05
author: matejoseph
comments: true
categories: [Computer Science, Theory, Uncategorized]
---
My first hunch was that determining if a boolean expression can be false was NP complete. After trying to prove it, I came up for a proof that it's actually in P:

1. We have any boolean expression S
2. Using a well documented algorithm, we convert it to <a title="Conjunctive Normal Form" href="http://en.wikipedia.org/wiki/Conjunctive_normal_form">Conjunctive Normal Form</a>
3. Formula becomes (x<sub>1,1</sub> v x<sub>1,2</sub> v ... v x<sub>1,n</sub>) Λ (x<sub>2,1</sub> v x<sub>2,2</sub> v ... v x<sub>2,n</sub>) Λ ... Λ (x<sub>n,1</sub> v x<sub>n,2</sub> v ... v x<sub>n,n</sub>)
4. Since we are trying to solve a setting of x<sub>1,1</sub> to x<sub>n,n</sub> that results in false, that is the same thing as solving for a setting that results in true in the inverted expression
5. Now we have ¬((x<sub>1,1</sub> v x<sub>1,2</sub> v ... v x<sub>1,n</sub>) Λ (x<sub>2,1</sub> v x<sub>2,2</sub> v ... v x<sub>2,n</sub>) Λ ... Λ (x<sub>n,1</sub> v x<sub>n,2</sub> v ... v x<sub>n,n</sub>))
6. Applying DeMorgan's ¬(x<sub>1,1</sub> v x<sub>1,2</sub> v ... v x<sub>1,n</sub>) v ¬ (x<sub>2,1</sub> v x<sub>2,2</sub> v ... v x<sub>2,n</sub>) v ... v ¬(x<sub>n,1</sub> v x<sub>n,2</sub> v ... v x<sub>n,n</sub>))
7. Applying DeMorgan's again (¬x<sub>1,1</sub> Λ ¬x<sub>1,2</sub> Λ ... Λ ¬x<sub>1,n</sub>) v (¬x<sub>2,1</sub> Λ ¬x<sub>2,2</sub> Λ ... Λ ¬x<sub>2,n</sub>) v ... v (¬x<sub>n,1</sub> Λ ¬x<sub>n,2 Λ</sub> ... Λ ¬x<sub>n,n</sub>))
8. Now we can determine a setting that results in true, by using the ANDed group that doesn't contain a contradiction. If all the ANDed groups, contain contradictions, then there is no setting that results in true.
9. This setting from step 8 that results in true for ¬S will result in false for S

This came up at work because we were trying to determine if a boolean expression could become false. Knowing this would allow us to make a decision on how to handle the query.

If I have made an error, let me know! If you're confused about one of the steps, feel free to ask.

Cheers,
Joseph

<script src="https://utteranc.es/client.js"
        repo="josephmate/josephmate.github.io"
        issue-number="20"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>