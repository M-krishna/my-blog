+++
title = "Nix Part 7: Lazy Evaluation (Part 2)"
date = 2026-07-23
template = "post.html"

[taxonomies]
tags = ["nix", "lazy-evaluation", "functional-programming"]
+++

In the [previous post](https://fazeneo.in/posts/nix-lazy-evaluation-part-1/), we learned that Nix evaluates expressions **only when their values are needed**. That's a nice mental model, but how can we actually observe it?

So far, we've simply trusted that Nix delayed evaluation. In this post, we'll prove it.

# Observing Evaluation with `builtins.trace`
Nix provides a built-in function called `builtins.trace` that lets us see when an expression is evaluated.

At first, I found its name a little confusing. After using it for a while, I realized it's easier to think of it like this:
> **"Print a message when this expression is evaluated."**

The function takes two arguments:
1. A message to print.
2. A value to return.

For example: `builtins.trace "Hello!" 42` evaluates to:
```
trace: Hello!
42
```
Notice two things happened. First, the message was printed. Second, the expression still evaluated to `42`. This is important because `builtins.trace` **doesn't change the value of an expression.** It simply lets us observe when that expression is evaluated. Let's use it to revisit the example from Part 1.
```nix
let
    x = builtins.trace "Evaluating x" (10 + 20);
in
100
``` 
Before running it, try to predict what happens. Will the message be printed?

The answer is **no**. The result is simply: `100`. Nothing else appears. Why?

Because the value of `x` was never needed. Since `x` wasn't evaluated, neither was the call to `builtins.trace`. Now let's make one small change.
```nix
let
    x = builtins.trace "Evaluating x" (10 + 20);
in
x
```
This time the output becomes:
```
trace: Evaluating x
30
```
Now Nix needed the value of `x`. Evaluating `x` caused `builtins.trace` to print its message before returning the value `30`. This is the first time we've been able to see lazy evaluation happening.

# Using `throw` as Proof
`builtins.trace` lets us observe evaluation. `throw` lets us prove that evaluation is delayed. Consider the following expression:
```nix
let
    x = throw "Boom!";
in
100
```
At first glance, this looks like it should fail. After all, `x` contains an error. Surprisingly, the expression evaluates successfully. Because the value of `x` was never needed. Since Nix never evaluated `x`, it never encountered the call to `throw`. Now let's change the expression slightly.
```nix
let
    x = throw "Boom!";
in
x
```
This time, the result is:
```
error:
Boom!
```
What changed?

The only difference is that the final result now depends on `x`. To produce the final value, Nix has to evaluate `x`. Evaluating `x` immediately reaches:
```nix
throw "Boom!"
```
which stops evaluation with an error.

This pair of examples is one of my favourites because the only thing that changed was **whether the value was needed**. Nothing else.

# Function Arguments are Lazy Too
Lazy evaluation doesn't only apply to variables. Function arguments are lazy as well. Consider this function:
```nix
x: 42
```
Notice something interesting. The parameter `x` is never used. Now let's call the function like this:
```nix
let
    f = x: 42;
in
f (throw "Boom!")
```
Before reading further, try to predict what happens. Does the program fail? It doesn't.

At first, this suprised me.

In many programming languages, function arguments are evaluated before the function is called. If Nix behaved that way, `throw "Boom!"` would be evaluated immediately, and the program would fail before the function even started executing. Instead, Nix first looks at the function body. The body is simply: `42`. Since the argument is never used, Nix never evaluates: `throw "Boom!"`.

Now let's modify the function.
```nix
let
    f = x: x;
in
f (throw "Boom!")
```
This time, the result is:
```
error:
Boom!
```
Why? Because the function body now depends on `x`. To return `x`, Nix first has to evaluate it. That evaluation reaches `throw`, and the program stops with an error. 

# Putting Everything Together
Let's look back at the three tools we've used throughout this post.

`builtins.trace` showed us **when** evaluation happens.

`throw` showed us **what happens if an expression is evaluated**.

Lazy function arguments showed us that laziness isn't limited to variables -- it applies throughout the language.

Although these examples are all different, they're all governed by the same rule:
> **An expression is evaluated only when its value is required.**

Once I understood that rule, examples like these stopped feeling magical. Instead, they all became predictable.

# My Mental Model
Whenever I see a Nix expression, I no longer assume it will be evaluated immediately. Instead, I imagine Nix asking a simple question:
```
Do I actually need this value? 
    │ 
    ▼ 
    Yes ─────────► Evaluate it 
    │ 
    ▼ No ─────────► Leave it alone
```
Whether the expression contains:
* arithemetic
* a function call
* `builtins.trace`
* or even `throw`

the decision is always the same.

If the value isn't needed, Nix simply leaves the expression untouched.

In the next post, we'll look at another important consequence of lazy evaluation: **sharing**. We'll see that once Nix evaluates an expression, it remembers the result instead of evaluating the same expression again.