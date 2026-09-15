+++
title = "Building a Network from scratch with iproute2"
date = 2026-09-12
description = "Building a tiny network from scratch using Linux network namespaces, veth pairs, IP addresses, routing, ARP, and ping."

[taxonomies]
tags = ["linux", "networking", "iproute2"]
+++

I've been learning computer networking from the ground up, starting with IP addresses and CIDR, and recently started exploring how Linux actually implements networking.

While learning `iproute2`, I came across Linux network namespaces and virtual Ethernet pairs (`veth`). They provide a surprisingly simple way to build an entire network on a single machine. 

So instead of just reading about networking concepts, I wanted to build a tiny network myself.

The experiment is simple:
* Create two isolated network namespaces.
* Create a virtual Ethernet cable between them.
* Attach one end of the cable to each namespace.
* Assign an IP address to each interface.
* Put both addresses in the same network.
* Ping one namespace from the other.

We'll then inspect what Linux actually did to make that communication possible. The final setup will look roughly like this:
```
         Linux machine
    ┌───────────────────────────────────────┐
    │                                       │
    │   namespace 1         namespace 2     │
    │                                       │
    │   10.0.0.1/24         10.0.0.2/24     │
    │       │                    │          │
    │     veth1                veth2        │
    │       │                    │          │
    │       └──── virtual cable ─┘          │
    │                                       │
    └───────────────────────────────────────┘
```
The goal isn't merely to make `ping` work. The goal is to understand why it works.

I'll be using Docker for this experiment. Spin up a docker container with Ubuntu to try carry out the experiment:
```bash
docker run -it --rm --privileged --name netlab ubuntu:24.04 bash
```
Inside the docker container we have to install `iproute2`:
```bash
apt update && apt install iproute2
```
Verify the installation using:
```bash
ip -V
```

# 1. The Building Blocks
Before we start running commands, let's understand the pieces we're going to use.

## Network namespaces
A Linux network namespace gives us an isolated networking environment. A network namespace has its own:
* Network interfaces
* IP addresses
* Routing table
* Neighbor table
* Network stack

This means we can create two namespaces and make them behave somewhat like two separate computers, even though both are running on the same physical machine. We'll call them:
* `ns1`
* `ns2`

Initially, these two namespaces are completely isolated from each other.

## Virtual Ethernet pair
We also need a way to connect the two namespaces. Linux provides something called a **veth pair**. A veth pair consists of two virtual Ethernet interfaces that are directly connected to each other. You can think of it as a virtual Ethernet cable:
```
veth1 ◄────────────────► veth2
```
Anything sent into one end comes out of the other end. This is exactly what we need.

# 2. Create Two Network Namespaces
Let's start by creating our two namespaces:
```bash
ip netns add ns1
ip netns add ns2
```
We can verify that they were created:
```bash
ip netns list
```
We should see something like:
```
ns1
ns2
```
At this point, we have two separate networking environments. But they don't have a network connection between them. In fact, if we inspect the interfaces inside one of them:
```bash
ip netns exec ns1 ip link
```
We'll see **loopback** (`lo`) interface along with other interfaces which we can't use for our experiment. The same is true for `ns2`. So our starting point is essentially:
```
┌──────────────┐       ┌──────────────┐
│     ns1      │       │     ns2      │
│              │       │              │
│      lo      │       │      lo      │
│              │       │              │
└──────────────┘       └──────────────┘

       No connection between them
```
Let's fix that.

# 3. Create a Virtual Ethernet Cable
We can create a `veth` (virtual ethernet) pair using:
```bash
ip link add veth1 type veth peer name veth2
```
This creates two interfaces:
```
veth1 ◄────────────────► veth2
```
These two interfaces are connected to each other. We can think of them as the two ends of a physical Ethernet cable. At this point, however, both interfaces are in our original/default network namespace. We haven't put them inside `ns1` and `ns2` yet.

# 4. Attach the Cable to the Namespaces
Now we'll move one end of the cable into each namespace. Move `veth1` into `ns1`:
```bash
ip link set veth1 netns ns1
```
And move `veth2` into `ns2`:
```bash
ip link set veth2 netns ns2
```
Our topology is now:
```
┌──────────────────┐                  ┌──────────────────┐
│       ns1        │                  │       ns2        │
│                  │                  │                  │
│     veth1        │                  │        veth2     │
│       │          │                  │          │       │
└───────┼──────────┘                  └──────────┼───────┘
        │                                        │
        └──────────── virtual cable ─────────────┘
```
We can verify this:
```bash
ip netns exec ns1 ip link
```
and:
```bash
ip netns exec ns2 ip link
```
Each namespace should now contain one end of the veth pair.

# 5. Bring the Interfaces Up
There's one more thing we need to do before using the interfaces. Creating an interface doesn't necessarily mean that it is operational. We need to bring each interface up:
```bash
ip netns exec ns1 ip link set veth1 up

ip netns exec ns2 ip link set veth2 up
```
To check their state, you can use:
```bash
ip netns exec ns1 ip link show veth1

ip netns exec ns2 ip link show veth2
```
The interfaces should now show `UP`. Our virtual Ethernet cable is now live:
```
ns1                                      ns2

veth1                                    veth2
  │                                        │
  └──────────── veth pair ─────────────────┘
```
But we haven't assigned IP addresses yet.

# 6. Assign IP Addresses
Let's create a small IP network. We'll use: `10.0.0.0/24`, and assign:
* `ns1` -> `10.0.0.1/24`
* `ns2` -> `10.0.0.2/24`

Inside `ns1`:
```bash
ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth1
```
Inside `ns2`:
```bash
ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth2
```
Notice something important here. We're assigning the IP addresses to the **network interfaces**:
* `veth1` -> 10.0.0.1
* `veth2` -> 10.0.0.2

The namespace itself doesn't have an IP address. The namespace provides the isolated networking environment, while the interfaces is what participates in the network. We can verify the addresses:
```bash
ip netns exec ns1 ip addr

ip netns exec ns2 ip addr
```

# 7. Why did we put them in the same network?
We deliberately choose:
```
10.0.0.1/24
10.0.0.2/24
```
Both addresses belong to: `10.0.0.0/24`. The `/24` tells us that the first 24 bits represent the network portion of the address. Therefore:
* `10.0.0.1/24` -> network: `10.0.0.0/24`
* `10.0.0.2/24` -> network: `10.0.0.0/24`

So `ns1` and `ns2` are directly connected to the same IP network. We can see the resulting route:
```bash
ip netns exec ns1 ip route
```
We should see something similar to:
```
10.0.0.0/24 dev veth1 proto kernel scope link src 10.0.0.1
```
This is an important line. It tells the kernel:

> To reach any address in `10.0.0.0/24`, use `veth1`.

So when `ns1` wants to communicate with `10.0.0.2`, it doesn't need a router. `10.0.0.2` is directly reachable through `veth1`.

# 8. Let's Ping!
Everything should now be in place. Let's try to communicate from `ns1` to `ns2`:
```bash
ip netns exec ns1 ping 10.0.0.2
```
And we should see something like:
```
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.0xx ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.0xx ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.0xx ms
```
It works!

We've just built a tiny network from scratch. But now comes the interesting question: **What exactly happened when we ran `ping`?**

# 9. If there's only one other computer, Why do we need a MAC Address?
We haven't spoke about MAC addresses so far. In this section, we'll get to know why MAC address is important when it comes to Ethernet/Local network communication.

This was something I initially found confusing. There are only two namespaces:
```
ns1 ◄──────────────► ns2
```
There is only one possible destination. So couldn't `ns1` just send the packet down the cable?

The answer is: **the Ethernet layer still uses MAC addresses.**

Even though there is only one other device on our virtual Ethernet link, Ethernet doesn't change its rules based on how many devices happen to be connected. Let's look at our setup again:
```
ns1                                      ns2

10.0.0.1                                10.0.0.2
   │                                        │
veth1                                     veth2
MAC A                                      MAC B
   │                                        │
   └──────────── virtual cable ─────────────┘
```
At the IP layer, `ns1` wants to send a packet to: `10.0.0.2`. But Ethernet doesn't deliver frames using IP addresses. Ethernet uses MAC addresses. So Linux needs to know:
```
10.0.0.2 -> which MAC address?
```
This is where **ARP** comes in.

# 10. ARP: Finding the MAC address
ARP stands for **Address Resolution Protocol**.

It's job is to discover the MAC address associated with an IPv4 address on the local network. When `ns1` wants to communicate with `10.0.0.2`, it can essentially ask:

> Who has `10.0.0.2`?

`ns1` responds:

> `10.0.0.2` is me. My MAC address is `MAC B`.

Linux can then remember this mapping. We can inspect the neighbor table:
```bash
ip netns exec ns1 ip neigh
```
After the ping, we should see something similar to:
```
10.0.0.2 dev veth1 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```
Conceptually, Linux now has:
```
IP address          MAC address

10.0.0.2       →    aa:bb:cc:dd:ee:ff
```
So even though there's only one device on the cable, the Ethernet frame still has a destination MAC address.

# 11. IP Address vs MAC Address
This experiment makes the distinction between IP and MAC addresses much easier to see.

* The IP address identifies the destination at the **Network layer**: `10.0.0.2`.
* The MAC address identifies the destination on the **local Ethernet link (Data-link layer)**: `aa:bb:cc:dd:ee:ff`

They serve different purposes. You can think of the process as:
```
             IP layer

        "I want to reach 10.0.0.2"
                    │
                    ▼
          Routing decision
                    │
                    ▼
        "10.0.0.2 is directly
             connected"
                    │
                    ▼
             ARP lookup
                    │
                    ▼
          "Its MAC is MAC B"
                    │
                    ▼
              Ethernet
```
The resulting frame conceptually looks like:
```
┌─────────────────────────────────────────┐
│ Ethernet Frame                          │
│                                         │
│ Destination MAC: MAC B                  │
│ Source MAC:      MAC A                  │
│                                         │
│   ┌─────────────────────────────────┐   │
│   │ IP Packet                       │   │
│   │                                 │   │
│   │ Source IP:      10.0.0.1        │   │
│   │ Destination IP: 10.0.0.2        │   │
│   │                                 │   │
│   └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```
So the fact that there are only two computers doesn't eliminate the need for MAC addresses. The Ethernet layer still operates using Ethernet's rules.

# 12. The complete journey of our Ping
Now we can put everything together. When we run:
```bash
ip netns exec ns1 ping 10.0.0.2
```
`ping` generated an ICMP packet. The packet has:
* Source IP: `10.0.0.1`
* Destination IP: `10.0.0.2`

The networking stack in `ns1` examines the destination. Because `10.0.0.2` belongs to the directly connected network: `10.0.0.0/24`, the routing table tells it to use: `veth1`.

Before sending the Ethernet frame, Linux needs the destination MAC address. If it doesn't already knows it, ARP resolves:
```
10.0.0.2 -> MAC B
```
Linux then constructs an Ethernet frame:
```
Destination MAC : Mac B
Source MAC      : Mac A
```
Inside that frame is our IP packet:
```
Source IP       : 10.0.0.1
Destination IP  : 10.0.0.2
```
The frame goes through:
```
          ns1
           │
           │
         veth1
           │
           │
   virtual Ethernet cable
           │
           │
         veth2
           │
           │
          ns2
```
`ns2` receives the frame, extracts the IP packet, sees that the destination IP is its own: `10.0.0.2`, and the ICMP layer responds. The response travels back through the same process. And that's why `ping` succeeds.

# 13. Inspecting everything we built
One of the best parts of this experiment is that Linux lets us inspect almost every piece of the network.

## Network interfaces
```bash
ip netns exec ns1 ip link
ip netns exec ns2 ip link
```

## IP addresses
```bash
ip netns exec ns1 ip addr
ip netns exec ns2 ip addr
```

## Routing tables
```bash
ip netns exec ns1 ip route
ip netns exec ns2 ip route
```

## Neighbor/ARP table
```bash
ip netns exec ns1 ip neigh
ip netns exec ns2 ip neigh
```

These aren't just commands to memorize. Each one exposes a different part of the networking stack:
* `ip link` -> interfaces
* `ip addr` -> IP addresses
* `ip route` -> routing decisions
* `ip neigh` -> IP <-> MAC mappings

This is one of the reasons `iproute2` is such a useful tool for learning networking. Instead of treating the network as a black box, we can inspect the state maintained by the kernel.

# 14. The Entire Experiment
Once we understand each individual step, the complete setup is surprisingly small.

* Create the namespaces:
```bash
ip netns add ns1
ip netns add ns2
```

* Create the virtual Ethernet cable:
```bash
ip link add veth1 type veth peer name veth2
```

* Move each end into its namespace:
```bash
ip link set veth1 netns ns1
ip link set veth2 netns ns2
```

* Bring the interfaces UP:
```bash
ip netns exec ns1 ip link set veth1 up
ip netns exec ns2 ip link set veth2 up
```

* Assign IP addresses:
```bash
ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth1
ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth2
```

And finally:
```bash
ip netns exec ns1 ping 10.0.0.2
```

That's it.

We've created two isolated networking environments, connected them with a virtual Ethernet cable, configured their interfaces, assigned IP addresses, and communicated between them.

# 15. What did we actually learn?
We started with:
```
┌──────────────┐       ┌──────────────┐
│     ns1      │       │     ns2      │
│              │       │              │
│  completely  │       │  completely  │
│   isolated   │       │   isolated   │
└──────────────┘       └──────────────┘
```
And built:
```
┌──────────────────┐                  ┌──────────────────┐
│       ns1        │                  │       ns2        │
│                  │                  │                  │
│   10.0.0.1/24    │                  │   10.0.0.2/24    │
│       │          │                  │          │       │
│     veth1        │                  │       veth2      │
└───────┼──────────┘                  └──────────┼───────┘
        │                                        │
        └──────────── virtual cable ─────────────┘
```
And in doing so, we encountered several fundamental networking concepts:
* **Network namespaces** provide isolated networking environments.
* **Network interfaces** are the actual endpoints through which a namespace participates in a network.
* A **veth pair** behaves like a virtual Ethernet cable.
* **IP addresses** identify endpoints at the network layer.
* **CIDR** determines which IP addresses belong to the same network.
* The **routing table** determines how Linux should reach a destination.
* **MAC addresses** are used by Ethernet to deliver frames on the local link.
* **ARP** maps an IPv4 address to a MAC address on the local network.
* **ICMP** is used by `ping` to test IP connectivity.

But the most important thing I took away from this experiment is that these aren't isolated concepts. They fit together.

A network interface can have an IP address.

That IP address belongs to a particular network determined by its prefix.

The routing table tells the kernel which interface to use for a destination.

If the destination is on the local Ethernet network, ARP can resolve its IP address to a MAC address.

The Ethernet frame can then travel across the link and deliver the IP packet to the destination.

All of that happened when we ran one simple command: `ping 10.0.0.2`

And instead of simply assuming that networking works, we built a tiny network ourselves and watched the pieces come together. That's a much better way to learn networking.