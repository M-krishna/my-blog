+++
title = "Internet From First Principles: Layer 1, The Physical Layer"
date = 2026-08-10
description = "What problem does the physical layer actually solve? Turning bits into something that can travel, and back again."

[taxonomies]
tags = ["networking", "osi-model", "first-principles"]

[extra]
toc = true
+++
I've been a software engineer for almost ten years, and I recently realized something uncomfortable: if you woke me up at 3am and asked me to explain how the internet actually works, layer by layer, I'd stumble. I know how to use it. I don't know how it works.

So I'm starting a series where I rebuild my understanding of networking from the ground up, one OSI layer at a time, asking a single question at each layer: **what problem does this layer exist to solve?** No hand-waving, no "it just works". If I can't explain it simply, I don't understand it yet.

First stop: Layer 1, the Physical Layer.

# The problem it solves
Computers think in 1s and 0s. Wires, radio waves, and fiber optic cables don't know what a "1" or a "0" is, they only know how to carry a signal. Something has to sit at the boundary and translate between "here is a bit" and "here is a voltage change, a pulse of light or a radio wave", in both directions.

That's the entire job of the physical layer. Nothing more. It doesn't know what a MAC address is. It doesn't know what an IP address is. It doesn't know if the bits it's carrying belong to a cat video or a bank transfer. It just moves raw signal back and forth and converts it to and from bits at each end.

# An analogy
Imagine you're standing in a pitch-dark cave with a friend on the other side, and all you have is a flashlight. You agree on a rule: flashlight ON means "1", flashlight OFF means "0". You start flashing a sequence, and your friend writes down the 1s and 0s as they see them.

Congratulations, you've just implemented a physical layer. You picked a medium (light), a signaling method (on/off), and a speed (how fast you flash). You haven't said anything meaningful yet, no words, no addresses, no message content. You've just proven that you can reliably move a sequence of two distinguishable states from one point to another. Everything else in networking gets built on top of that one gurantee.

# The technical bit
Physical layer implementations vary by medium, but they're all solving the same problem:
* **Copper(Ethernet cables)**: bits are represented as voltage changes on the wire.
* **Fiber optic**: bits are represented as pulses of light.
* **WiFi**: bits are represented as modulated radio waves.

None of these mediums are "the real one", they're just different ways of encoding the same abstraction. The physical layer is the only layer that has to deal with actual physics. Every layer above it works with clean, already-decoded bits and never has to think about any of that again.

One detail that tripped me up at first: this translation is bidirectional. It's not just "signal comes in, gets turned into bits." Your machine is also constantly converting outgoing bits into a physical signal to send. Every layer above physical hands it bits to transmit and receives bits back from it, the physical layer is doing the convertion work in both directions, simultaneously, without ever looking at what those bits mean.

There's also a layer of engineering here that's easy to skip past: encoding schemes like Manchester encoding, which insert signal transitions in the middle of each bit so that the receiving end can stay synchronized with the sender's clock, even without a shared clock signal. Cable standards (Cat5e, Cat6, Cat7) and connectors (RJ45, SFP+) exist because "just send a voltage down a wire" runs into very real limits on distance, speed, and interference, and someone has to standardize the physical shape of the solution.

# What it explicitly does NOT do
This is the part I keep coming back to, because it's the seam where a lot of confusion starts. The physical layer has:
* No addressing (no concepts of "who" a signal is for)
* No error correction (it can sometimes detect that a signal was garbled, but it doesn't try to fix it, it just hands up what it received, wrong or not)
* No understanding of frames, packets, or messages

It is, deliberately, dumb. That's a feature, not a limitation. Keeping this layer dumb is exactly what lets copper, fiber and radio all serve as valid substrates for every protocol built on top of them, none of these protocols have to know or care which physical medium is underneath.

# Try it yourself
A few commands I used to make it concrete on my own machine (Mac):
```bash
# List all network interfaces
ifconfig

# See hardware port details (which interface is WiFi, which is Ethernet)
networksetup -listallhardwareports

# Capture 10 raw frames and see the Ethernet header
sudo tcpdump -i en0 -c 10 -e
```
Running the last command is the moment this stopped being theoretical for me. Watching real frames scroll past, knowing that a moment ago they were literally voltage on a wire, is a small thing, but it made the abstraction feel real instead of assumed.

# Handoff to Layer 2
Once the physical layer has done it's one job (turning signal into bits, and bits into signal), it has no idea what to do with those bits next. It just hands them upward. The next layer, Data Link, is where those raw bits start getting organized into something meaningful: frames, addressed to specific devices on the local network.

That's where I'm headed next.

*This is part 1 of a series where I'm rebuilding my networking fundamentals from scratch. Part 2, the Data Link Layer, is next.*