+++
title = "Understanding IP Addresses and CIDR"
date = 2026-08-04
draft = true

[taxonomies]
tags = ["networking", "internet"]
+++

When I first started learning about networking, I kept seeing things like:
```
192.168.1.10
10.0.0.5
172.16.0.0/16
192.168.1.0/24
```
I understood that the first part was an IP address, but the `/24` or `/16` was less obvious. What does it mean? Why do we need it? And what exactly does an IP address represent?

This post is my attempt to build an understanding of IP addresses and CIDR from the ground up.

<hr>

# What is an IP address?
Let's start with a simple problem.

Suppose there are two computers connected to the same network:
```
Computer A
Computer B
```
If Computer A wants to send some data to Computer B, the network needs some way to identify the destination. This is where an IP address comes in. An **IP address identifies a network interface on an IP network.** For example:
```
192.168.1.10
```
An IPv4 address is made up of **32 bits**.

We usually don't write all 32 bits directly. Instead, we divide them into four groups of 8 bits and write each group as a decimal number. `192.168.1.10` can be represented in binary as:
```
11000000.10101000.00000001.00001010
```
Each group is called an **octet** because it contains 8 bits. So:
```
192      . 168      . 1        . 10 
11000000 . 10101000 . 00000001 . 00001010 
8 bits      8 bits       8 bits     8 bits
```
That gives us:
```
8 + 8 + 8 + 8 = 32 bits
```
So an IPv4 address is simply a 32-bit number that we normally write in a more convenient form.

<hr>

# But an IP address isn't just an identifier
This is where things become interesting. Consider this address:
```
192.168.1.10
```
We might be tempted to think of it as one single number that identifies a computer. But an IP address actually contains **two pieces of information**:
* Network identifer
* Host identifier

The **network portion** tells us which network the address belongs to. The **host portion** identifies a particular host within that network. For example, imagine:
```
Network: 192.168.1.0

Computer A: 192.168.1.10
Computer B: 192.168.1.20
Computer C: 192.168.1.30
```
All three computers belong to the same network: `192.168.1.0` but each has different host address.

This raises an important question:
> **How does a computer know which part of the IP address represents the network and which part represents the host?**

That's what the **subnet mask** tells us.

# The Subnet Mask