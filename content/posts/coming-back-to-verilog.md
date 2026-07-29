+++
title = "Coming back to nand2tetris, in Verilog"
date = 2026-07-29
description = "A two year gap, the toolchain I use, and the Verilog equivalent of hello world."

[taxonomies]
tags = ["verilog", "nand2tetris"]
+++

I started nand2tetris in January 2024, stopped in April, and picked it up again in
May 2026. This post is the restart: where the project got to, what I build it with,
and the smallest Verilog program that actually runs.

## The gap

The history is blunt about it:

```
2024-01-13  NAND GATE
2024-04-15  updated Makefile for sequencial logic
2026-05-03  started a refresher series on Verilog
```

Between January and April 2024 I got a long way. NAND, then Not, And, Or and Xor
built out of nothing but NAND. Then Mux and DMux and their 4-way and 8-way
variants, the 16-bit gates, HalfAdder, FullAdder, Add16, Inc16, the ALU, and then
the first sequential chips: a D flip flop, a 1-bit register, a 16-bit register.

Then I stopped for two years.

Coming back, the nand2tetris half was still there. I could still explain how a Mux
is wired, why a DMux is its mirror image, what the ALU control bits do. The Verilog
half was completely gone. I opened my own testbenches and could not confidently say
which signals needed to be `reg` and which needed to be `wire`, or why some
assignments used `=` and others `<=`.

So the first thing I did on returning was not the next chip. It was a refresher on
the language, worked through as three sets of notes before touching the project
again. Relearning a language while debugging a circuit means two unknowns at once,
and I would rather have one.

Which is also why these posts exist. Writing a thing down is how I get it to stay
put, and the two year gap is proof I need that.

One decision worth recording, since it is the reason any of this is harder than the
course intends: nand2tetris ships its own HDL and its own simulator, and I am not
using either. I am building the same chips in Verilog instead. The course HDL stops
existing the moment the course ends. Rebuilding a design I already understand is
the cheapest way to learn a real language, because the problem is never the thing
fighting me, only the syntax.

## The tools

Two commands do everything.

```bash
brew install icarus-verilog
```

`iverilog` compiles Verilog into a simulation binary. `vvp` runs that binary. It is
`gcc` then `./a.out`:

```bash
iverilog -o nand_gate nand_gate.v nand_gate_tb.v
vvp nand_gate
```

Worth being clear about what Icarus is, because it shapes everything that follows:
it is a simulator, not a synthesizer. It never produces anything you could load
onto real hardware. It reads a description of a circuit and plays it forward in a
pretend clock domain that only exists inside `vvp`. That is why a testbench can
have an `initial` block and a starting instant, which no physical chip has.

The third tool is the one that no longer works. GTKWave opens `.vcd` waveform
files, and as of May 2026 the Homebrew cask does not install on current macOS:

```bash
brew install --cask gtkwave   # broken on recent macOS
```

This turned out not to be a blocker. A testbench can print its signals to the
terminal instead, and at the scale of single gates a printed trace is easier to
read than a waveform anyway. It also copies into a blog post, which a screenshot
of a waveform viewer does not. The testbenches still write their `.vcd` files. I
just have nothing to open them with yet, and so far have not needed to.

## Hello world

The Verilog equivalent of hello world is not one file, it is two: a module, and a
testbench that drives it. Neither does anything on its own. That pairing is the
first real difference from writing software.

The module is a NAND gate, which is the right choice for reasons beyond tradition,
since in nand2tetris every other chip is eventually built from this one:

```verilog
module nand_gate(
    input wire a,
    input wire b,
    output wire y
);
    assign y = ~(a & b);
endmodule
```

Three ports and one line of logic. `assign` means continuous assignment: `y` is not
computed once, it is permanently wired to `~(a & b)` and re-evaluates whenever `a`
or `b` changes. That is a wire, not a statement.

Nothing in that file runs. To see it work, something has to feed it inputs and
report what comes out. That is the testbench:

```verilog
module nand_gate_tb;
    reg a; reg b;
    wire y;

    nand_gate uut(
        .a(a),
        .b(b),
        .y(y)
    );

    initial begin
       $dumpfile("nand_gate.vcd");
       $dumpvars(0, nand_gate_tb);

       a = 0; b = 0; #10;
       a = 0; b = 1; #10;
       a = 1; b = 0; #10;
       a = 1; b = 1; #10;

       $finish;
    end

    initial begin
        $monitor("[Time=%t] a=%b b=%b | y=%b", $time, a, b, y);
    end
endmodule
```

The testbench has no ports, because nothing drives it. It is the top of the
hierarchy. It declares `a` and `b` as `reg` because it assigns to them by hand, and
`y` as `wire` because the gate drives that. It instantiates the gate as `uut`, unit
under test, and connects each port by name.

Then `#10` waits ten time units. Nothing sleeps. Simulated time is a number that
`vvp` advances, so all four cases finish instantly. `$monitor` prints a line
whenever a watched signal changes, and `$finish` ends the simulation.

Compile and run:

```bash
iverilog -o nand_gate nand_gate.v nand_gate_tb.v
vvp nand_gate
```

```
VCD info: dumpfile nand_gate.vcd opened for output.
[Time=                   0] a=0 b=0 | y=1
[Time=                  10] a=0 b=1 | y=1
[Time=                  20] a=1 b=0 | y=1
[Time=                  30] a=1 b=1 | y=0
nand_gate_tb.v:20: $finish called at 40 (1s)
```

That is a NAND truth table: high everywhere except both inputs high. Two files, two
commands, and a circuit that demonstrably works.

Everything after this is the same loop at greater scale.
