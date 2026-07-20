+++
title = "Nix Part 7: Lazy Evaluation (Part 1)"
date = 2026-07-22
template = "post.html"
draft = true
+++

In the [previous post], we learned that `import` evaluates another Nix expression and returns its value. Throughout the series, we've focused on questions like:
* What does this expression evaluate to?
* What value does this function return?
* What does `import` give us?

But there's another equally important question that we haven't asked yet:
> **When does Nix evaluate an expression?** 

At first, this might sound like an implementation detail. As I continued learning Nix, however, I realised that it influences almost every part of the language. Understanding **when** expressions are evaluated turns out to be just as important as understanding **what** they evaluate to.

Before we look at how Nix behaves, let's start with a language many of us are already familiar with: Python.

# Eager evaluation in Python
Consider the following Python program:
```python
def expensive():
    print("Running expensive computation...")
    return 42

x = expensive()

print("Program finished.")
```
Before reading further, try to predict the output. The answer is:
```
Running expensive computation...
Program finished.
```
This might seem obvious, but it's worth stopping to think about **why** it happened.

The variable `x` is never used anywhere in the program. We don't print it, perform any calculations with it, or pass it to another function. Yet `expensive()` still runs. Why?

Because Python evaluates the right-hand side of an assignment immediately before storing the result in the variable. You can think of it like this:
```
expensive()
    │
    ▼
Compute the result
    │
    ▼
Store 42 in x
```
This evaluation strategy is called **Eager evaluation**.

Whenever Python encounters an expression, it evaluates it immediately, even if the result is never used later.

# Delaying the Computation
Now let's change the program slightly.
```python
def expensive():
    print("Running expensive computation...")
    return 42

x = lambda: expensive()

print("Program finished.")
```
Again, try to predict the output before running it.

This time, the output is simply:
```
Program finished.
```
Something interesting happened.

The message `"Running expensive computation..."` never appeared. Did Python suddenly become lazy? Not quite.

The difference is that `x` no longer stores the result of calling `expensive()`. Instead, it stores a **function** that knows *how* to call `expensive()` later. Nothing happens until we actually call the function.
```python
print(x())
```
Now the output becomes:
```python
"Running expensive computation..."
42
```
The computation didn't disappear. It was simply **delayed** until someone explicitly asked for the result.

# Thinking in Terms of Recipes
Consider these two Python examples. In the first example:
```python
x = expensive()
```
`x` contains the value produced by `expensive()`. The recipe has already been executed.

In the second example:
```python
x = lambda: expensive()
```
`x` doesn't contain the final value. Instead, it contains the recipe for producing that value. The recipe sits there doing nothing until someone calls the function.

This distinction between a **value** and a **recipe for producing a value** is exactly the intuition we'll need for understanding Nix.

# Nix Does This Automatically
In Python, we had to wrap our computation inside a function to delay it. Nix takes a different approach. The language itself delays evaluation automatically. Consider the following expression:
```nix
let
    x = 10 + 20;
in
100
```
Before reading further, try to predict the result. The expression evaluates to `100`. That's probably not suprising. What's more interesting is what **doesn't** happen. Although we defined `x`, Nix never evaluates: `10 + 20`. Why? Because nothing ever asks for its value.

You can think of the evaluation like this:
```
x = 10 + 20 
    │ 
    ▼ 
Store the recipe 
    │ 
    ▼ 
Is x needed? 
    │ 
    ▼ 
    No 
    │ 
    ▼ 
Don't evaluate it
```
Now let's make one small change.
```nix
let
    x = 10 + 20;
in
    x
```
This time, the final result depends on `x`. Since Nix needs to know the value of `x`, it finally evaluates: `10 + 20` producing: `30`. Notice what changed.

The expression defining `x` wasn't evaluated simply because it existed. It was evaluated because its value was required to produce the final result. This behavior is known as **lazy evaluation**.

Unlike Python, which evaluates expressions as soon as it encounters them, Nix waits until an expression's value is actually needed.

# My Mental Model
When I first encountered lazy evaluation, I tried to think of it as an optimization. Over time, I found a different mental model much more helpful.

Instead of imagining variables as boxes that already contain values, I imagine them as little cards containing **instructions** for producing those values.

Thinking of expressions as **recipes** rather than already-computed values completely changed how I understood evaluation in Nix. It also explains why Nix often appears to "ignore" parts of a program -- they simply aren't needed.

In the next post, we'll put this mental model to test using `throw`. We'll see that an expression can contain an error and yet never fail, simply because Nix never evaluates it.