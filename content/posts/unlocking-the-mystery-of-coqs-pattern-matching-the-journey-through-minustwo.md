+++
title = "Unlocking the Mystery of Coq's Pattern Matching: The Journey Through minustwo"
date = 2024-03-18
template = "post.html"
+++

I was learning Coq for fun, hoping someday that I would prove some theorems(JK). While learning Coq, specifically *pattern matching*, I stumbled upon this definition named ***minustwo***. As the name suggests, it subtracts 2 from the number you provide but with some cases. Let’s look at the definition to understand more.

> **DISCLAIMER:** I’m not an expert in Coq, I’m just experimenting
>

```coq
Definition minustwo (n: nat): nat :=
	match n with
	| O => O
	| S O => O
	| S (S m) => m
	end.
```

Let’s breakdown the above code. 

We have created a *definition* which in imperative languages we call them *functions*. The name of the definition is **minustwo,** which takes one argument *n* of type *nat* (short for Natural number).

The body of the definition contains a *match* statement. In Coq, we use *match* statement for *pattern matching*. You can think of *match* as *switch case* in other languages, but not exactly the same.

We have three cases:

1. If *n* is *O*, return *O*
2. If *n* matches with *S O*, return *O*
3. If *n* matches with *S (S m)*, return *m*

*O* represents capital “O” and not numeral “0”. In Coq, both represents the same thing. We can replace “O” with “0” as well.

But, what is *`S O`*? What exactly is *S* ? *S* is short for **Successor** which is a *constructor* in Coq(in-built). We can look at the signature of *S* by executing this command: `Check S.` Which gives us:

![Screenshot_2024-03-18_at_10-00-30_PM](https://mataroa.blog/images/9980440e.png)

This states that *S* takes in a natural number(nat) and returns a natural number(nat). So, 

- `S O.` returns 1
- `S 1.` returns 2
- `S 2.` returns 3
- and so on. We can check that by executing `Compute S n` , replace *n* which a natural number

![Screenshot_2024-03-18_at_10-02-32_PM](https://mataroa.blog/images/9719e746.png)

Instead of doing *S O*, *S 1*, *S 2* and so on. We could also do something like:

- `S (S O)` which returns 2
- `S (S (S O))` which returns 3
- `S (S (S (S O)))` which returns 4
- `S (S (S (S (S O))))` which returns 5

Now that we know what *S* is, and what it does, let’s go back to original code.

## Executing minustwo
---

How can we execute(call) ***minustwo***? We can use the ***Compute*** vernacular which Coq provides. We can do something like:

```coq
Compute minustwo 0. (* returns 0 *)
Compute minustwo 1. (* returns 0 *)
Compute minustwo 2. (* returns 0 *)
Compute minustwo 3. (* returns 1 *)
```

Let’s take a look at the first statement `Compute minustwo 0`. Essentially, we are passing `O` to ***minustwo*** definition. 

```coq
match n with
	| O => O
	| S O => O
	| S (S m) => m
end.
```

By looking at the body of the definition, we are handling the case for `O` , which is `O => O`. That is why when we pass `O` it returns `O`.

For, `Compute minustwo 1` , we are passing `1` to the ***minustwo*** definition, we are handling the case for *1*, which is *S O*. As we know, *S O = 1*. This case will match and it returns *O*.

For, `Compute minustwo 2`, we are passing *2* to the ***minustwo*** definition. For this case, we don’t have any direct case. What we have is *S (S m) ⇒ m.* So how does this work?

When we pass *2*, under the hood, Coq transforms *2* to *S (S O)*. So if we match this with the pattern in our case, it looks something like:

```
S (S O) = S (S m)
```

In our *match* case, we are returning *m* if the pattern matches to *S (S m)*. But what is *m*? If we look at the above analysis `S (S O) = S (S m)` , in the place of *m* we have *O* on LHS. So we’ll be returning *O* as the result which is exactly what we needed. Because **2 - 2 = 0**. Let’s do **5 - 2** now and see how it works.

## 5 - 2
---

`Compute minustwo 5.`  When we run this command we expect the output to be *3.* Let’s breakdown. As we know, Coq represents *5* under the hood as,

```
5 = S (S (S (S (S 0))))
```

So, let’s match this with our case:

```
S (S (S (S (S 0)))) = S (S m)
		            => S (S (S (S (S 0))))
		        m   => S (S (S 0))
```

*m* matches with `S (S (S 0))` which when reduced we get *3*. That is exactly what we want, because **5 - 2 = 3**. The reduction happens like:

```
m	=> S (S (S 0))
	=> S (S 1)
	=> S 2
	=> 3
```

## Conclusion
---
I was struggling to understand this for the past couple of hours before starting to write this post. I was finally able to grasp the concept after delving into the **`minustwo`** example and experimenting with different inputs. Through this journey, I gained a deeper appreciation for the elegance and power of pattern matching in Coq. It's amazing how a seemingly simple construct can unlock so much potential in our code.