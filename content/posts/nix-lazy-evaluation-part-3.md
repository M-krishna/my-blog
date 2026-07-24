+++
title = "Nix Part 7: Lazy Evaluation (Part 3)"
date = 2026-07-24
template = "post.html"

[taxonomies]
tags = ["nix", "lazy-evaluation", "functional-programming"]
+++

In the [previous post](https://fazeneo.in/posts/nix-lazy-evaluation-part-2/), we used `builtins.trace` and `throw` to observe lazy evaluation in action. We learned that Nix only evaluates an expression when its value is actually needed. That naturally raises another question. Suppose an expression **is** needed more than once. Does Nix evaluate it every single time?

The answer is **no**. Once Nix evaluates an expression, it remembers the result **for as long as that result is still needed during evaluation**, allowing it to be reused whenever the same value is required.

Let's see why this matters.

# One Expression, Two Uses
Consider the following expression.
```nix
let
    x = builtins.trace "Evaluating x" (10 + 20);
in
x + x
```
Before running it, try to predict the output.

Since `x` appears twice, you might expect the trace message to appear twice as well. Something like:
```
trace: Evaluating x
trace: Evaluating x
60
```
But that's not what happens. The actual output is:
```
trace: Evaluating x
60
```
The message is printed only once. Even though the value of `x` is used twice, the expression defining `x` is evaluated only once. Why?

# A Different Evaluation Strategy
Imagine a language that behaved differently. The first time `x` was needed, it would evaluate:
```nix
builtins.trace "Evaluating x" (10 + 20)
```
The second time `x` was needed, it would go back and evaluate exactly the same expression again. You could picture it like this:
```
Need x 
    │ 
    ▼ 
Evaluate expression 

Need x again 
    │ 
    ▼ 
Evaluate expression again
```
This evaluation strategy has a name. It is called **call by name**.

In a call-by-name language, evaluation is delayed until a value is needed, but the language **does not** remember previously computed results. Every use may trigger another evaluation.

# Nix Takes One More Step
Nix behaves differently. The first time `x` is needed, Nix evaluates the expression:
```nix
builtins.trace "Evaluating x" (10 + 20)
```
producing: `30`

Instead of discarding that result, Nix remembers it. The next time `x` is needed, Nix simply reuses the value it already computed. You can think of it like this:
```
Need x? 
    │ 
    ▼ 
Evaluate expression 
    │ 
    ▼ 
Remember result 
    │ 
    ▼ 
Need x again? 
    │ 
    ▼ 
Reuse remembered value
```
This evaluation strategy is called **call by need**.

The computation is delayed until the value is needed, and once it has been computed, it is never evaluated again.

# Comparing the Evaluation Strategies
We've now encountered three different evaluation strategies.

## Call by Value
This is the strategy used by languages like Python. Expressions are evaluated immediately.
```python
x = expensive()
```
Even if `x` is never used, `expensive()` still runs.

## Call by Name
Evaluation is delayed until the value is needed. However, every use may evaluate the original expression again. Nothing is remembered.

### Call by Need
This is the strategy used by Nix. Evaluation is delayed until the value is needed. After the first evaluation, the result is remembered and reused everywhere else.

The three strategies can be summarized like this:
| **Evaluation Strategy** | **When is the expression evaluated?** | **Is the result remembered?** |
|-------------------------|---------------------------------------|-------------------------------|
| Call by Value | Immediately | Yes |
| Call by Name | Only when needed | No |
| Call by Need | Only when needed | Yes |

Notice that **call by name** and **call by need** are almost identical. The only difference is whether the computed result is remembered.

# Why Sharing Matters
When I first learned about sharing, I assumed it was simply a performance optimization. Over time, I realized it's actually part of Nix's evaluation strategy.

Without sharing, Nix would behave like a call-by-name language. The same expression could be evaluated repeatedly whenever its value was required. Instead, Nix evaluates an expression once, remembers the result for as long as it is still needed during evaluation, and shares that value with every future reference. That's why the trace message appeared only once in our earlier example.

At this point, I had another question.
> **If Nix remembers evaluated values, won't memory usage keep growing forever?**

Fortuantely, the answer is **no**.

A remembered value only stays alive while something can still use it.

For example, while evaluating `x + x`, the value of `x` must remain available because the second `x` still needs it. Once the entire expression has finished evaluating, nothing can reference `x` anymore. At that point, the remembered value is no longer needed and becomes **eligible for garbage collection**.

You can think of the lifetime of a value like this:
```
Need the value? 
    ↓ 
Evaluate it 
    ↓ 
Remember the result 
    ↓ 
Can something still use it?
        /           \
       Yes          No
        |            |
    Keep it         Eligible for garbage collection
```

Notice the wording **eligible for garbage collection**. The runtime doesn't necessarily reclaim the memory immediately. It simply means the value is no longer reachable and can be reclaimed whenever appropriate.

One more thing I want to reiterate here which we already know is, If an expression is never needed, it is never evaluated in the first place.
```nix
let
    x = builtins.trace "Evaluating x" (10 + 20);
    y = 5;
in
y
```
Since nothing ever asks for `x`, the trace message is never printed, `10 + 20` is never computed, and there is no computed value that needs to be remembered or shared. This is one of the biggest advantages of lazy evaluation: work is performed only when it is actually needed.

# A Connection to Functional Programming
If you've studied lambda calculus or functional programming before, you may have encountered terms like **call by value**, **call by name**, and **call by need**. We've now seen all three. Python is an example of call by value. A hypothetical lazy language without sharing would use call by name. Nix uses call by need. Thankfully, you don't need to memorize these names.

If you understand **when** evaluation happens and **whether** the result is remembered, the terminology becomes much easier to remember.

# Looking Back
Over the last three posts, we've gradually built a mental model of lazy evaluation. In Part 1, we learned that expressions are evaluated only when their values are needed. In Part 2, we observed that behaviour using `builtins.trace` and `throw`. Finally, in this post, we learned that Nix remembers computed values instead of evaluating the same expression repeatedly. Together, these ideas explain much of Nix's behaviour. Expressions don't evaluate simply because they exist. They evaluate because someone needs their value. And once that value has been computed, Nix remembers it for the rest of the evaluation **or until nothing can reference it anymore**.

# My Mental Model
Whenever I read a Nix expression now, I imagine it going through three simple stages.
```
Expression
     ↓
Recipe
     ↓
Needed?
 ┌───┴────┐
 │        │
No       Yes
 │        │
 │        ▼
 │    Evaluate
 │        │
 │        ▼
 │ Remember Result
 │        │
 │        ▼
 │ Still Needed?
 │    ┌───┴────┐
 │    │        │
 │   Yes      No
 │    │        │
 └────┴──► Reuse
          │
          ▼
 Eligible for
Garbage Collection
```
Thinking about evaluation this way has made Nix feel much more predictable.

Instead of asking:
> **"Why wasn't this expression evaluated?"**

I ask:
> **"Was its value actually needed?"**

And if the answer is yes, I ask one more question:
> **"Has Nix already computed it before?"**

Those two questions explain most of the examples we've seen throughout this series.

With lazy evaluation behind us, we're finally ready to move on to one of the most important concepts in Nix: **derivations**. Up until now, we've been learning the Nix language itself. Next, we'll begin learning what the language was designed to describe.