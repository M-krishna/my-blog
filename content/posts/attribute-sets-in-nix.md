+++
title = "Nix Part 3: Understanding Attribute Sets"
date = 2026-07-17
template = "post.html"
draft = true
+++

In the [previous post](https://fazeneo.in/posts/functions-in-nix/), we learned about functions in Nix. While learning about functions that accept named arguments, we encountered syntax like this:
```nix
{ name, age }:
    "${name} is ${toString age} years old"
```

The `{ name, age }` part is an **attribute-set pattern** used by the function to extract attributes from the attribute set passed to it.

But what exactly is an attribute set?

Attribute sets are one of the most important data structures in Nix. If you read Nix code, you will encounter them everywhere: package definitions, function arguments, configurations, and many other places.

In this post, we'll look at how to create and work with attribute sets, and we'll try to understand why they are so important in Nix.

This is probably going to be the longest post I've written in this series so far, because there is quite a bit to cover when it comes to attribute sets. So, let's get started.

# Creating an Attribute Set
An attribute set is a collection of name-value pairs. For example:
```nix
{
    name = "Krishna";
    age = 29;
}
```
Here, `name` and `age` are the **attribute names**, and `"Krishna"` and `29` are their corresponding values. We can think of it as:
| Attribute name | Value |
|----------------|-------|
| name | "Krishna" |
| age | 29 |

An attribute set is itself an expression. Connecting this to what we learned in Part 1:
```nix
{
    name = "Krishna";
    age = 29;
}
```
evaluates to an attribute set. The values inside an attribute set don't have to be of the same type. For example:
```nix
{
    name = "Krishna";
    age = 29;
    isProgrammer = true;
    hobbies = ["programming" "reading"];

    pet = {
        name = "Zen";
        type = "cat";
    }
}
```
An attribute set can contain strings, integers, booleans, lists, functions, other attribute sets, and other Nix values.

# Accessing Attributes
We can access an attribute using a dot(`.`). For example:
```nix
let
    person = {
        name = "Krishna";
        age = 29;
    };
in
    person.name
```
This evaluates to: `"Krishna"`. Similarly `person.age` evaluates to `29`. The expression `person.name` can be read as:
> Get the attribute called `name` from the attribute set `person`.

# Nested Attribute Sets
Since an attribute set can contain another attribute set, we can build nested structures. For example:
```nix
let
    user = {
        profile = {
            firstName = "Krishna";
            lastName = "Murugan";
        };

        address = {
            city = "Tumkur";
            state = "Karnataka";
        };
    };
in
    user.profile.firstName
```
To access `firstName`, we follow the structure:
```
user 
 └── profile 
        └── firstName
```
Therefore `user.profile.firstName` evaluates to `"Krishna"`. Similarly `user.address.state` evaluates to `"Karnataka"`.

# Attribute Paths
Nix also provides a shorter way to define nested attribute sets. Instead of writing:
```nix
{
    user = {
        profile = {
            name = "Krishna";
        };
    };
}
```
We can write:
```nix
{
    user.profile.name = "Krishna";
}
```
Both describe the same nested structure. This syntax is known as an **attribute path**. Attribute paths become especially useful when working with deeply nested configurations.

# Checking Whether an Attribute Exists
Sometimes we may want to check whether an attribute exists before accessing it. Nix provides the `?` operator for this. For example:
```nix
let
    person = {
        name = "Krishna";
        age = 29;
    }
in
    person ? age
```
This evaluates to `true` whereas `person ? city` evaluates to `false`.

The `?` operator checks for the existence of an attribute without trying to access its value.

# Providing a Default Value
We can also provide a fallback value when an attribute doesn't exist using `or`. For example:
```nix
let
    person = {
        name = "Krishna";
    };
in
    person.age or 18
```
Since `age` doesn't exist, the expression evaluates to: `18`. If `age` were present, for example `{ age = 29; }`, the result would be: `29`.

This can be read as:
> Get `person.age`, or use `18` if the attribute doesn't exist.

# Recursive Attribute Sets
Consider the following attribute set:
```nix
{
    firstName = "Krishna";
    greeting = "Hello ${firstName}";
}
```
At first, we might expect `greeting` to be able to access the `firstName` attribute defined next to it.

However, attributes in a normal attribute set are not automatically brought into scope for the other attributes.

To allow attributes to refer to other attributes in the same set, we can use `rec`:
```nix
rec {
    firstName = "Krishna";
    greeting = "Hello ${firstName}";
}
```
Now `firstName` is available in the scope of `greeting`. The result would be:
```nix
{
    firstName = "Krishna";
    greeting = "Hello Krishna";
}
```
The `rec` keyword makes the attribute set  **recursive**, allowing its attributes to refer to attributes from the same set. For example:
```nix
rec {
    x = 10;
    y = x + 20;
}
```
Here, `y` evaluates to `30`.

# Merging Attribute Sets
Nix provides the `//` operator for merging two attribute sets. For example:
```nix
let
    a = {
        x = 1;
        y = 2;
    };

    b = {
        y = 100;
        z = 3;
    };
in
    a // b
```
The result is:
```nix
{
    x = 1;
    y = 100;
    z = 3;
}
```
Both attribute sets contain an attribute called `y`. When the same attribute exists on both sides, the value from the attribute set on the **right-hand side** wins. Therefore:
```nix
{ y = 2; } // { y = 100; }
```
results in: `{ y = 100; }`

A useful mental model is: `left // right`. Merge both attribute sets, with the right side overriding conflicting attributes from the left side.

# The `inherit` keyword
Consider the following code:
```nix
let
    name = "Krishna";
    age = 29;
in
{
    name = name;
    age = 29;
}
```
We have variables called `name` and `age`, and we create attributes with the same names.

Nix provides `inherit` as a shorter way to write this:
```nix
let
    name = "Krishna";
    age = 29;
in
{
    inherit name age;
}
```
The expression: `inherit name;` is roughly equivalent to: `name = name;`. Similarly, `inherit name age;` is roughly equivalent to:
```nix
name = name;
age = age;
```
This is useful when the attribute name and the variable name are the same.

# Inheriting Attributes from Another Attribute Set
We can also inherit attributes directly from another attribute set. Consider:
```nix
let
    person = {
        name = "Krishna";
        age = 29;
    };
in
{
    name = person.name;
    age = person.age;
}
```
Using `inherit`, we can write:
```nix
let
    person = {
        name = "Krishna";
        age = 29;
    };
in
{
    inherit (person) name age;
}
```
The expression: `inherit (person) name;` is roughly equivalent to: `name = person.name;`.

This gives us two useful mental models:
```nix
inherit x;
```
means roughly:
```nix
x = x;
```
And:
```nix
inherit (attrs) x;
```
means roughly:
```nix
x = attrs.x;
```

# The `with` keyword
The `with` keyword temporarily makes the attributes of an attribute set available by name inside an expression. For example:
```nix
let
    person = {
        name = "Krishna";
        age = 29;
    };
in
with person;
    name
```
Without `with`, we would write: `person.name`. Inside `with person;`, we can refer to `name` directly. Therefore the expression evaluates to `"Krishna"`. The general structure is:
```
with attributeSet;
    expression
```
The attributes from `attributeSet` become available while evaluating the expression that follows.

# An Interesting Detail About `with`
Consider this example:
```nix
let
    x = 100;

    attrs = {
        x = 5;
    };
in
with attrs;
    x + 1
```
At first, I expected the result to be: `6` because `attrs` contains: `x = 5`. However, the result is: `101`.

The existing lexical variable `x` from the `let` expression takes precedence over the `x` introduced by `with`. So `with` doesn't simply override variables that are already in scope.

This is one reason `with` can sometimes make code harder to understand. When reading:
```nix
with attrs;
    x
```
It may not always be immediately obvious where `x` comes from. Using explicit attribute access: `attrs.x` can often make the source of a value clearer.

# Attribute Sets as Function Arguments
In the previous post, we saw functions like:
```nix
{ name, age }:
    "${name} is ${toString age} years old"
```
It is important to distinguish this:
```nix
{
    name = "Krishna";
    age = 29;
}
```
from this:
```nix
{ name, age }:
    ...
```
The first is an **attribute-set value**. The second uses an **attribute-set pattern** as a function argument. When we call:
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
The function receives one argument:
```nix
{
    name = "Krishna";
    age = 29;
}
```
The pattern `{ name, age }` extracts `name` and `age` attributes from that argument. Even though the syntax looks similar, an attribute-set function pattern serves different purpose.

# Why Attribute Sets Matter in Nix
Attribute sets are used everywhere in Nix. For example, a package definition might contain something like:
```nix
stdenv.mkDerivation {
    pname = "hello";
    version = "1.0";
    src = "...";
}
```
We don't need to understand `mkDerviation` yet. For now, the important thing to notice is that:
```nix
{
    pname = "hello";
    version = "1.0";
    src = "...";
}
```
is an attribute set being passed to a function. Similarly, Nix package files often begin with: `{ stdenv, fetchurl }:`. This is an attribute-set pattern used by a function.

Once I started recognizing the difference between attribute-set **values** and **attribute-set** patterns, real-world Nix code becomes easier to read.

# My Mental Model
An attribute set is a collection of named values:
```nix
{
    name = "Krishna";
    age = 29;
}
```
* We can access attribtutes `person.name`
* Nest attribute sets `user.profile.name`
* Merge them `a // b`
* Allow attributes to refer to each other using `rec`
* Avoid repeating names using `inherit`

Attribute sets are more than just another data structure in Nix. They are one of the primary ways Nix represents structured data, passes named values to functions, and describes configurations.

As we continue learning Nix and eventually start reading package definitions, we'll see attribute sets almost everywhere.