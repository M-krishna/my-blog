+++
title = "Nix Part 1: Everything in Nix is an expression"
date = 2026-07-13
template = "post.html"
+++

In this blog post, we are going to look at expressions in the Nix programming language. We will try to understand what it means when we say *everything is an expression* in Nix.

# What is Nix?
Nix is a declarative, dynamically typed, functional programming language. It is also lazily evaluated, similar to Haskell. The language is primarily used to describe packages, builds, development environments, and system configurations. 

In this post, I won't be going into how Nix is used to build packages. Instead, I'll focus on the programming language itself -- specifically *expressions*. 

By the end of this post, you'll have a basic understanding of expressions and expression evaluation in Nix.

# What are Expressions?
To put it simply, **an expression is a piece of code that can be evaluated to produce a value.**

If you come from an imperative programming language such as C, Python, or JavaScript, you may have heard about **statements vs expressions**.

A statement performs an action or controls the execution of a program. Let's look at a few examples in Python.

### Example for statements
```python
name = "Bob"

if name == "Bob":
    print("Hello Bob")

for i in range(10):
    print(i)
```

Expressions, on the other hand, evaluates to values.

### Example for expressions
```python
10 + 5 * 2 # evaluates to 20

count + 1 # evaluates to a number

age >= 18   # evaluates to either True or False
```

This distinction becomes important when learning Nix because, unlike many imperative languages, Nix is an expression-oriented language.

# Expressions in Nix
As the title of this blog post suggests, everything is an expression in Nix. Let's took a look at few examples.
```nix
"Hello"             # evaluates to a string
42                  # evaluates to an integer
[1 2 3]             # evaluates to a list
{ name = "Bob"; }   # evaluates to an attribute set
```

Each of these examples above is an expression by itself and evaluates to a value. Expressions can also perform computations.

`10 + 5`

When Nix evaluates this expression, it produces the value `15`. Similarly:

`10 > 5`

evaluates to true.

So an expression doesn't have to be a value written directly in the code. It can also be a computation that produces a value when evaluated.

# Combining Expressions
Expressions can contain other expressions. Let's look at `let` expression:
```nix
let
    x = 10;
    y = 20;
in
x + y
```
At first, if you're coming from an imperative programming language, you might look at `x = 10` and `y = 20` as statements that are executed one after another. But that's not how I find it useful to think about this code in Nix. 

The entire `let ... in ... ` is a single expression.

The `let` part introduces local variables that can be used in the expression after `in`. In this example, the expression after `in` is:

`x + y`

Since `x` is `10` and `y` is `20`, this evaluates to `30`. Therefore, the entire `let ... in` expression evaluates to `30`.

We can visualize it like this:
```nix
let
    x = 10;
    y = 20;
in
    x + y
      ↓
      30
```
This was an important idea for me when learning Nix: `let` is not a standalone statement. It is part of a larger expression that eventually evaluates to a value.

# A Nix file contains an Expression
The same idea applies to an entire `.nix` file. For example, a file called `main.nix` could contain just:

`10 + 20`

Evaluating this file produces:

`30`

The file could instead contain an attribute set:
```nix
{
    name = "Bob";
    age = 35;
}
```

In that case, evaluating the file produces an attribute set. It could even contain a function:

```nix
x: x * 2
```

In this case, evaluating the file produces a function. This gives us a useful mental model:

```
Nix file
    ↓
evaluate the expression
    ↓
value
```
The resulting value doesn't have to be a number or a string. It could be a list, an attribute set, a function, or some other Nix value.

# A mistake I made while learning Nix
This idea became clearer to me when I tried to write something like this in a file:
```nix
let
    square = x: x * x;
in
    square 6

let
    multiply = x: y: x * y;
in
    multiply 3 5
```
I expected Nix to evaluate the first piece of code and then continue with the second, similar to how a program written in an imperative language can contain multiple statements that are executed one after another.

Instead, I got a syntax error.

The problem is that I had written two separate `let ... in` expressions next to each other without combining them into a single expression. One way to write this is:
```nix
let
    square = x: x * x;
    multiply = x: y: x * y;
in
{
    squared = square 6;
    multiplied = multiply 3 5;
}
```
Now the entire file contains a single `let ... in` expression. The expression after `in` evaluates to:
```nix
{
    squared = 36;
    multiplied = 15;
}
```
And that attribute set becomes the value of the entire `let ... in` expression.

# Expressions can be Nested
Since expressions produce values, they can be used inside other expressions. For example:
```nix
{
    result = 10 + 20;

    doubled =
        let
            x = 5;
        in
            x * 2;
}
```
Here, `10 + 20` is an expression that evaluates to `30`. The `let ... in` expression evaluates to `10`. Those values then become the values of the `result` and `doubled` attributes. The entire attribute set is itself another expression.

This is how larger Nix programs are built: smaller expressions are combined to form larger expressions.

# My Mental Model
When reading Nix code, I find it useful to ask:
> **What value does this expression evaluate to?**

A number evaluates to a number. An arithmetic expression evaluates to the result of the computation. A `let ... in` evaluates to the value of the expression after `in`. A function expression evaluates to a function. Even the contents of `.nix` file evaluate to a value.

The mental model I use:
```
expression
    ↓
evaluation
    ↓
value
```

This idea becomes important as we learn more of the Nix language because the different constructs we encounter are not isolated statements executed one after another. They are expressions that can be combined to build larger expressions.