+++
title = "Nix Part 5: Understanding Paths"
date = 2026-07-20
template = "post.html"
+++

In the [previous post](https://fazeneo.in/posts/working-with-lists/), we looked at functions, attribute sets and lists. In this post, we're going to learn about another built-in data type in Nix: **paths**.

At first glance, paths may seem like ordinary strings. After all, both of them can look very similar. For example: `./main.nix` and `"./main.nix"` look almost identical. However they are **not** the same thing.

By the end of this post, we'll understand what paths are, how they're different from strings, and why Nix treats them as a separate data type.

Let's get started.

# Paths are their own Data Type
One thing that surprised me when learning Nix was that a path isn't just a string. Consider these two expressions:
```nix
./main.nix

"./main.nix"
```
The first one is a **path**. The second one is a **string**.

We can verify this using `builtins.typeOf`:
```nix
builtins.typeOf ./main.nix
```
return `"path"`. Whereas:
```nix
builtins.typeOf "./main.nix"
```
returns `"string"`.

Although they may look similar, Nix understands them differently. 

# Writing Relative Paths
Most of the time we'll refer to files relative to the current file.

Suppose our project looks like this:
```
project/
    ├── main.nix 
    └── utils/
        └── math.nix
```
From `main.nix`, we can refer to `math.nix` using: `./utils/math.nix`. The `./` means:
> "Start from the directory containing the current file."

Similarly, `..` means:
> "Go to the parent directory."

For example:
```
project/ 
├── common/ 
│   └── util.nix 
└── examples/ 
    └── main.nix
```
From `examples/main.nix`, we can reach `util.nix` using:
```nix
../common/util.nix
```

# Paths can be Combined
Suppose we have:
```nix
let
    configs = ./configs;
in
...
```
We can create another path by appending a string:
```nix
configs + "./dev.json"
```
Conceptually, this becomes:
```nix
./configs/dev.json
```
Notice that the result is still a **path**, not a string. Nix allows adding a string to a path because it extends the existing path. However, adding one path to another doesn't make much sense:
```nix
./configs + ./other
```
There isn't a well-defined way to combine two complete paths together, so Nix doesn't allow it.

# Why does Nix have a Path Type?
At first, I wondered why Nix doesn't simply use strings for everything. The answer is that a path represents something different. A string is simply text. A path represents the location of a file or directory.

By distinguishing between the two, Nix knowns when we're referring to files rather than ordinary text. This becomes especially important later when we start importing files and building packages.

# My Mental Model
The biggest takeaway from this post is that **a path is its own data type.** Although paths and strings may sometimes look similar, they serve different purposes.
```nix
"./main.nix"
```
is just a string. Whereas:
```nix
./main.nix
```
is an actual path value understood by Nix.

In the next post, we'll see why this distinction matters, because we'll start using paths together with one of Nix's most useful language features: `import`.