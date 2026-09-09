+++
title = "VLSM: Stacking Subnets Like Tetris"
date = 2026-09-09
description = "How to stack different-sized subnets, cross octet boundaries, and avoid the notorious 'Minus Two' trap."
[taxonomies]
tags = ["networking", "first-principles", "infrastructure"]
+++

In our previous post, we looked at Fixed Length Subnet Masking (FLSM) and how to carve a network into perfectly equal chunks using the binary tug-of-war. 

FLSM is a great foundational concept, but in the real world, it has one glaring flaw: **it is terribly inefficient.** 

Imagine you are handed a `192.168.10.0/24` network. You need a large subnet of 60 IPs for your virtual machines, and a tiny subnet of just 2 IPs to link two physical switches together. If you use FLSM, you are forced to make every subnet the exact same size. To satisfy the VM requirement, you have to split the network into chunks of 64. 

You hand 64 IPs to your virtual machines (perfect), but you also have to hand 64 IPs to your switch link, immediately wasting 62 perfectly good addresses. 

To solve this, we have to bend the rules. We need **Variable Length Subnet Masking (VLSM)**.

## Stacking Blocks Like Tetris

If FLSM is like slicing a pizza into perfectly equal slices, VLSM is like playing Tetris. It allows you to change the block size on the fly, stacking differently sized networks back-to-back so that absolutely no space is wasted.

To play this game, you only need to follow one strict rule: **The Golden Rule of VLSM.** 

You *must* sort your network requirements from largest to smallest before you calculate anything. Because subnets rely on binary math (powers of 2), larger subnets require specific boundaries on the IP odometer. If you place a tiny subnet at the beginning of your network, your larger subnets will no longer align with the binary grid, and the math breaks down.

Always place your biggest blocks first.

## The "Minus Two" Trap

Before we build a network, we need to address a trap that catches almost every engineer when they first learn VLSM.

Recall our formula for finding usable hosts: `2^h - 2` (where *h* is the number of host bits). We subtract 2 because the very first IP is reserved for the Network Address, and the very last is reserved for the Broadcast Address.

Let's say your web server subnet needs 100 IPs. You calculate `2^7 = 128`. Subtract 2, and you get 126 usable hosts. 

Here is the trap: **When you are calculating where the *next* subnet begins, do not add 126.**

Those two unusable addresses didn't vanish into thin air; they still take up physical space on the network. The **block size**—the actual chunk of real estate the subnet consumes—is the full 128. If your web servers start at `.0`, the next subnet must start exactly 128 spaces later at `.128`. 

You only use the "minus two" rule to check if your servers will fit. When stacking blocks, you only ever step forward by the pure powers of 2 (16, 32, 64, 128).

## Real-World Example 1: The Home Lab (4th Octet)

Let's put this into practice. We are carving up `192.168.10.0/24` for a home lab. We have three requirements, and we've already sorted them from largest to smallest:

1. **Proxmox Virtual Machines:** Needs 60 IPs
2. **Raspberry Pi Cluster:** Needs 20 IPs
3. **Management Network:** Needs 12 IPs

Let's stack the blocks:

**1. Proxmox VMs (60 IPs)**
* We need a block size that fits 60. `2^6 = 64`. 
* Because it is our first subnet, it starts at the very beginning: `192.168.10.0`.
* It consumes 64 spaces, ending at `.63`.

**2. Raspberry Pi Cluster (20 IPs)**
* We need a block size that fits 20. `2^5 = 32`. 
* We start exactly where the last one left off: `192.168.10.64`.
* It consumes 32 spaces. `64 + 32 = 96`. It ends at `.95`.

**3. Management Network (12 IPs)**
* We need a block size that fits 12. `2^4 = 16`.
* We start exactly where the Pi cluster ended: `192.168.10.96`.
* It consumes 16 spaces. `96 + 16 = 112`. It ends at `.111`.

Look at the efficiency! We satisfied all our requirements, and our last IP was `.111`. Everything from `192.168.10.112` all the way to `.255` is completely untouched and saved for future infrastructure.

## Real-World Example 2: The Cloud VPC (3rd Octet)

Now let's tackle a larger scenario where the boundaries cross over into the 3rd octet. You are designing a Virtual Private Cloud for a custom Django application. Your cloud provider gives you a massive starting block: `10.50.0.0/16`.

Sorted requirements:
1. **Database Pool:** Needs 1,000 IPs
2. **Web Servers:** Needs 500 IPs
3. **Load Balancers:** Needs 100 IPs

**1. Database Pool (1,000 IPs)**
* `2^10 = 1024`. Our block size is 1,024 IPs. 
* To get 10 host bits, our mask becomes `/22` (32 - 10 = 22).
* A `/22` means our boundary falls in the **3rd octet** (bits 17-24). 
* We have 2 host bits left in the 3rd octet (`2^2 = 4`). So, our subnets step by **4** in the 3rd octet.
* **Network:** `10.50.0.0/22`. It steps up by 4, so it ends right before `10.50.4.0`. (The broadcast is `10.50.3.255`).

**2. Web Servers (500 IPs)**
* `2^9 = 512`. Our block size is 512 IPs.
* Our mask becomes `/23` (32 - 9 = 23).
* We start exactly where the DB pool ended: `10.50.4.0/23`.
* A `/23` leaves 1 host bit in the 3rd octet (`2^1 = 2`). The step is **2** in the 3rd octet.
* **Network:** `10.50.4.0/23`. It steps by 2, ending right before `10.50.6.0`. (The broadcast is `10.50.5.255`).

**3. Load Balancers (100 IPs)**
* `2^7 = 128`. Our block size is 128 IPs.
* Our mask becomes `/25` (32 - 7 = 25).
* We start exactly where the Web Servers ended: `10.50.6.0/25`.
* A `/25` boundary is in the **4th octet**. The block size is 128.
* **Network:** `10.50.6.0/25`. It ends at `10.50.6.127`.

Even though we are working with thousands of IPs and moving across octets, the Tetris game is exactly the same!

## Three Real-World Gotchas

The math on paper is perfect, but building actual infrastructure requires a bit of practical foresight. Keep these three rules in mind when taking VLSM into production:

### 1. The "Tight Packing" Trap
In our exercises, we stacked the subnets perfectly back-to-back to be as mathematically efficient as possible. In a production environment, tight packing can backfire. If your Web Server traffic scales up and you suddenly need more IPs, you cannot simply expand the subnet mask because the Database network is already occupying the very next IP address! Network engineers intentionally leave empty IP blocks (gaps) between critical subnets so they can grow later without having to re-IP the entire environment.

### 2. The `/31` Point-to-Point Exception
Standard subnetting rules dictate that a `/30` is the smallest useful subnet. It has a block size of 4, leaving exactly 2 usable IPs (perfect for a direct cable link between two switches). However, modern networking equipment supports a special rule (RFC 3021) that allows a **`/31`** subnet. A `/31` has a block size of 2, and it breaks the normal rules by eliminating the Network Address and Broadcast Address entirely. Both IPs are 100% usable, making it the most efficient way to connect two routers.

### 3. Isolation by Default
Subnetting is fundamentally about isolation. Once you carve a root network into a `/26` for virtual machines and a `/27` for a container cluster, those two groups of devices can no longer talk to each other by default. Even if they are plugged into the exact same physical switch, the subnet mask acts as a logical wall.

To break through that wall, you must configure a gateway (a router or a firewall) to explicitly pass traffic between the different blocks. And that is exactly what we will tackle in our next post.