+++
title = "Nix Part 6: Understanding import"
date = 2026-07-21
template = "post.html"
+++

In the [previous post](https://fazeneo.in/posts/understanding-paths/), we learned that paths are their own data type in Nix. While that may have seemed like a small detail at first, paths become much more useful when we start importing code from other files.

Up until now, every example in this series has lived inside a single `.nix` file. That's perfectly fine for learning, but real Nix projects are usually split across multiple files. Just like in any other programming language, keeping everything in one file quickly becomes difficult to maintain.

This is where `import` comes in.

Unlike languages such as Python or JavaScript, `import` in Nix doesn't "load a module" in the traditional sense. Instead, it follows a much simpler idea:
> **`import` evaluates another Nix expression and returns its value.**

That single sentence is enough to understand almost every example we'll see in this post. Let's get started.

# Every Nix File is an Expression
One of the first things we learned in this series is that **everything in Nix is an expression**. That idea doesn't suddenly stop when we move to another file. Suppose we have a file called `number.nix` and that file contains the expression `100`. Notice that the file contains a single expression. Now consider another file called `main.nix` which contains:
```nix
import ./number.nix
```
When Nix evaluates this expression, it first evaluates `number.nix`. Since `number.nix` evaluates to `100`, the result of the import is simply `100`.

You can think of it like this:
```
number.nix
    │ 
    ▼
   100
    │ 
    ▼
import ./number.nix
    │ 
    ▼
   100
```
The important thing to remember is that `import` **returns whatever the imported file evaluates to.**

# Importing Attribute Sets
A Nix file doesn't have to return a number. Suppose `person.nix` contains:
```nix
{ name = "Alice"; age = 25; }
```
Then we can write:
```nix
let
    person = import ./person.nix;
in
    person.age
```
The imported file evaluates to an attribute set. Therefore `person` is simply another attribute set, exactly as if we had written it ourselves. Accessing `person.age` evaluates to `25`. This shows that imported value behave no different from values defined locally.

# Importing Functions
Since functions are values in Nix, a file can also evaluate to a function. Suppose `square.nix` contains: `x: x * x`. Now consider:
```nix
let
    square = import ./square.nix;
in
    square 6
```
The import evaluates to the function: `x: x * x`. So after the import, our code is conceptually equivalent to:
```nix
let
    square = x: x * x;
in
    square 6
```
Applying the function produces: `36`.

Once again, `import` simply returns whatever the file evaluates to.

# Importing a Function that takes an Attribute Set
Let's look at another example. Suppose `greet.nix` contains:
```nix
{ name }: "Hello ${name}"
```
Notice that this file still evaluates to a function.

The only difference is that the function expects an attribute set instead of a single value. We can call it like this:
```nix
(import ./greet.nix) {
    name = "Krishna";
}
```
The result is: `"Hello Krishna"`.

Nothing special happened here. The imported file returned a function. We immediately applied that function to an argument.

# Why does this work?
When I first saw code like this:
```nix
(import ./double.nix) 10
```
It looked strange. After thinking about it for a while, I realized that nothing magical was happening. Suppose `double.nix` contains: `x: x * 2`. The first thing Nix does is evaluate: `import ./double.nix`, which produces `x: x * 2`. Now our code becomes `(x: x * 2) 10`.

If you've studied lambda calculus or functional programming before, this should look familiar. We're simply applying the value `10` to a function. The evaluation looks like:
```
import ./double.nix 
    │ 
    ▼ 
x: x * 2 
    │ 
    ▼ 
(x: x * 2) 10 
    │ 
    ▼ 
    20
```
Understanding this was one of those moments where `import` suddenly becomes much less mysterious.

# Importing Lists
A file can also return a list. Suppose `data.nix` contains:
```nix
[ { name = "Neo"; } { name = "Luna"; } ]
```
Since the imported value is just a list, we can use every list operation we've already learned. For example:
```nix
builtins.map
    (pet: pet.name)
    (import ./data.nix)
```
evaluates to:
```nix
[ "Neo" "Luna" ]
```
Notice that we didn't need a `let` binding. `import ./data.nix` evaluates to a list, and that list is passed directly to `map`.

This reinforces one of the most important ideas in functional programming:
> If an expression evaluates to a value, we can use it anywhere a value is expected.

# A Different Way to Think about `import`
When I was first learning Nix, I tried to compare `import` with Python's `import` statement. That turned out to be the wrong mental model. Instead, I now think about it like this:
```
Nix file 
    │ 
    ▼ 
Evaluate the file 
    │ 
    ▼ 
Return the resulting value
```
That's all `import` does. It doesn't matter whether the file evaluates to:
* a number
* a string
* a list
* an attribute set
* or even a function

`import` simply returns that value.

# My Mental Model
The biggest takeaway from this post is that a **Nix file is just another expression**. If a file evaluates to: `100`, then importing it gives us: `100`. If it evaluates to:
```nix
{ name = "Alice"; }
```
then importing it gives us that attribute set. If it evaluates to a function:
```nix
x: x * x
```
then importing it gives us that function. In other words:
```
Nix file 
    │ 
    ▼ 
Expression 
    │ 
    ▼ 
Value 
    │ 
    ▼ 
import returns that value
```
Once I started thinking about `import` this way, the syntax becomes much easier to understand. Instead of remembering special rules, I simply ask myself one question:
> **What does this file evaluate to?**

Whatever the answer is, that's exactly what `import` returns.

In the next post, we'll explore another important feature of the Nix language: **lazy evaluation**, and see how Nix only evaluates expressions when their values are actually needed.