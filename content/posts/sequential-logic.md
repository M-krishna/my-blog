+++
title = "Sequential Logic"
date = 2024-03-07
template = "post.html"
+++

To know more about **Sequential logic**, we need to first know about **Combinational logic**.

## Combinational logic

In digital circuits combinational logic refers to circuits whose output solely depends on the current values of the input. Whenever the input the changes, the output changes respectively. Combinational logic doesn’t involve time, it is purely depend on the current state of the values.

Some examples of logic circuits that comes under Combination logic are **NAND, AND, OR, NOT, XOR** gates. These are elementary gates that are used to perform boolean algebra. Let’s look at a specific gate: **AND**. The truth table of the AND gate looks like this:

```plaintext
| A | B | OUT |
|---|---|-----|
| 0 | 0 |  0  |
| 0 | 1 |  0  |
| 1 | 0 |  0  |
| 1 | 1 |  1  |
```

To reiterate, whenever the value of the input signals changes, the output changes.

## Sequential logic

Sequential logic on the other hand deals with ***time***. Whenever we take ***time*** into consideration, we have to deal with memory and state. In Sequential logic, the output not only depends on the current input but also on the previous inputs. Sequential logic has ***state***, which is nothing but ***memory***. But the combinational logic doesn’t have state since it doesn’t deal with ***time***. Some examples of logic circuits that comes under Sequential logic are **Flip-Flop, Bit, Registers, RAM**. 

There are two types of Sequential logic circuits:

1. **Synchronous**
2. **Asynchronous**

But before looking into these two types of sequential logic circuits, we need to know about ***clock signals***. What are clock signals?

## Clock signals

In electronics, a clock signal is an electronic logic signal(voltage or current) that oscillates between a high and a low state at a constant frequency. 

In a Synchronous logic circuit, the clock signal is applied to all storage devices, like flip-flops and causes them to change state simultaneously.

A clock signal is produced by electronic oscillator called a clock generator. The most common form of clock signal is the square wave form.

Circuits using clock signal for synchronization may become active at either raising edge, falling edge or in the case of double data rate, both in the raising edge and falling edge of the clock cycle.

## Synchronous Sequential logic

In Synchronous Sequential logic circuits, the state of the device(circuit) changes only at discrete time intervals in response to a ***clock signal***. For example, a Data Flip Flop(DFF) can be implemented using Synchronous sequential logic:

```verilog
module DFF(
	input D, clk,
	output reg Q
);
	always @(posedge clk) Q <= D;
endmodule
```

In the above code, we have implemented DFF using Verilog. DFF takes a data input(**D**) and a clock signal(**clk**). Whenever the clock signal goes from ***negedge(0)*** to ***posedge(1),*** the output would be the input.

## Asynchronous Sequential logic

In Asynchronous Sequential logic, the state of the device(circuit) can change at any time in response to changing inputs. For example, the same DFF can be implemented using Asynchronous sequential logic:

```verilog
module DFF(
	input D, clk, reset,
	output Q
);
	always @(posedge clk or posedge reset) begin
		if (reset) begin
			Q <= 0;
		end else begin
			Q <= D;
		end
	end
endmodule
```

In the above code example, we’ve added an input named **reset**, which is part of the `always` block sensitive list. Along with the clock signal(`clk`), whenever the `reset` is `1` our circuit resets the value of the output to `0`. In this case, the DFF resets the value of the output to `0`.

This is what we mean by *the state of the device(circuit) can change at any time in response to changing inputs*.