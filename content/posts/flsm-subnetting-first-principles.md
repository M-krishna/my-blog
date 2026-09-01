+++
title = "Subnetting From First Principles: FLSM and the IP Odometer"
date = 2026-09-01
description = "Master Fixed Length Subnet Masking (FLSM) by understanding binary bit borrowing and the odometer analogy."

[taxonomies]
tags = ["networking", "first-principles", "infrastructure"]
+++

If you look at subnetting strictly through decimal numbers, it looks like dark magic. The rules seem arbitrary, the math feels disconnected, and memorizing cheat sheets becomes the only way to survive.

But if you look at it from first principles—the way the computer actually processes it—subnetting is just a binary tug-of-war. 

In this post, we'll break down **Fixed Length Subnet Masking (FLSM)**. This is the process of taking a large network and cutting it into perfectly equal-sized chunks. We'll explore how to borrow bits, how to find the "interesting octet," and why IP addresses behave exactly like a car's odometer.

---

## The Binary Tug-of-War

An IPv4 address is 32 bits long. When you are handed a root network like `192.168.100.0/24`, that `/24` prefix tells you exactly where the boundary lies. The first 24 bits are locked in as the network address. You cannot touch them. 

The remaining 8 bits are your "host portion". To create smaller subnets, you have to steal bits from the host portion and reassign them to the network. 

Because we are working in binary, every bit you borrow doubles the number of subnets you can create:
* Borrow 1 bit: 2 subnets
* Borrow 2 bits: 4 subnets
* Borrow 3 bits: 8 subnets

Every bit you borrow increases your total number of networks, but directly steals from the number of devices you can put inside them.

## The "Round Up" Rule

Let's say you need to split a network into exactly 5 subnets. Because you can only create subnets in chunks of powers of 2 (2, 4, 8, 16...), you can't create exactly 5. 

You use the formula `2^n` (where `n` is the number of borrowed bits) and find the smallest power of 2 that meets your requirement. 
* `2^2 = 4` (Too small)
* `2^3 = 8` (Perfect)

You borrow 3 bits. You'll physically create 8 subnets, use 5, and keep 3 in reserve for future growth.

## The Interesting Octet and the Odometer

Here is where most people get tripped up. Let's say you take a `172.16.0.0/16` network and borrow 4 bits, making it a `/20`. 

Where is that boundary? 
* Octet 1: Bits 1-8
* Octet 2: Bits 9-16
* **Octet 3: Bits 17-24**
* Octet 4: Bits 25-32

The subnet mask ends at bit 20, which is smack in the middle of the 3rd octet. This makes the 3rd octet the **Interesting Octet**. It is the *only* place the network address will increment.

Because you have 4 host bits remaining in that 3rd octet (Bits 21, 22, 23, and 24), your block size (or step size) is `2^4 = 16`. Your networks will count up by 16:
* `172.16.0.0`
* `172.16.16.0`
* `172.16.32.0`

But wait—if the network only steps by 16, how does a `/20` subnet hold 4,096 total hosts?

**Think of an IP address like a car's odometer.** 
The 4th octet is entirely made of host bits. It can hold values from `0` to `255`. When you count up IP addresses and hit `.255`, the odometer "rolls over," resetting the 4th octet to `0` and incrementing the 3rd octet by 1.

Every single time the 3rd octet goes up by 1, a full block of 256 addresses just rolled through the 4th octet. 

So, if our subnets step by 16 in the 3rd octet:
`16 steps × 256 addresses = 4,096 IPs`.

That step of 16 is literally just the number we have to write in the 3rd octet to represent jumping over 4,096 IP addresses. 

---

## Real-World FLSM in Action

Let's put this theory to the test with two practical examples.

### Example 1: The Device Split
Imagine you are starting with a standard home network of `192.168.1.0/24`. You want to isolate your main devices (Mac, Windows laptop, phones) from a local experimental lab environment. You decide you need exactly **4 equal subnets**.

1. **Find Borrowed Bits (n):** Using `2^n >= 4`, we see that `2^2 = 4`. We need to borrow 2 bits.
2. **New Subnet Mask:** We started with `/24` and borrowed 2 bits, making the new mask **`/26`**.
3. **Determine Block Size:** The boundary is in the 4th octet. We have 6 host bits left (`32 - 26 = 6`). Our block size is `2^6 = 64`.

The subnets will increment by 64 in the 4th octet:
* Subnet 1: `192.168.1.0/26` (Usable: `.1` to `.62`)
* Subnet 2: `192.168.1.64/26` (Usable: `.65` to `.126`)
* Subnet 3: `192.168.1.128/26` (Usable: `.129` to `.190`)
* Subnet 4: `192.168.1.192/26` (Usable: `.193` to `.254`)

### Example 2: The Virtual Server Pool
You are provisioning a large virtual network for a Proxmox hypervisor using a private block: `10.50.0.0/16`. You want to dedicate different subnets to different types of VMs (Web Servers, Databases, Caching, Monitoring, Staging). You need **6 equal-sized subnets**.

1. **Find Borrowed Bits (n):** You need 6 subnets. `2^2 = 4` (too small), but `2^3 = 8` works perfectly. You borrow 3 bits.
2. **New Subnet Mask:** Adding 3 bits to a `/16` gives a new mask of **`/19`**.
3. **Determine Block Size:** The `/19` puts our boundary in the 3rd octet (bits 17-24). We borrowed 3 bits, leaving 5 host bits in this specific octet. Our block size is `2^5 = 32`.

The subnets will increment by 32 in the **3rd octet**:
* Subnet 1: `10.50.0.0/19`
* Subnet 2: `10.50.32.0/19`
* Subnet 3: `10.50.64.0/19`
* Subnet 4: `10.50.96.0/19`
* Subnet 5: `10.50.128.0/19`
* Subnet 6: `10.50.160.0/19` *(Leaving two unused subnets starting at `.192` and `.224`)*

If we look closely at **Subnet 5**, we can see the odometer working:
* **Network Address:** `10.50.128.0`
* **First Usable Host:** `10.50.128.1`
* **Last Usable Host:** `10.50.159.254` (The IPs roll over continuously until hitting this wall)
* **Broadcast Address:** `10.50.159.255` 

---

## Conclusion

Fixed Length Subnet Masking (FLSM) is the foundation of network design. By mastering the binary tug-of-war, the round-up rule, and the odometer mental model, you stop guessing and start engineering. 

But FLSM has one major drawback: it forces every subnet to be the exact same size. If you need a subnet for 500 web servers and another subnet for a simple point-to-point router link that only requires 2 IP addresses, FLSM wastes massive amounts of address space. 

To solve this, we have to bend the rules. In the next post, we will tackle **Variable Length Subnet Masking (VLSM)**, where we learn how to carve out different-sized subnets from the exact same network block.