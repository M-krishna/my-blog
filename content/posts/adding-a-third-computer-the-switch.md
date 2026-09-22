+++
title = "Adding a third computer, the Switch"
date = 2026-09-22

[taxonomies]
tags = ["linux", "networking", "switch", "bridge"]
+++

In the [last post](https://fazeneo.in/posts/building-a-network-from-scratch-with-iproute2/) we connected two computers with a virtual Ethernet cable and watched them talk. Two computers on a single cable is the simplest possible network:
```
   h1                          h2
    │                          │
    └──── virtual cable ───────┘
```
There was nothing to figure out. When `h1` sent something, there was only one place it could go.

Now let's ask a small question that turns out to open up a lot: **what happens when we add a third computer?**

The experiment for this post:
* See why our cable can't connect three computers.
* Build the device that can: a switch.
* Understand the one simple thing a switch does.
* Watch it work, and inspect it with our own eyes.

As before, we'll use Docker so nothing touches your real machine. Spin up an Ubuntu container:
```bash
docker run -it --rm --privileged --name netlab ubuntu:24.04 bash
```
And install the tools we'll need inside it:
```bash
apt update && apt install -y iproute2 iputils-ping tcpdump
```

# 1. Why our cable can't connect three computers
A virtual Ethernet pair, our "cable", has exactly two ends:
```
   end 1 ◄──────────────► end 2
```
One end goes into the computer, the other end goes into another. That is all a cable is: a link between two points.

So the moment we want a third computer, the cable runs out of ends. There is no way to plug three computers into a thing that has two ends.
```
   h1        h2        h3
    │         │         │
    ?    a single cable can't join three
```
We need a different piece: something with many attachment points, where every computer plugged in can reach every other. In other words, we need shared wiring, one common medium that all the computers hang off.

That device is what this post is about. But before we build it, we should understand the new problem that shared wiring creates, because the whole design of the device exists to solve it.

# 2. The new problem: who is this for?
Go back to two computers for a second. When `h1` sent a frame, it could not go anywhere except `h2`. There was no ambiguity.

Now put three computers on shared wiring, where whatever one of them sends is physically heard by all the others. A new problem appears that simply did not exist before:
> When `h1` sends something, both `h2` and `h3` receive it. So how does the data say "this is meant for `h2`, not `h3`"?

With two computers there was nothing to sort out. With three, "everyone hears everything" means each piece of data now has to carry a label saying who it is for. Otherwise every computer would treat every message as its own.

That label is an address. And it is not the IP address you might expect. It is a lower-level address that lives on the network card itself.

# 3. The MAC address, and how a card decides
Every network card is given an identifier when it is made. It is called a **MAC address** (Media Access Control address). It is six bytes, written in hexadecimal, like this:
```
02:42:ac:11:00:03
```
You don't need to memorize anything about its structure. For now it is enough to think of it as a name burned into each card, different for every card.

Data does not travel on a wire as loose bytes. It travels wrapped in an envelope called a **frame**. A frame carries, among other things, two addresses at the front:
```
┌────────────┬────────────┬──────┬───────────┬──────────┐
│  dst MAC   │  src MAC   │ type │  payload  │ checksum │
└────────────┴────────────┴──────┴───────────┴──────────┘
```
* `dst MAC` is the destination: who this frame is for.
* `src MAC` is the source: who sent it.
* `type` says what kind of thing is inside the payload.
* `payload` is the actual contents (for example, an IP packet).
* `checksum` lets the receiver detect if the frame got corrupted along the way.

Now here is the rule that solves our "who is this for?" problem. On our shared wiring, every card receives every frame. For each frame that arrives, a card asks one question:
> Is the `dst MAC` on this frame my own address?

* If yes, the card keeps the frame and hands the contents up to the operating system.
* If no, the card silently throws the frame away.

Notice the card checks the **destination address**, not the source. That makes sense: the source tells you who sent it, which cannot tell you whethere *you* are the intended recipient. Only the destination field can. And a card that isn't the intended recipient just drops the frame quietly. It does not reply, it does not complain, it does nothing.

So on shared wiring, sending to one computer really means: put the frame on the wire with that computer's MAC address in the destination field, and trust everyone else to ignore it.

## One special address
There is one destination value that every card accepts, no matter what:
```
ff:ff:ff:ff:ff:ff
``` 
This is called the **broadcast address**. A frame sent to it means for everyone in the wire, and every card keeps it instead of dropping it. It is way to say "everybody, listen to this one". We'll see it in action shortly, because it is how a computer finds another computer's MAC address in the first place. (That mechanism is [ARP](https://fazeneo.in/posts/how-computers-figure-out-ip-addresses-are-in-same-network/), which we covered before.)

So we now have a way to label frames. What we still need is the physical device that carries them between three or more computers.

# 4. The simple device that works badly: a hub
The simplest possible device with many attachment points is called a **hub**. Each attachment point is called a **port**. A hub does exactly one thing: whatever frame arrives on one port, it copies out of every other port.
```
             ┌───────── hub ─────────┐
   h1 ──────►│ port 1                │
             │        copies to all  │
   h2 ◄──────│ port 2      other     │
             │             ports     │
   h3 ◄──────│ port 3                │
             └───────────────────────┘
```
Combined with the card rule from the last section, this actually works. If `h1` sends a frame to `h2`, the hub copies it to both `h2` and `h3`. `h2` sees its own address in the destination and keeps it. `h3` sees an address that isn't its own and drops it. The right computer gets the frame.

But look at what happened: `h3` was disturbed for no reason. It received a frame, checked it, and threw it away, all for a conversation it had nothing to do with. With three computers that is a small waste. With three hundred computers, every single conversation bothers all the others. Everyone is constantly receiving and discarding frames meant for someone else.

We can do much better. The device just needs to pay a little attention instead of blindly copying.

# 5. The switch
A **switch** looks the same from the outside as a hub: a box with many ports, one computer plugged into each. The difference is that a switch reads each frame and makes a decision, instead of copying blindly.

The whole cleaverness is a single table. The switch keeps a table that maps each MAC address to the port it lives on:
```
   MAC address          port
   ----------------     ------
   02:...:01            port 1
   02:...:02            port 2
```
That table is the entire secret of a switch. Everything else follows from how the table gets filled and how it gets used.

## How the table gets filled: learning
The table starts completely empty. The switch is never told anything. It fills the table by watching traffic that was passing through it anyway.

Here is the trick. Every frame carries a source MAC (who sent it). So whenever a frame arrives on a port, the switch reads that source address and records:
> "This MAC address is reachable through this port."

It learned that fact for free, just by looking at a frame that was passing by. Over time, as computers send frames, the switch quietly builds up a picture of which computer is on which port. This is called **learning**.

## How the table gets used: forwarding
When a frame arrives and the switch needs to send it onward, it looks at the frames destination MAC and checks its table. There are three cases:
1. **The destination is in the table**. The switch knows exactly which port that computer is on, so it sends the frame out of that one port only. Nobody else is disturbed. This is the good case, and it is the whole point of the switch.
2. **The destination is not in the table yet.** The switch has never seen that address, so it doesn't know where to send it. Rather than give up, it sends the frame out of every port except the one it came in on. This is called **flooding**. It is safe: every computer that isn't the intended recipient will drop the frame anyway (that's the card rule from earlier). The only cost is a little wasted effort. And the reply that comes back will teach the switch where that computer is, so it usually only has to flood once.
3. **The destination is the broadcast address**. (`ff:ff:ff:ff:ff:ff`) This frame is meant for everyone, so the switch floods it on purpose, out of every port except the incoming one.

That is the complete behaviour of a switch. It learns by reading source addresses, and it forwards by looking up destination addresses. Nothing more.

A few things worth noticing:
* The switch never changes the frames. What goes in comes out unchanged.
* The switch has no address of its own in this job. The computers using it don't even know it is there. It is invisible.
* The switch only understands MAC addresses. It has no idea what an IP address is.

## A note on names
You will hear "switch" and "bridge" used for this. They are the same device. "Bridge" is the older word, from when the box had just two ports and joined two segments together. "Switch" is what it got called once it had many ports. Linux uses the word "bridge" for its software version, which is what we'll create next. So in this post, switch and bridge mean the same thing.

# 6. Building it
Let's build three computers connected by one switch. We'll reuse the network namespace idea from the last post: a **network namespace** is an isolated networking environment on our machine that behaves like a separate computer, with its own interfaces, addresses, and tables.

First, create the switch. In Linux the software switch is a "bridge":
```bash
ip link add br0 type bridge
ip link set br0 up
```
The first line creates a bridge named `br0`. The second line turns it ON (interfaces and devices start switched off and ignore everything until brought up).

Now we create our first computer, `h1`, and plug it into the switch. I'll spell out every line for `h1`, then repeat the pattern for others.
```bash
ip netns add h1
```
This creates an empty computer.
```bash
ip link add h1-eth type veth peer name h1-br
```
This creates our cable, a veth pair, with two ends named `h1-eth` and `h1-br`. One end (`h1-eth`) will go inside the computer and become its network card. The other end (`h1-br`) will plug into the switch.
```bash
ip link set h1-br master br0
ip link set h1-br up
```
The first line plugs the `h1-br` end into the switch. `master br0` means "attach this to the bridge `br0` as one of its ports". The second line turns that port ON.
```bash
ip link set h1-eth netns h1
```
This moves the other end of the cable, `h1-eth`, inside the computer. It now appears as that computer's network card, and disappears from the outside.
```bash
ip netns exec h1 ip link set lo up
ip netns exec h1 ip link set h1-eth up
ip netns exec ip addr add 10.0.1.10/24 dev h1-eth
```
`ip netns exec h1 ...` runs a command inside the computer `h1`. The first line turns ON the loopback interface (the computer's ability to talk to itself, good hygiene). The second turns on the network card. The third gives the card an IP address, `10.0.1.10`, with a `/24` prefix.

Now `h2` and `h3`, exactly the same pattern, just different names and addresses:
```bash
ip netns add h2
ip link add h2-eth type veth peer name h2-br
ip link set h2-br master br0
ip link set h2-br up
ip link set h2-eth netns h2
ip netns exec h2 ip link set lo up
ip netns exec h2 ip link set h2-eth up
ip netns exec h2 ip addr add 10.0.1.11/24 dev h2-eth

ip netns add h3
ip link add h3-eth type veth peer name h3-br
ip link set h3-br master br0
ip link set h3-br up
ip link set h3-eth netns h3
ip netns exec h3 ip link set lo up
ip netns exec h3 ip link set h3-eth up
ip netns exec h3 ip addr add 10.0.1.12/24 dev h3-eth
```
All three computers now hang off one switch, and all three addresses are in the same network, `10.0.1.0/24`:
```
   h1            h2            h3
 10.0.1.10     10.0.1.11     10.0.1.12
    │             │             │
 h1-eth        h2-eth        h3-eth
    │             │             │
 h1-br         h2-br         h3-br
    │             │             │
    └─────────[  br0  ]─────────┘
```

# 7. Watching it work
Now let's watch the switch learn and forward, and confirm everything we just described.

First, ask `h1` what MAC addresses it currently knows about:
```bash
ip netns exec h1 ip neigh
```
Nothing prints. This is the neighbour table, the list of "IP address to MAC address" mappings `h1` has learned, and right now it is empty.`h1` does not yet know any other computer's MAC address.

Let's watch the traffic while we make it talk. Open a second terminal into the same container:
```bash
docker exec -it netlab bash
```
And in that second terminal, start capturing frames on the switch. `tcpdump` prints frames it sees on an interface. The `-e` flag makes it show MAC addresses, which is what we care about here:
```bash
tcpdump -i br0 -n -e
```
Leave that running. Back in the first terminal, ping `h2` from `h1`, just once:
```bash
ip netns exec h1 ping -c1 10.0.1.11
```
In the capture you'll see something like this (your MAC address will differ):
```
ARP, Request who-has 10.0.1.11 tell 10.0.1.10, ...
   dst MAC ff:ff:ff:ff:ff:ff
ARP, Reply 10.0.1.11 is-at <h2 MAC>, ...
   dst MAC <h1 MAC>
IP 10.0.1.10 > 10.0.1.11: ICMP echo request
IP 10.0.1.11 > 10.0.1.10: ICMP echo reply
```
Read what happened, top to bottom:
* `h1` did not know `h2`'s MAC address, so it asked, using the broadcast address `ff:ff:ff:ff:ff:ff`. This is the "everyone listen" frame. The switch flooded it to `h2` and `h3`.
* Both `h2` and `h3` received the question. `h3` saw it was about an address that wasn't its own and ignored it. `h2` saw its own address and replied, this time directly to `h1`, not to everyone.
* Now that `h1` had `h2`'s MAC address, the actual ping (the ICMP echo request) went out, and the reply came back.

That broadcast question and direct reply is ARP, which found `h2`'s MAC. What matters for this post is the switch's part: it flooded the broadcast question, and delivered a direct reply.

Now look at what `h1` learned:
```bash
ip netns exec h1 ip neigh
```
This time there is an entry mapping `10.0.1.11` to `h2`'s MAC address, marked something like `REACHABLE`. That mapping did not exist a minute ago.

And, most importantly for this post, look at what the switch itself learned. The switch keeps its own MAC-to-port table, called the forwarding database:
```bash
bridge fdb show | grep -v permanent
```
You should see your computer's MAC addresses listed against their ports (`h1-br`, `h2-br`). The switch built that table by itself, purely by watching the frames go by. Nobody typed it in. This is learning, made visible.

# 8. Is an ARP request the same thing as flooding?
In the capture above, two things both ended up reaching every computer: the ARP request, and the switch flooding it. It is easy to treat them as one thing. But they are different, and the difference is worth getting straight: it comes down to *who is deciding, and what they are deciding about.*

An **ARP request** is a decision made by the sending computer. `h1` had an IP address and wanted the matching MAC, so it chose to ask a question, and it addressed that question to the broadcast address because it had no better option. ARP is about the contents: "I know an IP, I want the MAC." It is a question a computer asks.

**Flooding** is a decision made by the switch. The switch does not know or care what is inside the frame. It looked at the destination MAC and decided how many ports to send the frame out of. Flooding is about delivery: "send this out of every port except the one it came in on." It is an action a switch takes.

So when the ARP request got flooded, it was not flooded *because* it was ARP. The switch had no idea it was ARP. It flooded it purely because the destination MAC was the broadcast address, `ff:ff:ff:ff:ff:ff`.

And that points to something worth knowing: an ARP request is only one of the two situations that make a switch flood. A switch floods a frame when:
1. The destination MAC is the broadcast address. An ARP request falls here.
2. The destination MAC is a specific address the switch has not learned yet. This has nothing to do with ARP. If a computer sends an ordinary frame to a specific MAC, but the switch's table doesn't have that MAC yet, the switch floods it anyway, simply because it doesn't know which single port to use. The reply teaches it, and it won't need to flood to that address again.

So all broadcast frames get flooded, but not everything that gets flooded is a broadcast. An ARP request is flooded for the first reason. An ordinary frame to an address the switch hasn't learned is flooded for the second. In every case, the switch is deciding based only on the destination MAC and its own table, never on what the frame is for.

# 9. What we built: a broadcast domain
Step back and look at what we have. Three computers on one switch. A frame sent to one computer reaches only that computer. A frame sent to the broadcast address reaches all of them.

That last property has a name. A set of computers wired together such that a broadcast frame from any one of them reaches all the others is called a **broadcast domain**. Our switch, with its three computers, is one broadcast domain. One shared space where a shout reaches everyone.

This is a term worth holding on to, because everything that comes next is built on it. A single network, the kind where computers can find each other by shouting (ARP) and reach each other directly, is really just one broadcast domain. The size of that shared space, who can hear whom, is the thing that decides what counts as "the same network."

# 10. What's next
We started with two computers on a cable, and now we have as many computers as we like sharing one network, connected by a switch that quietly learns where everyone is and delivers frames straight to them.

But every one those computers is in the same network, `10.0.1.0/24`. They can all hear each other's shouts. What happens when we want to reach a computer in a *different* network, one that our shout can't reach at all?

That is the wall that a switch cannot get past, no matter how many ports it has. Crossing from one network to another needs a different device: **a router**. That is the next post.

For now, the piece to remember is this:
```
   One switch
        │
        ▼
   One broadcast domain
        │
        ▼
   One network, where computers find each other by shouting
```