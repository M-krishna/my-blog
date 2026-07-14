+++
title = "Nix Part 2: Understanding Functions and Function Application"
date = 2026-07-15
template = "post.html"
+++

In the [previous post](https://fazeneo.in/posts/everything-in-nix-is-an-expression/), we learned that everything in Nix is an expression and that expressions evaluate to values. We also briefly saw that a Nix file can contain a function expression:

```nix
x: x * 2
```
But what exactly does this syntax mean? How do we give a function a name? How do functions with multiple arguments work? And how do we apply a function to an argument?

In this post, we'll try to understand how functions work in Nix.

# Defining a function
Let's start with a simple function:
```nix
x: x * 2
```
A function in Nix consists of an argument, followed by a colon (`:`), followed by the function body. We can roughly read the above function as:
> Take an argument called `x` and evaluate `x * 2`

The part before the colon is the function argument: `x`. And the part after the colon is the function body: `x * 2`. Connecting this to what we learned in the previous post, the entire function is itself an expression: `x: x * 2`. When Nix evaluates this expression, the resulting value is a function. This is the important mental model: a function is also a value in Nix.

# Applying a Function
Defining a function by itself doesn't evaluate its body. To use the function, we need to apply it to an argument. For example:
```nix
(x: x * 2) 5
```
Here `x: x * 2` is the function, and `5` is the argument being passed to it.

Unlike languages such as Python or JavaScript, Nix doesn't use parentheses to apply a function:
```
double(5)
```
Instead, function application is written using whitespace:
```nix
double 5
```
In our example:
```nix
(x: x * 2) 5
```
the argument `5` is bound to `x`, so the function body becomes:
```nix
5 * 2
```
which evaluates to:
```nix
10
```

# Giving a Function a name
So far, our function doesn't have a name:
```nix
x: x * 2
```
Nix doesn't require special syntax such as `def` in Python or `function` in JavaScript to create a named function. Instead, we can create a function and bind it to a name using `let`:
```nix
let
    double = x: x * 2;
in
    double 5
```
Here:
```nix
x: x * 2
```
evaluates to a function, and that function is bound to the name `double`. We can then apply `double` to `5`:
```nix
double 5
```
which evaluates to `10`.

So `double` is not a special kind of function declaration. It is simply a name bound to a function value.

# Functions with multiple arguments
Let's say we want to write a function that multiplies two numbers. We might write:
```nix
x: y: x * y
```
At first, this might look like a function that takes two arguments. But functions in Nix technically take one argument at a time.

The expression:
```nix
x: y: x * y
```
can be read as:
```nix
x: (y: x * y)
```
The outer function takes `x` as its argument. Its body is another function:
```nix
y: x * y
```
That function takes `y` as its argument.

We can bind the function to a name:
```nix
let
    multiply = x: y: x * y;
in
    multiply 3 5
```
Function application associates from left to right, so:
```nix
multiply 3 5
```
can be understood as:
```nix
(multiply 3) 5
```
First: `multiply 3` returns a new function:
```nix
y: 3 * y
```
Then we apply that function to `5`:
```nix
(y: 3 * y) 5
```
which evaluates to `15`.

What looks like a function with multiple arguments is therefore a chain of functions, each taking one argument.

# The Connection to Lambda Calculus
The way functions are written in Nix may look unusual if you're coming from an imperative programming language, but the idea has its roots in **Lambda calculus**, a small mathematical system for expressing computations using functions.

In Lambda Calculus, an anonymous function that takes `x` and returns `x * 2` can be written as:
```
λx. x * 2
``` 
The equivalent function in Nix is:
```nix
x: x * 2
```
The syntax is different, but the structure is similar:
| Lambda Calculus | Nix |
|-----------------|-----|
| `λx. x * 2` | `x: x * 2` |
| `λx` is the parameter | `x` is the parameter |

**Function application** is also similar. In Lambda Calculus:
| Lambda Calculus | Nix |
|-----------------|-----|
| `(λx. x * 2) 5` | `(x: x * 2) 5` |

In both cases, `5` is applied to the function. This connection becomes even more apparent when we look at functions with multiple arguments.

Our Nix function:
```nix
x: y: x * y
```
has a similar structure to:
```
λx. λy. x * y
```

Neither of these is fundamentally a function that takes two arguments at once. The outer function takes one argument and returns another function:
```
λx. (λy. x * y)
```
Similarly, in Nix:
```nix
x: (y: x * y)
```

This also explains why:
```nix
multiply 3 5
```
can be understood as:
```nix
(multiply 3) 5
```

First, `multiply 3` returns a new function. That returned function is then applied to `5`.

> **NOTE:** You don't need to know Lambda Calculus to use Nix, but knowing this connection helped me understand why functions in Nix behave the way they do. What initially looked like unusual syntax started to feel much more natural when I recognized the same model of functions being applied one argument at a time. 

# Partial Application
Because `multiply 3` returns another function, we don't have to provide all the arguments at once. For example:
```nix
let
    multiply = x: y: x * y;
    triple = multiply 3;
in
    triple 5
```
Let's look at:
```nix
multiply 3
```
Since multiply is:
```nix
x: y: x * y
```
applying it to `3` gives us a new function that behaves like:
```nix
y: 3 * y
```
We bind this function to the name `triple`. Therefore:
```nix
triple 5
```
evaluates to: `15`

This is known as partial application. We apply a function to some of its arguments and get another function back.

# What about functions with no arguments?
In some programming languages, we can define a function that takes no arguments:
```python
def answer():
    return 42
```
Nix functions, however, take one argument.

If we don't need the argument, we can simply ignore it:
```nix
_: 42
```
The `_` is still a function argument. We are just indicating that we don't care about its value. For example:
```nix
(_: 42) "anything"
```
evaluates to: `42`.

The function receives `"anything"` as its argument but never uses it. Another pattern we may encounter is:
```nix
{}: 42
```
This is also not technically a function with no arguments. It is a function that expects an empty attribute set as its argument:
```nix
({}: 42) {}
```
which evaluates to `42`.

# Functions That Accept Attribute Sets
So far, our functions have accepted simple values:
```nix
x: x * 2
```
But a function can also accept an attribute set. For example:
```nix
{ name, age }:
    "${name} is ${toString age} years old"
```

We can bind it to a name and call it like this:
```nix
let
    describePerson =
        { name, age }:
            "${name} is ${toString age} years old";
in
    describePerson {
        name = "Krishna";
        age = 29;
    }
```
The function still receives only one argument. That argument is the entire attribute set:
```nix
{
    name = "Krishna";
    age = 29;
}
```
The pattern:
```nix
{ name, age }
```
extracts the `name` and `age` attributes so that they can be used inside the function body.

This style of function becomes especially important when reading Nix package definitions. We often see files that begin like this:
```nix
{ stdenv, fetchurl }:

stdenv.mkDerivation {
    # ...
}
```
The entire file evaluates to a function that accepts an attribute set containing `stdenv` and `fetchurl`.

# Default Arguments
When accepting an attribute set, we can provide a default value for an attribute:
```nix
{ name, age ? 18 }:
    "${name} is ${toString age} years old"
```
Now `age` is optional. For example:
```nix
let
    describePerson =
        { name, age ? 18 }:
            "${name} is ${toString age} years old";
in
    describePerson {
        name = "Alice";
    }
```
Since `age` wasn't provided, Nix uses the default value `18`.

# Accepting additional Attributes
By default, a function using an attribute-set pattern expects only the attributes it declares. For example:
```nix
{ name }: name
```
If we want to allow additional attributes, we can use `...`:
```nix
{ name, ... }: name
```
Now we can pass:
```nix
{
    name = "Krishna";
    age = 29;
    city = "Tumkur";
}
```
The function extracts `name` and allows the additional attributes to be present.

# Functions are Values
The main idea I want to remember from this post is that functions are values in Nix. The expression:
```nix
x: x * 2
```
evaluates to a function. That function can be bound to a name:
```nix
double = x: x * 2;
```
It can be applied to a value:
```nix
double 5
```
It can return another function:
```nix
x: y: x * y
```
And it can accept complex values such as attribute sets:
```nix
{ name, age }: ...
```

Once I started thinking of functions as values rather than special declarations, function syntax in Nix became much easier to understand.