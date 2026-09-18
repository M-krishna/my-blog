+++
title = "How does a computer know if an IP Address is on the same network?"
date = 2026-09-18
template = "post.html"
draft = false
+++

Before a computer can send an IP packet directly to another computer on an Ethernet network, it needs to know whether the destination IP address belongs to the same network as itself.

Why?

Because if the destination is on the same network, the computer needs the destination computer's MAC address. It can discover that MAC address using ARP (Address Resolution Protocol). If the destination is not on the same network, it should not send an ARP request for the destination IP.

So the question is:
> **How does a computer calculate whether a destination IP belongs to the same network?**

The answer is: **it uses the subnet mask and a bitwise AND operation.**

# An Example
Suppose our computer has:
* IP address: `192.168.1.10`
* Subnet mask: `255.255.255.0`

The subnet mask can also be written as: `192.168.1.10/24`. The `/24` tells us that the first 24 bits are the network portion of the IP address.

In binary,
```
IP address:

11000000.10101000.00000001.00001010
```
```
Subnet mask:

11111111.11111111.11111111.00000000
```

We can visually separate the network and host portions:
```
11000000.10101000.00000001 | 00001010
<--------- network --------> <---host-->
```

# Finding our network address
The computer performs a bitwise AND between its IP address and the subnet mask. The AND operation works like this:
```
1 AND 1 = 1
1 AND 0 = 0
0 AND 1 = 0
0 AND 0 = 0
```
Applying it our IP:
```
11000000.10101000.00000001.00001010
11111111.11111111.11111111.00000000
-----------------------------------
11000000.10101000.00000001.00000000
```
The result is: `192.168.1.0`.

Therefore: `192.168.1.10` `AND` `255.255.255.0 = 192.168.1.0`. This is our network address.

# Now check the destination
Suppose we want to send a packet to: `192.168.1.20`. The computer performs the same calculation on the destination IP:
```
11000000.10101000.00000001.00010100
11111111.11111111.11111111.00000000
-----------------------------------
11000000.10101000.00000001.00000000
```
The result is: `192.168.1.0`. Now we compare the two network addresses:
* Our network: `192.168.1.0`
* Destination network: `192.168.1.0`

They are the same. Therefore:
> **The destination IP is on the same network as us.**

The computer can now look for the destination's MAC address, and if it doesn't already know it, it can send an ARP request. Conceptually,
```
Destination IP
      │
      ▼
(Destination IP) AND (subnet mask)
      │
      ▼
Destination network
      │
      │ compare with
      ▼
Our network
      │
      ├── Same ──→ ARP request for destination MAC
      │
      └── Different ──→ Destination is not local (try the router)
```

# What if the destination is different?
Now suppose the destination is: `192.168.2.20`. Calculate its network address:
```
11000000.10101000.00000010.00010100
11111111.11111111.11111111.00000000
-----------------------------------
11000000.10101000.00000010.00000000
```
The result is: `192.168.2.0`. Now we compare the network addresses:
* Our network: `192.168.1.0`
* Destination network: `192.168.2.0`

They are different: `192.168.1.0 != 192.168.2.0`. Therefore, the destination is not on our local network.

The important point is that the computer does **not** need to send an ARP request asking: **who has `192.168.2.20`?** because it has already determined that `192.168.2.20` isn't a local destination.

# The Calculation
So the entire calculation can be reduced to three steps:

## 1. Calculate our network
(our IP) **Bitwise AND** (our subnet mask)

## 2. Calculate the destination's network
(destination IP) **Bitwise AND** (our subnet mask)

## 3. Compare the results
```
if:
    our network == destination network
then:
    destination is on the same network
otherwise:
    destination is on a different network
```

**For example:**
* Our IP:       `192.168.1.10`
* Subnet mask:  `255.255.255.0`
* Destination:  `192.168.1.20`

The calculation is:
```
192.168.1.10 AND 255.255.255.0 -> 192.168.1.0

192.168.1.20 AND 255.255.255.0 -> 192.168.1.0
```
Same result: `192.168.1.0 == 192.168.1.0`. Therefore, the destination is local.

That's the fundamental calculation behind determining whether an IP address belongs to the same subnet.

# What's next?
So far, we have looked at the case where the destination is either on our local network or determined to be somewhere else.

But what happens when the destination **isn't** on our network? That' where things get more interesting.

In the next post, we'll follow what happens after the computer determines that the destination is remote. We'll look at **routers, routing tables, how the computer searches the routing table for a matching route, the default route/default gateway, and how the packet eventually gets sent toward its destination**.

For now, the important piece to remember is:
```
Same network?
     │
     ├── Yes → ARP for the destination
     │
     └── No  → We need to figure out where to send the packet next
```