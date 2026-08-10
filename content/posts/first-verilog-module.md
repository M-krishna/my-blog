+++
title = "Writing and Simulating Your First Verilog Module"
date = 2026-08-03
draft = true

[taxonomies]
tags = ["Verilog", "Digital Design", "Simulation"]
+++

In the [previous post], we learned one of the most confusing topics in Verilog -- `wire` vs `reg`. But we haven't actually built or simulated anything yet. That's about to change.

In this post, we'll build the simplest logic gate possible: a **NAND gate**. More importantly, we'll learn how to **test** our design using a **testbench**. 

By the end of this article, you'll know how to:
* Write your first Verilog module.
* Write a testbench.
* Understand what an `initial` block is.
* Learn how `initial` differs from `always`.
* Simulate your design and verify that it works.

Let's get started.

# Building a NAND Gate
If you've taken the nand2tetris course, you already know how important the NAND gate is. One of the beautiful things about digital logic is that every other logic gate can be built using only NAND gates. We'll eventually get there. For now, let's build a single NAND gate in Verilog.
```verilog
module nand_gate(
    input wire a,
    input wire b,
    output wire y
);
    assign y = ~(a & b);
endmodule
```
If you've read the previous article, nothing here should be surprising. We have two inputs (`a` and `b`) and one output (`y`). That's our entire hardware description. Simple. But how do we know it actually works? That's where the simulation comes in.

# What is a Testbench?
Imagine you've written a function in Python. How do you verify that it works? You call it with different inputs and check the outputs. For example,
```python
print(add(2, 3))
print(add(10, 5))
```
A testbench serves exactly the same purpose in Verilog. It provides inputs to your design and observes the outputs. One important thing to remember is this:
> A testbench is **not hardware**.

It's simply a Verilog program that exists to test your hardware. It will never be synthesized onto an FPGA or ASIC. It's only job is to answer one question:
> "Does my design behave the way I expect?"

# Writing our first testbench
Let's create a new file called 