+++
title = "Lazy evaluation"
date = 2024-03-15
template = "post.html"
+++

Trying to figure out what ***lazy evaluation*** is after knowing Haskell is fully lazy. Let’s write a normal sum function which takes in two arguments and add them and return the value. There would be two variants of the function:

1. Eager evaluation
2. Lazy evaluation

## Eager evaluation

```tsx
function sum(a: number, b: number): number {
	return a + b
}

console.log(sum(10 + 20, 100))
```

In the above code snippet, we are calling the `sum` function by passing two arguments. First is **10 + 20** and the second is **100**. 

What happens in this situation is, Typescript will first evaluate **10 + 20** and gets the result which would be **30**. Then it would call the `sum` function by passing in **30 and 100**

## Lazy evaluation

```tsx
function sum(a: () => number, b: () => number): () => number {
	return () => a() + b()
}

console.log(sum(() => (10 + 20), () => 100)())
```

In the above code snippet, what happens is, we’ve defined the arguments and return types as functions, which would return a number.

So, when the calling the `sum` function, we wouldn’t pass the numbers directly, instead we pass in functions which returns a number. Now the return type of the `sum` function is a *function that returns a number,* which would be the final result.

## Creating a Lazy type

Let’s create a lazy type and update the above lazy sum function

```tsx
type Lazy<T> = () => T

function sum(a: Lazy<number>, b: Lazy<number>): Lazy<number> {
	return () => a() + b();
}
```

## Don’t need, don’t evaluate

How can we avoid big computations that are not needed? We don’t evaluate them. Let’s take a look at couple of examples that demonstrate this.

```tsx
function recurse<T>(): T {
	return recurse();
}

function sum(a: number, b: number): number {
	return a;
}

console.log(sum(10 + 20, recurse()))
```

In the above code snippet, we have two functions. One is a recursive function named `recurse` and another one is `sum`. 

The `recurse` function just calls itself and it doesn’t contain any base case so it won’t exit. The `sum` function, doesn’t actually adds the two arguments, instead it returns the first argument as it is. 

Finally we call the `sum` function by passing two things. One is **10 + 20 = 30** and the other is we are passing(calling) the `recurse` function. What happens when we run the code?

> We’ll get: `[ERR]: Maximum call stack size exceeded`
> 

Even before actually executing the function, Javascript will first evaluate 10 + 20 which gives 30. And after that, it calls the recurse function. Since this function doesn’t have a base case, it won’t exit. Which leads to stack size exceed. Even though we didn’t do anything with the “return value” of the `recurse` function inside the `sum` function, our program crashes. How can we resolve this?

```tsx
function sum(a: Lazy<number>, b: Lazy<number>): Lazy<number> {
	return a;
}

console.log(sum(() => 10 + 20, () => recurse()));
```

What happens if we run the above program? We’ll get:

> `[LOG]: () => 10 + 20`
> 

This is because our `sum` function returns the first argument as it is, which is nothing but a function. As you see from the above result, it doesn’t evaluate the expression because we haven’t really called the function. Let’s now call the `sum` function.

```tsx
console.log(sum(() => 10 + 20, () => recurse())());
```

It’ll return `30` as the return value.

As you can see, we are only returning `a`(the first argument) and not the “value” of the `recurse` function. But since we made it evaluate lazily, it won’t crash our program by going into *infinite recursion* before actually executing our function. This is the advantage of lazy evaluation.

## Generators

Generators are part of Javascript that does lazy evaluation. Let’s modify our sum function and make it use generator.

```tsx
function* sum(a: number, b: number): Generator<number> {
	yield a + b
}

let sumGenerator: Generator<number> = sum(10 + 20, 100);

console.log(sumGenerator.next().value)
```

In the above code snippet, we have a *generator function* named **sum**. The generator function returns a *generator*. We call the **sum** function by passing in **10 + 20 = 30** and **100**. The execution won’t happen, instead it returns a *generator*. The generator will have various different states that I’m not gonna go into.

The generator will give have a **next** function, which upon called will execute the expression(30 + 100), which gives the result 130.

## Real world

Let’s try to create a function that generates an infinite list of numbers. Let’s look at two different ways:

1. Naive approach
2. Generator approach (Lazy evaluation)

## Naive approach

```tsx
function infiniteList(): number[] {
	let n: Array<number> = []
	for (let i = 0; ; i++) {
		n.push(i);
	}
	return n;
}

infiniteList();
```

If we try to execute the above code snippet, the function will keep on executing and doesn’t return anything. We haven’t mentioned the upper limit in the for loop because we wanted to generate an infinite list. Since the function won’t return anything, its not useful. How would we solve this?

## Generator approach

```tsx
function* infiniteList(): Generator<number> {
	for (let i = 0;; i++) yield i
}

let infiniteListGenerator: Generator<number> = infiniteList();

for (let i of infiniteListGenerator) {
	console.log(i)
}
```

> **DISCLAIMER**: Running the above code eats up your memory fast
> 

In the above code snippet, we’ve created a *generator function* named **infiniteList,** which iterates from 0 to Infinity. Then we are calling the generator function which returns a generator(**infiniteListGenerator**). Finally we are iterating over the generator which gives us values from 0 to Infinity.

The difference between the naive approach and the generator approach is that, the naive approach won’t even return anything until it completes the calculation. On the other hand, the generator approach evaluates it one by one, once it evaluates it returns the result of the evaluation back to the caller.

Using this feature, now we can easily able to generate infinite lists. Let’s say we wanted first 10, 100, 1000 and 10,000 values. We could do something like:

```tsx
Array.from(new Array(10), () => infiniteListGenerator.next().value);
Array.from(new Array(100), () => infiniteListGenerator.next().value);
Array.from(new Array(1000), () => infiniteListGenerator.next().value);
Array.from(new Array(10000), () => infiniteListGenerator.next().value);
Array.from(new Array(100000), () => infiniteListGenerator.next().value);
```

Of course it would take performance hits once the value gets bigger. But the point is, its possible to generate infinite list with the help of lazy evaluation. Next time when you want to avoid big computations when generating something, try lazy evaluation.