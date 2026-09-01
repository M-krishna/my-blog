+++
title = "How a Router Reads an IP Address: Subnet Masks, CIDR, and the Two Addresses You Can't Use"
date = 2026-08-28
description = "Working out from first principles what an IP address really is, why it needs a subnet mask to mean anything, and how the network and broadcast addresses fall out of that."
draft = true

[taxonomies]
tags = ["networking", "homelab", "ipv4"]
+++

I have been poking at my home router while trying to get a homelab off the ground, and I kept running into the same handful of words: IP address, subnet mask, CIDR, network address, broadcast address. I learned all of this years ago and promptly forgot it. So this post is me rebuilding it from the bottom, in the order the ideas actually depend on each other.

The whole thing exists to answer one question, which a router asks millions of times a second:

> A packet just arrived with a destination address. Do I hand it to something on a wire I'm directly attached to, or do I pass it along to another router?

Everything below is the machinery for answering that.

## An IP address is one number

An IPv4 address is a 32-bit unsigned integer. That's the entire definition. It is a number between 0 and 4,294,967,295.

We never write it as one number, because `3232235786` is unreadable. Instead we chop the 32 bits into four groups of 8 bits, write each group as a decimal number, and separate them with dots. A group of 8 bits is called an **octet**, and since 8 bits can hold values from 0 to 255, each octet is a number from 0 to 255.

So `192.168.1.10` is four octets. The dots are punctuation for humans. The machine sees:

```
192      168      1        10
11000000 10101000 00000001 00001010
```

If you glue those together you get one 32-bit number: `11000000101010000000000100001010`.

One thing worth being precise about: an IP address does not identify a machine. It identifies a **network interface**. A laptop with both Wi-Fi and Ethernet plugged in has two addresses. A router has one address on every interface it owns, and that is the reason routers exist at all: they sit with a foot in two or more networks.

### The binary you need

You cannot follow any of the rest of this without being able to move between decimal and binary for a single octet, so here is the whole trick. Each of the 8 bit positions has a fixed value:

```
128  64  32  16  8  4  2  1
```

To convert 168 to binary, walk left to right and greedily take each place value if it fits in what's left:

```
168 - 128 = 40   -> 1
 40 -  64         -> 0  (64 doesn't fit in 40)
 40 -  32 = 8    -> 1
  8 -  16         -> 0
  8 -   8 = 0    -> 1
  0                -> 0
  0                -> 0
  0                -> 0

168 = 10101000
```

To go the other way, add up the place values wherever there's a 1. `11000000` is 128 + 64 = 192.

Two values are worth memorising because they show up constantly: `255` is `11111111` (all bits on) and `0` is `00000000` (all bits off).

## Why a flat address doesn't work

Suppose addresses meant nothing except "this specific interface". Then every router on earth would need a table with four billion rows to know where to send anything. That is obviously impossible.

The fix is grouping. We declare that some of the leading bits of an address identify a **network**, and the remaining trailing bits identify a **host** within that network. A host is any device holding an address on that network.

Now a router only needs to know about networks, not individual hosts. "Anything starting with these first 24 bits goes out of port 3" collapses 256 addresses into one table entry. Real internet routers use this to collapse the whole address space into a few hundred thousand entries instead of four billion.

This split is the central idea of the entire post. Everything else is notation for expressing where the split falls.

## The subnet mask marks the split

The split point is expressed by a second 32-bit number called the **subnet mask**. It is written in the same dotted decimal notation, which makes it look like an address, but it is not one. It is a bit pattern.

The rule: wherever the mask has a `1`, the corresponding bit of the address belongs to the network part. Wherever the mask has a `0`, that bit belongs to the host part.

```
address  192.168.1.10     11000000 10101000 00000001 00001010
mask     255.255.255.0    11111111 11111111 11111111 00000000
                          \--------- network -------/ \-host-/
```

So in this network, the first 24 bits (`192.168.1`) name the network, and the last 8 bits are free to identify individual hosts.

The mask is required to be a run of 1s followed by a run of 0s, with no mixing. `255.255.255.0` is legal. `255.0.255.0` is not, and no operating system will accept it. This constraint is what makes the network part a **prefix**: a contiguous chunk of leading bits. Routers depend on that, because comparing prefixes is a cheap operation you can do in hardware, while comparing scattered bit positions is not.

A detail that took me a while to internalise: **the subnet mask is never transmitted in the packet.** An IPv4 header carries a source address and a destination address, and that is all. The mask is purely local configuration. Your machine uses its own mask to interpret its own situation, and each router uses its own table to interpret the destination. Two machines on the same wire configured with different masks will disagree about who their neighbours are, and things will break in confusing ways.

## CIDR notation is just a count

Since the mask is always a run of 1s followed by a run of 0s, writing out all 32 bits is wasteful. The only information in it is *how many* leading 1s there are. So we write that number after a slash:

```
192.168.1.10/24
```

The `/24` is the **prefix length**: 24 network bits, and therefore 32 - 24 = 8 host bits. This is **CIDR notation**, from Classless Inter-Domain Routing, the scheme introduced in 1993 that replaced an older system.

That older system is worth one paragraph, because it explains why `255.255.255.0` feels like a default. Before CIDR, addresses were **classful**: the leading bits of the address itself dictated the split. A first octet of 1 to 126 was Class A with a fixed /8, 128 to 191 was Class B with a fixed /16, and 192 to 223 was Class C with a fixed /24. You could not choose. An organisation needing 500 addresses had to take a Class B and waste 65,000 of them. CIDR threw the classes out and made the prefix length an independent, freely chosen number, which is exactly what "classless" means. `192.168.x.x` addresses are in the old Class C range, which is why /24 still feels natural there, but that is habit and not a rule.

Here is the correspondence between the two notations. It is worth being able to reconstruct this rather than memorise it, since every mask octet is just a run of leading 1s:

| CIDR | Mask | Host bits | Total addresses | Usable hosts |
|------|------|-----------|-----------------|--------------|
| /8   | 255.0.0.0       | 24 | 16,777,216 | 16,777,214 |
| /16  | 255.255.0.0     | 16 | 65,536     | 65,534 |
| /22  | 255.255.252.0   | 10 | 1,024      | 1,022 |
| /23  | 255.255.254.0   | 9  | 512        | 510 |
| /24  | 255.255.255.0   | 8  | 256        | 254 |
| /25  | 255.255.255.128 | 7  | 128        | 126 |
| /26  | 255.255.255.192 | 6  | 64         | 62 |
| /27  | 255.255.255.224 | 5  | 32         | 30 |
| /28  | 255.255.255.240 | 4  | 16         | 14 |
| /29  | 255.255.255.248 | 3  | 8          | 6 |
| /30  | 255.255.255.252 | 2  | 4          | 2 |
| /31  | 255.255.255.254 | 1  | 2          | 2 (special) |
| /32  | 255.255.255.255 | 0  | 1          | 1 (special) |

The total address count is `2^(32 - prefix)`. The usable count is that minus 2, and the next two sections explain which two go missing.

Notice that the prefix length going up by one halves the network. A /25 is half a /24. This is why subnetting always splits in powers of two.

## The network address: host bits all zero

Given an address and a mask, the **network address** is the address you get by setting every host bit to 0. It is the name of the network itself, the thing routers put in their tables. It is also the numerically lowest address in the range.

You compute it with a **bitwise AND** between the address and the mask. Bitwise AND compares the two numbers bit by bit and outputs 1 only when both inputs are 1:

```
0 AND 0 = 0
0 AND 1 = 0
1 AND 0 = 0
1 AND 1 = 1
```

This does exactly what we want. In the network part the mask is all 1s, so `bit AND 1` returns the bit unchanged. In the host part the mask is all 0s, so `bit AND 0` returns 0 and the host bits are wiped out.

For `192.168.1.10/24`:

```
address  11000000 10101000 00000001 00001010    192.168.1.10
mask     11111111 11111111 11111111 00000000    255.255.255.0
AND      -----------------------------------
network  11000000 10101000 00000001 00000000    192.168.1.0
```

With a /24 you can do this in your head, because the boundary lands neatly between octets and you just zero the last octet. The interesting cases are the ones where it doesn't.

## The broadcast address: host bits all one

The **broadcast address** is the mirror image: every host bit set to 1. It is the numerically highest address in the range, and it means "every host on this network". A packet sent to it is delivered to all of them.

You compute it with a **bitwise OR** against the inverted mask. Bitwise OR outputs 1 when either input is 1:

```
0 OR 0 = 0
0 OR 1 = 1
1 OR 0 = 1
1 OR 1 = 1
```

The inverted mask (flip every bit) is called the **wildcard mask**. For `255.255.255.0` the wildcard is `0.0.0.255`. OR-ing with it leaves the network part alone (`bit OR 0` returns the bit unchanged) and forces every host bit to 1.

```
address  11000000 10101000 00000001 00001010    192.168.1.10
wildcard 00000000 00000000 00000000 11111111    0.0.0.255
OR       -----------------------------------
broadcast 11000000 10101000 00000001 11111111   192.168.1.255
```

A simpler way to say the same thing: take the network address and set all host bits to 1. Or, easiest of all, the broadcast is one below the next network's address.

Broadcast is genuinely used, not a curiosity. DHCP works this way: a machine with no address yet shouts a request to the broadcast address because it has no idea who the server is. ARP, which resolves an IP address to a hardware address, broadcasts too. It is worth noting that this is a Layer 3 broadcast, and when it goes out on Ethernet it is carried in a frame addressed to the Layer 2 broadcast address `ff:ff:ff:ff:ff:ff`. The two are related but separate mechanisms.

Broadcast is also the reason a network is sometimes called a **broadcast domain**: it is exactly the set of machines that will hear each other's broadcasts. Routers do not forward broadcasts, which is what keeps one chatty network from flooding the world.

## Which addresses you can actually assign

So in every network, two addresses are spoken for: the lowest (host bits all 0, the network address) and the highest (host bits all 1, the broadcast address). Everything strictly between them is assignable to a real interface. That is where the `- 2` in the usable host count comes from.

For `192.168.1.0/24`:

```
192.168.1.0                  network address, not assignable
192.168.1.1 - 192.168.1.254  usable, 254 addresses
192.168.1.255                broadcast address, not assignable
```

There are two exceptions where the rule is deliberately suspended:

- A **/32** has zero host bits, so it is a single address. It is used for loopback interfaces and for host routes, meaning a route to one specific machine.
- A **/31** has one host bit, giving two addresses and, by the normal rule, zero usable ones. That would be useless, so RFC 3021 says that on a point-to-point link (two routers connected directly, nobody else on the wire) both addresses are usable. There is no need for broadcast when there is only one possible recipient. Before this, people burned a /30 on every link and wasted two addresses each time.

## Working an example where the boundary is not obvious

Take `192.168.1.130/26`.

/26 means 26 network bits and 6 host bits. The mask is 26 ones then 6 zeros:

```
11111111 11111111 11111111 11000000   =   255.255.255.192
```

The boundary falls inside the fourth octet, so we have to look at the bits.

```
130      = 10000010
mask bits = 11000000
```

Network address, AND them:

```
10000010
11000000
--------
10000000  = 128
```

So the network address is `192.168.1.128`.

Broadcast, set the 6 host bits to 1:

```
10000010  ->  10111111  = 128 + 32 + 16 + 8 + 4 + 2 + 1 = 191
```

Broadcast is `192.168.1.191`. Usable range is `192.168.1.129` through `192.168.1.190`, which is 62 addresses, matching `2^6 - 2`.

### Listing every subnet a prefix allows

The bit arithmetic above answers one question at a time: given this address, what network is it in? A more useful thing to see is the whole picture at once, meaning every subnet a /26 can possibly produce inside `192.168.1.0/24`. That turns out to be short, and once you can see it you rarely need the AND and the OR again.

Start by noticing that in almost every mask, three of the four octets are boring. They come out as 255 (all network bits) or 0 (all host bits). Exactly one octet has the boundary running through the middle of it, and it is the only one worth thinking about. Call it the **interesting octet**: the one whose mask value is neither 255 nor 0.

For a /26, write out 26 ones across the four octets:

```
octet 1   octet 2   octet 3   octet 4
11111111  11111111  11111111  11000000
   8    +    8    +    8    +    2      = 26 network bits
```

The first three octets are entirely network bits, so each is 255. The fourth has 2 network bits and 6 host bits, so its value is 128 + 64 = 192. That is where the 192 in `255.255.255.192` comes from. It was not chosen, it falls out of counting to 26. The fourth octet is the interesting one.

Now mark the fourth octet with `N` for a network bit and `H` for a host bit:

```
N N H H H H H H
```

A subnet is, by definition, all the addresses that agree on the network bits. So to enumerate every subnet, enumerate every value the two `N` bits can take. Two bits, four combinations, and that is the complete list. For each one the six `H` bits are free to run from `000000` to `111111`, so each subnet covers a run of consecutive addresses:

```
NN=00:   00000000  to  00111111   =    0  to   63
NN=01:   01000000  to  01111111   =   64  to  127
NN=10:   10000000  to  10111111   =  128  to  191
NN=11:   11000000  to  11111111   =  192  to  255
```

Reading the left column against the place values `128 64 32 16 8 4 2 1`: `00000000` is 0, `01000000` is 64, `10000000` is 128, and `11000000` is 128 + 64 = 192. So the four subnets begin at 0, 64, 128 and 192. Those numbers are not a rule to memorise, they are simply what two bits can spell.

Each run of addresses above is one subnet, and you have already met both of its ends under other names. The start has all host bits zero, so it is the network address. The end has all host bits one, so it is the broadcast address. Written out in full, the four subnets are:

| Subnet | Network address | Broadcast address | Usable range |
|---|---|---|---|
| `192.168.1.0/26`   | 192.168.1.0   | 192.168.1.63  | .1 to .62 |
| `192.168.1.64/26`  | 192.168.1.64  | 192.168.1.127 | .65 to .126 |
| `192.168.1.128/26` | 192.168.1.128 | 192.168.1.191 | .129 to .190 |
| `192.168.1.192/26` | 192.168.1.192 | 192.168.1.255 | .193 to .254 |

Those four side by side are exactly `192.168.1.0/24` cut into quarters, which is what subnetting means in practice.

This table earns its keep twice. Reading an address gets faster: hand me `192.168.1.130/26` and I look for the subnet containing 130, find the third row, and read off network `.128` and broadcast `.191` without touching binary. And planning a network becomes concrete, because the table is the complete menu. You cannot decide that your subnet will start at `.100`, since `01100100` does not have its host bits zeroed and so is not a network address. These four are the only choices a /26 offers here.

### The spacing shortcut

The subnets above are evenly spaced, 64 apart. That spacing is worth being able to get directly, because then you can jump to the right row without writing the table out.

The spacing is just how many addresses one subnet covers, which is `2^(host bits)`. Six host bits gives `2^6 = 64`. You can also read it straight off the mask: **256 minus the interesting octet's mask value**, here `256 - 192 = 64`.

Why that subtraction works is easiest to see from the last subnet. The mask has every network bit set to 1, which is the largest pattern those bits can hold, so the mask octet value always equals the start of the final subnet: 192 in this case. That final subnet runs to 255, the top of the octet. So its size is `256 - 192 = 64`, and since every subnet in the octet is the same size, that is the spacing for all of them.

Here is every possibility, which is a short enough list to just recognise:

| Network bits in the octet | Host bits | Spacing `2^(host bits)` | Mask octet |
|---|---|---|---|
| 0 | 8 | 256 | 0 |
| 1 | 7 | 128 | 128 |
| 2 | 6 | 64 | 192 |
| 3 | 5 | 32 | 224 |
| 4 | 4 | 16 | 240 |
| 5 | 3 | 8 | 248 |
| 6 | 2 | 4 | 252 |
| 7 | 1 | 2 | 254 |
| 8 | 0 | 1 | 255 |

The two right-hand columns add to 256 on every row, which is the shortcut stated as a table instead of as arithmetic.

One boundary case: /8, /16, /24 and /32 have no interesting octet, since every mask octet is 255 or 0. The split falls cleanly between octets and there is nothing to shortcut.

### One more, in the third octet

Take `172.16.20.200/22`.

22 network bits: 8 + 8 + 6, so the boundary is six bits into the third octet.

```
11111111 11111111 11111100 00000000   =   255.255.252.0
```

The interesting octet is the third one, and its mask value is 252, so the spacing is `256 - 252 = 4`. Subnets therefore begin at third octets 0, 4, 8, 12, 16, 20, 24, and so on. 20 is itself a multiple of 4, so it is the start of a subnet:

- Network address: `172.16.20.0`
- Broadcast address: `172.16.23.255` (the subnet spans third octets 20, 21, 22 and 23, and within the last of those the fourth octet runs up to 255)
- Usable range: `172.16.20.1` to `172.16.23.254`, which is `2^10 - 2 = 1022` addresses

Checking with the bits, since the third octet is where it gets interesting:

```
20       = 00010100
mask     = 11111100
AND      = 00010100 = 20      -> network third octet
OR wild  = 00010111 = 23      -> broadcast third octet
```

Note that `172.16.20.255` is an ordinary usable address here. It only looks like a broadcast if you assume /24 out of habit.

## What the router does with all this

Now the payoff. A router keeps a **routing table**: a list of destination prefixes, each with an instruction about where to send matching packets. Something like:

```
Destination        Next hop         Interface
192.168.1.0/24     directly         eth0
192.168.10.0/24    directly         eth1
10.0.0.0/8         192.168.1.254    eth0
0.0.0.0/0          203.0.113.1      wan0
```

When a packet arrives, the router takes the destination address and, for every row, ANDs it with that row's mask and checks whether the result equals that row's network address. Every row that matches is a candidate.

When several rows match, the router picks the one with the **longest prefix**. This is **longest prefix match**, and it is the whole routing algorithm in one sentence. A longer prefix describes a smaller, more specific set of addresses, so preferring it means preferring the most specific known route.

The last row, `0.0.0.0/0`, has a prefix length of zero. Zero network bits means the mask is all zeros, so ANDing anything with it gives zero, so it matches every possible address. It is the **default route**, and because its prefix is the shortest possible, longest prefix match guarantees it is only chosen when nothing else fits. That is exactly what "send it to my ISP if I don't recognise it" should mean.

Your own machine runs a smaller version of the same logic every time it sends a packet. Given a destination, it asks: does `destination AND my_mask` equal `my_network`?

- **Yes.** The destination is on my own wire. Use ARP to find its hardware address and send the frame straight to it. No router involved.
- **No.** It's somewhere else. Send the frame to the hardware address of my **default gateway** and let it worry about the rest.

The default gateway is a router interface sitting on your network, which is why the gateway address has to fall inside your own subnet. If it doesn't, your machine will conclude the gateway is remote, and to reach anything remote it needs the gateway, and the whole thing deadlocks. This is a classic misconfiguration and now you can see exactly why it fails.

This also explains a symptom you may have hit: two machines on the same physical switch that can't talk to each other. If one is `192.168.1.10/24` and the other is `192.168.1.130/26`, the first thinks they share a network and ARPs directly, while the second computes a different network address, concludes the first is remote, and sends to its gateway. The wire is fine. The arithmetic disagrees.

## Splitting a network, which is what "subnetting" means

Say your router hands out `192.168.1.0/24` and you want to separate your homelab boxes from everything else on the house Wi-Fi. You borrow bits from the host part and give them to the network part.

Take one bit and you get two /25s:

```
192.168.1.0/25     .0   to .127    (usable .1 to .126)
192.168.1.128/25   .128 to .255    (usable .129 to .254)
```

Take two bits and you get four /26s of 64 addresses each, starting at 0, 64, 128, 192. Take three and you get eight /27s of 32 each. Every bit you borrow doubles the number of networks and halves the size of each, and each split costs you two more addresses to network and broadcast overhead.

The practical point for a homelab is that separate subnets do not talk to each other without a router deciding to forward between them, which is the basic mechanism behind putting your IoT devices somewhere they can't reach your file server. In practice you would pair this with VLANs so the separation is enforced at Layer 2 as well, but the addressing half of the story is the arithmetic above.

## Address ranges that mean something special

Not every address is available for general use. The ones worth knowing:

- **10.0.0.0/8, 172.16.0.0/12, and 192.168.0.0/16** are private ranges, defined in RFC 1918. They are not routed on the public internet, and everyone reuses them behind NAT. That is why your home network and mine can both be 192.168.1.0/24. Note that the middle one is a /12, so it covers 172.16.0.0 through 172.31.255.255, not just 172.16.x.x.
- **127.0.0.0/8** is loopback. The whole /8 is reserved even though almost everyone only ever uses 127.0.0.1.
- **169.254.0.0/16** is link-local. A machine assigns itself one of these when DHCP fails. Seeing a 169.254 address is a reliable sign that something didn't get an address properly.
- **100.64.0.0/10** is carrier-grade NAT space, used by ISPs between their customers and the public internet. If your router's WAN address is in this range, you are behind CGNAT and cannot accept inbound connections without a tunnel. This matters the moment you try to reach a homelab service from outside.
- **224.0.0.0/4** is multicast, meaning delivery to a group of interested hosts rather than to everybody.
- **0.0.0.0** means "unspecified". As a /0 prefix it is the default route. As a listening address it means "all local interfaces", which is why binding a service to 0.0.0.0 exposes it on every interface and binding to 127.0.0.1 keeps it local.
- **255.255.255.255** is the limited broadcast address, meaning "everyone on this wire" without needing to know the local network address. Routers never forward it. DHCP discovery uses it.

## Checking your own machine

None of this sticks until you compute it by hand and then confirm it. On Linux:

```bash
ip -4 addr show
```

Look for a line like `inet 192.168.1.42/24 brd 192.168.1.255 scope global eth0`. There is your address, your prefix, and the broadcast address the kernel derived from them. Work out the network address yourself before reading it off anything.

```bash
ip route
```

The first line is usually `default via 192.168.1.1 dev eth0`, which is the `0.0.0.0/0` entry, and below it a line like `192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.42`, which is the directly connected network. That is the two-branch decision from earlier, written out.

To check your arithmetic, `ipcalc` gives you everything at once:

```bash
ipcalc 192.168.1.130/26
```

Or without installing anything, using Python's standard library:

```python
import ipaddress

interface = ipaddress.ip_interface("192.168.1.130/26")
network = interface.network

print(network.network_address)      # 192.168.1.128
print(network.broadcast_address)    # 192.168.1.191
print(network.netmask)              # 255.255.255.192
print(network.num_addresses)        # 64
print(list(network.hosts())[0])     # 192.168.1.129
print(list(network.hosts())[-1])    # 192.168.1.190
```

And to see the AND directly, so it stops being abstract:

```python
address_bits = int(ipaddress.ip_address("192.168.1.130"))
mask_bits = int(ipaddress.ip_address("255.255.255.192"))

network_bits = address_bits & mask_bits
broadcast_bits = network_bits | (~mask_bits & 0xFFFFFFFF)

print(ipaddress.ip_address(network_bits))     # 192.168.1.128
print(ipaddress.ip_address(broadcast_bits))   # 192.168.1.191
```

That last one is the entire post in four lines. An address is a number, a mask is a number, AND gives you the bottom of the range, OR with the inverted mask gives you the top.

## A note on IPv6

IPv6 addresses are 128 bits instead of 32, written as eight groups of four hex digits. The prefix idea carries over unchanged: `2001:db8::/32` means the same thing structurally as a /32 does in IPv4, and longest prefix match still decides routing. Two differences worth flagging. Subnets are conventionally /64 regardless of how many machines are on them, because the lower 64 bits are used for address autoconfiguration. And there is no broadcast address at all: IPv6 dropped it in favour of multicast groups, so "all nodes on this link" is a specific multicast address rather than the top of the range. That means the "minus 2" rule does not apply.

## Summary

- An IPv4 address is a 32-bit number, written as four octets for readability.
- The subnet mask is a second 32-bit number, a run of 1s then 0s, splitting the address into a network prefix and a host part.
- CIDR notation writes the mask as the count of leading 1s.
- The network address is the address with all host bits 0, computed by ANDing address with mask. It is the lowest address in the range and the name of the network.
- The broadcast address is the address with all host bits 1, the highest in the range, meaning "everyone here".
- Everything strictly between them is assignable, giving `2^(32 - prefix) - 2` usable addresses, with /31 and /32 as deliberate exceptions.
- Routers store prefixes and choose the longest one that matches, with `0.0.0.0/0` as the catch-all default.
- Your own machine does a one-line version of this to decide between ARPing directly and handing the packet to its gateway.

If you can compute the network and broadcast addresses for something like `10.14.203.77/19` on paper and then check it with `ipcalc`, you have the whole thing.