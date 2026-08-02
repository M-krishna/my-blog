+++
title = "Wire vs Reg in Verilog"
date = 2026-08-02

[taxonomies]
tags = ["Verilog", "Digital Design", "Hardware", "Nand2Tetris"]
+++
If you've just started learning Verilog, you've probably come across two keywords that seem deceptively simple:
* `wire`
* `reg`

At first glance, they sound straightforward. A **wire** must represent a wire, and a **reg** must represent a hardware register, right?

Unfortunately, that's where the confusion begins. One of the biggest misconceptions beginners have is believing that a `reg` always becomes a hardware register (or a flip-flop). It doesn't.

In fact, you can write perfectly valid Verilog where a `reg` becomes nothing more than a few logic gates.

So why is it called `reg` then?

The answer lies in Verilog's history, and by the end of this article, you'll understand not only **when** to use `wire` and `reg`, but also **why** Verilog makes this distinction in the first place.

Let's start with `wire`.

# What is a `wire`?
A `wire` is exactly what its name suggests. It represents a physical wire in a digital circuit. A wire has one simple job:

> "Carry a value from one place to another."

Notice what I didn't say. I didn't say it **stores** a value.

A wire never remembers anything. It simply reflects whatever is currently driving it.

Imagine a battery connected to a light bulb.
```
+---------+
| Battery |
+---------+
      |
      |
      |  (wire)
      |
+---------+
|  Bulb   |
+---------+
```
The wire doesn't store electricity. It simply carries it from the battery to the bulb. If the battery turns off, the wire immediately reflects that change.

The same idea applies to Verilog. You can declare a wire like this:
```verilog
wire a;
```
This creates a single-bit wire. Most circuits, however, work with multiple bits at once.
```verilog
wire [7:0] data;
wire [15:0] instruction;
wire [31:0] address;
```
The notation `[7:0]` means there are eight bits.
```
Bit position
7 6 5 4 3 2 1 0
```
Bit 7 is the **most significant bit (MSB)** and bit 0 is the **least significant bit (LSB)**. If you've completed nand2tetris, you'll probably recognize declarations like:
```verilog
wire [15:0] out;
```
Since almost every component in nand2tetris course operates on 16-bit values:

## A wire must be driven
Here's an important rule:
> "A wire cannot create its own value."

Something else must drive it. Usually that's one of two things:
* a continuous assignment (`assign`)
* the output of another module

For example,
```verilog
wire y;

assign y = a & b;
```
Here, the expression
```verilog
a & b
```
is driving `'y'`.

You can also connect a wire directly to another module.
```verilog
wire sum;

HalfAdder ha (
    .a(a),
    .b(b),
    .sum(sum),
    .carry(carry)
);
```
Here, the `HalfAdder` module drives the value of `sum`.

The important thing to remember is that **a wire never decides its own value**. It always gets its value from somewhere else.

## What happens if nothing drives a `wire`?
Suppose we write this:
```verilog
wire x;
```
That's it. No `assign`. No module output. Nothing. What value does `x` have? The answer is:
```
Z
```
`Z` stands for **High Impedance**.

You can think of it as a disconnected wire. Imagine cutting a wire between two components,
```
Battery      Bulb

 +---+        +---+
 |   |        |   |
 +---+        +---+

     wire is disconnected
```
Since nothing is driving the wire, it doesn't know whether it should be 0 or 1. Simulation tools represent this state using `Z`. For beginners, it's usually enough to think of `Z` as **"nothing is driving the signal."**

## Continuous assignments
Now let's look at one of the most common uses of a wire.
```verilog
wire y;

assign y = a & b;
```
Many programmers coming from C, C++, Java or Python read this as:
> "Compute `a & b` once and store the result in `y`."

That isn't what's happening.

The keyword `assign` describes **continuous hardware**. A better way to read it as:
> "Keep `y` equal to `a & b` forever."

Suppose the inputs change like this:
| a | b | y |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

You never write code to tell Verilog to recompute `y`. Instead, the hardware itself continuously evaluates the expression. As soon as either input changes, the output changes automatically. That's exactly how an AND gate behaves in real hardware.

When you write:
```verilog
assign y = a & b;
```
You're not describing sequence of instructions. You're describing an AND gate.

This is one of the biggest mindset shifts when learning a hardware description language. In software, you write instructions that execute one after another. In Verilog, you're describing hardware that exists all at once. The AND gate isn't waiting for your program to call it. It simply exists, continuously producing an output based on its inputs. Once this idea clicks, `wire` becomes much easier to understand. A wire doesn't store values. It simply connects pieces of hardware together.

# What is a `reg`?
Now comes the keyword that has confused Verilog beginners.
```verilog
reg y;
```
If you guessed `reg` stands for **register**, you're not alone. That' exactly what most beginners assume. After all, that's what the name suggests. Unfortunately, the name is a little misleading.

A `reg` is **not** simply a hardware register. Instead, you can think of it as **a variable that can be assigned inside an `always` block**.

That's the key distinction.

Whenever you assign to a signal inside an `always` block, Verilog requires that signal to be declared as a `reg`.
```verilog
reg y;

always @(*) begin
    y = a & b;
end
```
Notice that there is no `assign` statement. Instead, we're assigning `y` inside an `always` block. Since `y` is assigned procedurally, it must be declared as a `reg`. Just like `wire`, a `reg` can also represent multiple bits.
```verilog
reg a;
reg [7:0] data;
reg [15:0] instruction;
```

## Why isn't `y` declared as a `wire`?
This is one of the most common mistakes beginners make. Consider the following code.
```verilog
wire y;

always @(*) begin
    y = a & b;
end
```
This looks perfectly reasonable. After all, `y` isn't storing anything. Shouldn't it be a `wire`?

The answer is **no**.

A `wire` can only be driven by a continuous assignment (`assign`) or by another module's output. It **cannot** be assigned inside an `always` block. That's why Verilog reports an error. The correct version is:
```verilog
reg y;

always @(*) begin
    y = a & b;
end
```
Likewise, the opposite is also true.
```verilog
reg y;

assign y = a & b; // Error
```
A continuous assignment can only drive a `wire`. The correct declaration is:
```verilog
wire y;

assign y = a & b;
```
At this point, you might be wondering something. If `reg` stands for register, where is the register in this code?
```verilog
reg y;

always @(*) begin
    y = a & b;
end
```
There isn't one.

Even though `y` is declared as `reg`, the synthesized hardware is simply an AND gate. No flip flops. No storage. Just combinational logic.

This surprises many beginners because the name `reg` suggests that a hardware register must exist. In reality, **the declaration alone doesn't determine the hardware**. The hardware depends on **how the signal is assigned**. Let's look at two examples.

## Example 1: Combinational Logic
```verilog
reg y;

always @(*) begin
    y = a & b;
end
```
The `@(*)` means "whenever any input used inside the block changes." Every time `a` or `b` changes, the expression is evaluated again. Nothing is remembered between evaluations. The synthesized hardware is simply an AND gate, even though `y` is declared as a `reg`, there is no register anywhere in the circuit.

## Example 2: Sequential Logic
Now consider this example:
```verilog
reg q;

always @(posedge clk) begin
    q <= d;
end
```
This time, the `always` block executes only on the rising edge of the clock. Unlike the previous example, `q` must remember its previous value until the next clock edge. That requires storage. The synthesized hardware is a D flip-flop.

This is where the name `reg` finally matches the hardware. The important thing to understand is that **both examples use the same `reg` keyword.** The difference isn't the declaration. The difference is the kind of `always` block you're using. One describes combinational logic. The other describes sequential logic with storage. That's why it's inaccurate to say that "`reg` is a register".

A better way to think about it is:
> A `reg` is a signal that is assigned inside an `always` block.

Whether it eventually becomes an AND gate, a Multiplexer, a latch, or a flip-flop depends entirely on the hardware described by that `always` block.

# `wire` vs `reg`
By now, you've probably noticed something interesting. Neither `wire` not `reg` tells us what hardware will be created. Instead, they tell Verilog **how a signal is driven**.

A `wire` is driven continuosuly, either by an `assign` statement or by another module's output. A `reg`, on the other hand, is driven procedurally inside an `always` block. That's really the distinction.

> **NOTE:** A `reg` does **not** automatically store a value. Whether storage is inferred depends on the `always` block.

## Simple questions to ask whenever you declare a signal
**Who is driving this signal?**

If the answer is:
* an `assign` statement -> use a `wire`.
* another module's output -> use a `wire`.
* an `always` block -> use a `reg`.

That's all there is to it. After writing a few modules, this quickly becomes second nature.

## Common Beginner Mistakes
Let's look at a few mistakes that almost everyone makes when learning Verilog.

### Mistake #1: Assigning to a `wire` inside an `always` block
```verilog
module AndGate(
    input wire a,
    input wire b,
    output wire y
);
    always @(*) begin
        y = a & b;
    end
endmodule
```
This won't compile because `y` is a `wire`, and `wire`s cannot be assigned inside an `always` block. The fix is simply:
```verilog
module AndGate(
    input wire a,
    input wire b,
    output reg y
);
    always @(*) begin
        y = a & b;
    end
endmodule
``` 
Notice that changing `wire` to `reg` **doesn't** suddenly create a flip-flop. This module still synthesizes to nothing more than an AND gate.

### Mistake #2: Using `assign` on a `reg`
The opposite mistake is just as common.
```verilog
reg y;

assign y = a & b;
```
This is illegal because continuous assignments can only drive nets (`wire`). The correct version is:
```verilog
wire y;

assign y = a & b;
```

### Mistake #3: Thinking `reg` means "Hardware register"
Many beginners see this code:
```verilog
reg y;

always @(*) begin
    y = a & b;
end
```
and immediately conclude that a hardware register must exist because `y` is declared as a `reg`. It doesn't. The synthesized circuit simply an AND gate. No Storage. No clock. No flip-flops. The keyword `reg` only tells Verilog that `y` is assigned procedurally. It says nothing about the hardware that will eventually be synthesized.

## Why is it called `reg` then?
This is a great example of history leaving its fingerprints on a programming language.

When Verilog was originally designed, there were two different kinds of objects.
* **Nets**, represented by `wire`, modeled physical connections between pieces of hardware.
* **Variables**, represented by `reg`, stored values assigned by procedural blocks.

At the time, Verilog was primarily a **hardware simulation language**, not a synthesis language. As synthesis tools became more sophisticated, they learned to convert `always` block into real hardware.

Sometimes that hardware was combinational logic. Sometimes it was latch. Sometimes it was a flip-flop.

The `reg` keyword never changed, even though it could now describe many different kinds of hardware. That's why its name feels misleading today. Modern Verilog users have simply learned that `reg` doesn't necessarily mean "register".

## A note about SystemVerilog
If you've read newer Verilog code, you may have noticed something interesting. Instead of writing:
```verilog
reg y;
```
you'll often see:
```systemverilog
logic y;
```

SystemVerilog introduced the `logic` type to remove much of the confusion surrounding `wire` and `reg`. A `logic` can be used almost everywhere a `reg` was used, making code easier to read and reason about. That said, understanding `wire` and `reg` is still important.

# Closing Thoughts
At first, `wire` and `reg` seem like two mysterious keywords that you simply have to memorize. Fortunately, they are much simpler than they appear.

A `wire` represents a connection that is driven continuously.

A `reg` represents a signal that is assigned procedurally inside an `always` block.

Neither keyword tells you what hardware will be synthesized. That depends entirely on **how you describe the hardware**, not on the declaration itself. Whenever you find yourself wondering which one to use, don't ask:

> **"Does this signal store a value?"**

Instead ask:

> **"Who is driving this signal?"**

If it's an `assign` statement or another module's output, use a `wire`.

If it's assigned inside an `always` block, use a `reg`.

That's the mental model I wish someone had taught me when I first started learning Verilog.