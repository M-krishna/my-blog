+++
title = "Nix Part 4: Working with Lists"
date = 2026-07-19
template = "post.html"
draft = true
+++

In the [previous posts](https://fazeneo.in/posts/attribute-sets-in-nix/), we looked at expressions, functions and attribute sets. In this post, we're going to look at another important data structure in Nix: **lists**.

If you've used languages like Python or JavaScript, lists should feel familiar. A list is simply an ordered collection of values.

But creating a list is only the beginning. We often want to transform every element in a list, keep only the elements that satisfy a condition, or combine all the elements into a single result.

In Nix, we'll mainly use `map`, `filter` and `builtins.foldl'` for these operations.

By the end of this post, we'll have a mental model that looks something like:

* `map`     -> transform each element
* `filter`  -> keep some elements
* `foldl'`  -> combine elements into a result

So let's get started.

# Creating a List
A list in Nix is created using square brackets:
```nix
[ 1 2 3 4 5 ]
```
Unlike some other programming languages, elements in a Nix list are separated by **whitespace**, not commas.

For example, this is valid:
```nix
[
    1
    2
    3
    4
    5
]
```
A list is also an expression. Connecting this to what we learned in Part 1:
```nix
[ 1 2 3 ]
```
evaluates to a list value containing three element.

# Lists Can Contain Different Types
The elements of a Nix list don't have to be of the same type. For example:
```nix
[
    1
    "Krishna"
    true

    {
        name = "Neo";
        type = "Cat";
    }

    [ 1 2 3 ]
]
```
This list contains:
* an integer
* a string
* a boolean
* an attribute set
* another list

This is possible because Nix is dynamically typed.

# Accessing Elements
We can access an element from a list using `builtins.elemAt`. For example:
```nix
let
    animals = [
        "cat"
        "dog"
        "fox"
        "bear"
    ];
in
    builtins.elemAt animals 0
```
This evaluates to `"cat"`.

List indexing starts from `0`, so:
| index | value |
|-------|-------|
| 0 | "cat" |
| 1 | "dog" |
| 2 | "fox" |
| 3 | "bear" |

Therefore: `builtins.elemAt animals 2` evaluates to `"fox"`.

# Finding the length of a list
Nix provides `builtins.length` to find the number of elements in a list. For example:
```nix
builtins.length [ 10 20 30 40 50 ]
```
evaluates to: `5`.

Later in the post, we'll see that we can also calculate the length ourselves using `builtins.foldl'`. Doing that is unnecessary in real code when `builtins.length` already exists, but it is a useful exercise for understanding folds.

# Concatenating Lists
We can combine two lists using the `++` operator. For example:
```nix
[ 1 2 ] ++ [ 3 4 ]
```
evaluates to: `[ 1 2 3 4 ]`.

The `++` operator creates a new list contaning the elements from both lists.

# Transforming Lists with `map`
One of the most common things we want to do with a list is transform every element. For example, suppose we have: `[1 2 3 4 5]` and we want `[ 2 4 6 8 10 ]`. We can use `map`:
```nix
map (x: x * 2) [ 1 2 3 4 5 ]
```
The function: `x: x * 2` is applied to every element in the list. Conceptually we can think of it like this:
```
1 -> 2
2 -> 4
3 -> 6
4 -> 8
5 -> 10
```
A useful mental model is:
```
map
    function
    list
```
Or
```
map function list
```
`map` takes a function and a list, applies the function to every element, and returns a new list.

# `map` and Function Application
Since we learned about functions in Part 2, it is worth looking at the following expression more closely:
```nix
map (x: x * 2) [ 1 2 3 ]
```
Function application in Nix associates from left to right, so we can think of this as:
```nix
(map (x: x * 2)) [ 1 2 3 ]
```
First, `map (x: x * 2)` returns a function waiting for a list. We then apply that function to: `[ 1 2 3 ]`.

This is another example of the ideas we saw while learning about curried functions and partial applications. We can use this to create a reusable function:
```nix
let
    double = map (x: x * 2);
in
    double [ 1 2 3 4 5 ]
```
Here `map (x: x * 2)` returns a function that expects a list. We bind that function to `double`. Therefore `double [ 1 2 3 4 5]` evaluates to: `[ 2 4 6 8 10]`.

# Mapping Over Attribute Sets
Lists often contain attribute sets. For example:
```nix
[
    {
        name = "Krishna";
        age = 29;
    }

    {
        name = "Bob";
        age = 22;
    }

    {
        name = "Alice";
        age = 35;
    }
]
```
Suppose we only want the names:
```nix
[
    "Krishna"
    "Bob"
    "Alice"
]
```
We can use `map`:
```nix
map
    (person: person.name)
    [
       {
            name = "Krishna";
            age = 29;
        }

        {
            name = "Bob";
            age = 22;
        }

        {
            name = "Alice";
            age = 35;
        } 
    ]
```
For every `person`, we return `person.name`. The result is `[ "Krishna" "Bob" "Alice" ]`.

# Selecting Elements with `filter`
While `map` transforms every element, `filter` decides which elements should remain in the list. For example, suppose we have: `[ 1 2 3 4 5 6 ]`, and we want to keep only the even numbers. We can write:
```nix
builtins.filter
    (x: x - (x / 2) * 2 == 0)
    [ 1 2 3 4 5 6 ]
```
> **NOTE:** Nix doesn't have `mod` (`%`) operator.

The function passed to `filter` must return a boolean value. For each element, the result is conceptually:
* 1 -> false
* 2 -> true
* 3 -> false
* 4 -> true
* 5 -> false
* 6 -> true

Elements for which the function returns `true` are kept. The result is `[ 2 4 6 ]`.

A useful mental model is:
```
filter
    predicate
    list
```
A **predicate** is simply a function that returns `true` or `false`.

We could write the above example like this as well:
```nix
let
    mod = a: b: a - (a / b) * b;
    inp = [ 1 2 3 4 5 6 ];
    onlyEven = (x: mod x 2 == 0);
in
    builtins.filter onlyEven inp
```

# Filtering Strings
Suppose we have:
```nix
[
    "cat"
    "elephant"
    "dog"
    "hippopotamus"
]
```
and we want to keep only strings whose length is greater than `3`. We can write:
```nix
builtins.filter
    (animal: builtins.length animal > 3)
    [
        "cat"
        "elephant"
        "dog"
        "hippopotamus"
    ]
```
The result is:
```nix
[ "elephant" "hippopotamus" ]
```
Again, the function doesn't transform the values. It only decides whether each value should remain in the resulting list. This gives us an important distinction:
* `map`     -> **What should this element become?**
* `filter`  -> **Should this element remain?**

# Combining `filter` and `map`
We can combine list operations. Suppose we have:
```nix
let
    people = [
        {
            name = "Krishna";
            age = 29;
        }

        {
            name = "Alice";
            age = 22;
        }

        {
            name = "Bob";
            age = 35;
        }
    ];
in
...
```
Let's say we want the names of people older than `30`. First, we can filter the list:
```nix
builtins.filter
    (person: person.age > 30)
    people
```
This gives us:
```nix
[
    {
        name = "Bob";
        age = 35;
    }
]
```
Then we can map over the result:
```nix
map
    (person: person.name)
    (builtins.filter
        (person: person.age > 30)
        people)
```
The final result is: `[ "Bob" ]`.

We can read this from inside out:
```
people
    ↓
filter people older than 30
    ↓
map each person to their name
    ↓
[ "Bob" ]
```
The idea of combining small functions is common in functional programming. 

# Combining Elements with `builtins.foldl'`
`map` transforms elements.

`filter` selects elements.

But what if we want to combine all the elements into a single result?

This is where `builtins.foldl'` comes in. For example, let's calculate the sum of: `[ 1 2 3 4 ]`. We can write:
```nix
builtins.foldl'
    (acc: x: acc + x)
    0
    [ 1 2 3 4 ]
```
At first, `foldl'` may look more difficult than `map` or `filter`.

It takes three things:
```nix
builtins.foldl'
    function
    initial accumulator
    list
```
In our example, the *initial accumulator* is `0` and the function is `acc: x: acc + x`.

For every element, the function receives:
* the current accumulator
* the current element

We can think of the evaluation like this: `initial accumulator = 0`
* 0 + 1 = 1
* 1 + 2 = 3
* 3 + 3 = 6
* 6 + 4 = 10

The final result is `10`.

# Computing a Product with `foldl'`
We can also calculate the product of all the numbers.
```nix
builtins.foldl'
    (acc: x: acc * x)
    1
    [ 1 2 3 4 ]
```
The evaluation looks like: `initial accumulator = 1`
* 1 * 1 = 1
* 1 * 2 = 2
* 2 * 3 = 6
* 6 * 4 = 24

The final result is `24`.

Notice that we started with `1` instead of `0`. If we started with `0`: `0 * anything = 0`. So the final result would always be `0`.

The initial accumulator is therefore an important part of a fold.

# What Happens When the List is Empty?
Consider:
```nix
builtins.foldl'
    (acc: x: acc + x)
    100
    []
```
The result is `100`. Why?

A fold starts with the initial accumulator and processes each element in the list. But there are no elements in `[]`. Therefore, the folding function is never called. We start with: `100` and since there is nothing to process, we finish with: `100`.

This gave me a useful mental model:
> The accumulator is already the result before the list is processed. Each element gives us an opportunity to transform that result.

# Using `foldl'` to Calculate the Length
Earlier, we used: `builtins.length [ 1 2 3 4 ]`. But we can implement the same idea using `foldl'`:
```nix
let
    length' =
        list:
            builtins.foldl'
                (count: _: count + 1)
                0
                list;
in
    length' [ 10 20 30 40 50 ]
```
The result is: `5`. 

The folding function is `count: _: count + 1`. We don't care about the value of the current element, so we use `_`. For every element, we increase the count by `1`. Conceptually: `initial count = 0`.

* first element     -> 1
* second element    -> 2
* third element     -> 3
* fourth element    -> 4
* fifth element     -> 5

Again, we should use `builtins.length` in real code when all we want is the length of a list. But implementing it ourselves helps us understand how a fold works.

# Using `foldl'` to Build a List
A fold doesn't have to produce a number. The accumulator can be a list as well. For example,
```nix
builtins.foldl'
    (acc: x: acc ++ [x * 2])
    []
    [ 1 2 3 ]
```
Let's follow the accumulator:

* `initial accumulator = []`.
* process 1: `[] ++ [ 1 * 2 ]` = `[2]`
* process 2: `[2] ++ [ 2 * 2 ]` = `[2 4]`
* process 3: `[2 4] ++ [3 * 2]` = `[2 4 6]` 

The final result is: `[ 2 4 6 ]`.

This produces the same result as:
```nix
map (x: x * 2) [ 1 2 3 ]
```

That doesn't mean we should replace `map` with `foldl'`. If our intention is to transform every element, `map` communicates that intention much more clearly. But this example shows that a fold is more general. The accumulator can be almost any kind of value.

# Reversing a List with `foldl'`
Consider:
```nix
builtins.foldl'
    (acc: x: [ x ] ++ acc)
    []
    [ 1 2 3 4 ]
```
Let's follow the accumulator:

* `initial accumulator = []`
* process 1: `[1] ++ []` = `[1]`
* process 2: `[2] ++ [1]` = `[2 1]`
* process 3: `[3] ++ [2 1]` = `[3 2 1]`
* process 4: `[4] ++ [3 2 1]` = `[4 3 2 1]`

The result is: `[ 4 3 2 1 ]`.

The important difference is where we add the current element. This:
```nix
acc ++ [x]
```
adds the element to the end. Whereas:
```nix
[ x ] ++ acc
```
adds the element to the beginning.

# Using `foldl'` Like `filter`
We can even use a fold to build a list conditionally:
```nix
builtins.foldl'
    (acc: x:
        if x > 2
        then acc ++ [x]
        else acc)
    []
    [ 1 2 3 4 5 6 ]
```
For `1` and `2`, we return the accumulator unchanged. For `3`, `4`, `5`, `6`, we add the current element to the accumulator. The result is: `[ 3 4 5 ]`.

Again, in real code:
```nix
builtins.filter (x: x > 2) [ 1 2 3 4 5 6 ]
```
would communicate the intention more clearly. But implementing the same behavior with `foldl'` helps reveal what a fold can do.

# `map`, `filter`, and `foldl'`
At this point, my mental model is:
```nix
map
    transform every element
    list -> list
```

```nix
filter
    decide which elements remain
    list -> list
```

```nix
foldl'
    carry an accumulator through the list
    list -> some result
```

For example: 

`map (x: x * 2) [ 1 2 3 ]` asks:
> What should each element become?

`builtins.filter (x: x > 2) [ 1 2 3 ]` asks:
> Which elements should remain?

```nix
builtins.foldl'
    (acc: x: acc + x)
    0
    [ 1 2 3 4]
```
asks:
> How should each element update the accumulated result?

# My Mental Model
A list is an ordered collection of values: `[ 1 2 3 ]`.

* We can access an element: `builtins.elemAt list 0`
* We can find its length: `builtins.length list`
* We can combine lists: `a ++ b`
* We can transform elements: `map function list`
* We can select elements: `builtins.filter predicate list`
* We can combine elements into a result: `builtins.foldl' function initialValue list`

The most useful distinction for me is:
* `map`     -> transform
* `filter`  -> select
* `foldl'`  -> accumulate

These operations becomes even more useful when lists contain attribute sets, because we can combine the data structures and functions we've learned so far.

At this point, the individual pieces of Nix are starting to connect. Functions operate on lists, list contain attribute sets, and functions can extract or transform the values inside those attribute sets.

That is one of the things I'm starting to appreciate about learning a functional language, instead of thinking primarily in terms of instructions and loops, we start thinking about transforming values into other values.