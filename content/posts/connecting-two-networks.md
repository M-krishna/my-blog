+++
title = "Connecting two networks, the Router"
date = 2026-09-26

[taxonomies]
tags = ["linux", "networking", "router", "switch", "bridge"]
+++
In the [last post](https://fazeneo.in/posts/adding-a-third-computer-the-switch/) we put several computers on one switch and watched them talk. They could all reach each other because they shared one network, one space where a shout (ARP request) reaches everyone.

At the end of that post we hit a wall:
> **What happens when we want to reach a computer in a *different* network, one that our shout can't reach at all?**

That is this post. We'll build two separate networks, watch them fail to talk to each other in two different ways, and then build and configure the device that connects them: a **router**.

The experiment:
* Build two separate networks that cannot reach each other.
* Watch the failure, and understand exactly why it happens.
* Build a router, a machine that sits in both networks at once.
* Configure it so the two networks can finally talk.
* Watch a single packet cross from one network to the other, and see what changes along the way. 

As before, everything runs inside a throwaway Docker container so nothing touches your real machine:
```bash
docker run -it --rm --privileged --name netlab ubuntu:24.04 bash
```
And inside it:
```bash
apt update && apt install -y iproute2 iputils-ping tcpdump
```

# 1. Building two separate networks
We'll build two separate networks side by side. Each is a switch with one computer plugged into it. The two networks use different address ranges:
* Network 1: `10.0.1.0/24`, with switch `br0` and computer `h1` at `10.0.1.10`.
* Network 2: `10.0.2.0/24`, with switch `br1` and computer `h2` at `10.0.2.10`.

First, the two switches:
```bash
ip link add br0 type bridge
ip link set br0 up
ip link add br1 type bridge
ip link set br1 up
```
Now `h1`, plugged into `br0`. This is the same pattern from the last post, so I'll show it without re-explaining every line:
```bash
ip netns add h1 # h1 computer
ip link add h1-eth type veth peer name h1-br # Ethernet cable
ip link set h1-br master br0 # connect h1-br end to the bridge
ip link set h1-br up # bring h1-br end of the cable UP
ip link set h1-eth netns h1 # connect h1-eth end to the h1 computer
ip netns exec h1 ip link set lo up # bring lo interface UP
ip netns exec h1 ip link set h1-eth up # bring h1-eth interface UP
ip netns exec h1 ip addr add 10.0.1.10/24 dev h1-eth # assign IP addr to h1-eth interface
```
And `h2`, plugged into `br1`, in the other network:
```bash
ip netns add h2
ip link add h2-eth type veth peer name h2-br
ip link set h2-br master br1
ip link set h2-br up
ip link set h2-eth netns h2
ip netns exec h2 ip link set lo up
ip netns exec h2 ip link set h2-eth up
ip netns exec h2 ip addr add 10.0.2.10/24 dev h2-eth
```
Here is what we have. Two networks, with no connection between them:
```
   Network 1  (10.0.1.0/24)        Network 2  (10.0.2.0/24)

        h1                              h2
     10.0.1.10                      10.0.2.10
        │                              │
     [  br0  ]                      [  br1  ]
```
`br0` and `br1` are two completely separate switches. Nothing joins them. They may as well be in different buildings.

# 2. The wall: they cannot talk
Let's try to reach `h2` from `h1`:
```bash
ip netns exec h1 ping -c2 10.0.2.10
```
You get this, right away:
```
connect: Network is unreachable
```
Two things are worth noticing, and both are clues.

First, it failed **instantly**. There was no waiting, no timeout. Compare that to the last post, where a ping to a live computer took a moment while ARP went out and came back. This failed before anything like that could happen.

Second, nothing went on the wire at all. If you run a capture on `br0` while you retry the ping, you'll see it stay silent:
```bash
tcpdump -i br0 -n
```
No ARP request, nothing. In the last post, even reaching a computer meant `h1` first shouted "who has this address?" Here, `h1` did not even shout. It gave up before sending a single frame.

So `h1` decided, on its own, before touching the wire, that this was hopeless. Why? To answer that, we need to meet the thing `h1` consulted to make that decision.

# 3. Why it failed instantly: the route table
Before a computer sends any packet, it has to answer one question: *can I deliver this myself, directly or not?* It answers that question by looking at a list of instructions called the **route table**.

Let's look at `h1`'s route table:
```bash
ip netns exec h1 ip route
```
You'll see a single line, something like:
```
10.0.1.0/24 dev h1-eth proto kernel scope link src 10.0.1.10
```
Read it as an instruction in plain English:
> **To reach anything in `10.0.1.0/24`, send it directly out of the interface `h1-eth`. Those addresses are right here on this wire.**

That is the meaning of `scope link`: the destinations in that range are reachable directly, on this wire, by shouting for them. And `proto kernel` means nobody typed this line in. The kernel created it automatically the moment we gave `h1` the address `10.0.1.10/24`. Giving a computer an address with a `/24` prefix silently adds one route: "my own network is right here."

Now the failure makes sense. When we asked `h1` to reach `10.0.2.10`, it went through its route table looking for an instruction that covers that address:
* `10.0.1.0/24`? No, `10.0.2.10` is not in that range.
* ...and that's the only line. There is nothing else.

No instruction covers `10.0.2.10`. So `h1` has no idea what to do with it, and it says so immediately: `Network is unreachable`. That message means exactly one thing: **I have no route that covers that destination**. It is not "the destination is down" or "the network is broken." It is "I don't even have instructions for getting there."

This is the real meaning of two networks being separate. It is not the wiring. It is that neither computer has any instruction for reaching the other.

# 4. What we need: a machine that sits in both networks
So `h1` needs a new instruction. But what could that instruction say? It cannot say "`10.0.2.10` is here on my wire", because it isn't. A shout for it would reach nobody.

The instruction has to say something different:
> **To reach network 2, don't try to deliver it yourself. Hand the packet to a helper on your own wire, and let the helper pass it along.**

For that to work, the helper must be two things at once:
1. **Reachable by `h1` directly**. It must be on `h1`'s own wire (network 1), so `h1` can hand it a frame by shouting for it, just like reaching any normal neighbour.

2. **Also present in network 2**. So that once it has the packet, it can turn around and deliver it into the other network.

A machine that sits in two networks at once, and is willing to pass packets between them, is a **router**. That is the whole idea. Not a special box. Just a machine with one foot in each network.

To be in two networks at once, the machine needs two network cards, one plugged into each switch:
```
   Network 1                         Network 2

     h1                                  h2
  10.0.1.10                          10.0.2.10
     │                                  │
  [  br0  ]                          [  br1  ]
     │                                  │
     └──────────────  r1  ─────────────┘
              (a leg in each network)
```
Let's build `r1`.

# 5. Building the router
The router is just another computer (namespace), but with two cables, one into each switch.

Create the machine:
```bash
ip netns add r1
```
Give it a leg into `br0` (network 1):
```bash
ip link add r1-eth0 type veth peer name r1-br0
ip link set r1-br0 master br0
ip link set r1-br0 up
ip link set r1-eth0 netns r1
ip netns exec r1 ip link set r1-eth0 up
```
And a second leg into `br1` (network 2):
```bash
ip link add r1-eth1 type veth peer name r1-br1
ip link set r1-br1 master br1
ip link set r1-br1 up
ip link set r1-eth1 netns r1
ip netns exec r1 ip link set r1-eth1 up
```
Now the important part. Each leg gets an address **in the network that leg gives in:**
```bash
ip netns exec r1 ip addr add 10.0.1.1/24 dev r1-eth0
ip netns exec r1 ip addr add 10.0.2.1/24 dev r1-eth1
```
So the router has two addresses. `10.0.1.1` is its face in network 1, the address `h1` will hand packets to. `10.0.2.1` is its face in network 2, the address `h2` will use. A router does not have "an address". It has one address per network it touches.

Look at the router's route table now:
```bash
ip netns exec r1 ip route
```
Two lines this time, one per network:
```
10.0.1.0/24 dev r1-eth0 proto kernel scope link src 10.0.1.1
10.0.2.0/24 dev r1-eth1 proto kernel scope link src 10.0.2.1
```
Because the router has an address in each network, the kernel auto-created an instruction for each. The router considers *both* networks to be right here on its own wires. It is the only machine so far that can reach both directly. That is exactly what having a leg in each network buys us.

Now, we want to verify whether our router can talk to both the devices (`h1` and `h2`). To verify that, we can ping those machines from the router.
```bash
ip netns exec r1 ping -c1 10.0.1.10
ip netns exec r1 ping -c1 10.0.2.10
```
You should be able to see a successful ping and 0% packet loss. If you see otherwise, that means there is something wrong with your setup. Fix that before proceeding to the next step.

# 6. Telling the hosts to use the router
The router exists, but `h1` still doesn't know about it. Remember, `h1`'s route table has only the one auto-created line for its own network. We need to add the new instruction from section 4.

Give `h1` a route that sends network-2 traffic to the router:
```bash
ip netns exec h1 ip route add 10.0.2.0/24 via 10.0.1.1
```
Read it as: "to reach anything in `10.0.2.0/24`, hand it to `10.0.1.1`. That word `via` is the key. Unlike the auto-created line, this instruction does not say "deliver directly." It says "give it to this helper instead." And `10.0.1.1` is the router's leg in network 1, which `h1` can reach directly, so this works.

Now, a point that trips up almost everyone. We have told `h1` how to reach network 2. But we have said nothing to `h2` about how to reach network 1. And a conversation needs both directions: `h1`'s ping has to get to `h2`, and `h2`'s reply has to get back to `h1`. Those are two separate journeys, and each needs its own instruction.

So `h2` needs the mirror-image route:
```bash
ip netns exec h2 ip route add 10.0.1.0/24 via 10.0.2.1
```
"To reach network 1, hand it to `10.0.2.1`," which is the router's leg in network 2. We'll come back to why this second line matters so much in section 8. For now, add it.

# 7. The second wall: the ping that hangs
Everything looks ready. The router is in both networks. Both hosts have routes pointing at it. Let's ping:
```bash
ip netns exec h1 ping -c2 10.0.2.10
```
This time the failure is different. It does not say `Network is unreachable`. Instead it just **hangs**, and eventually times out with no reply.

That change is itself progress. `h1` now has a route for network 2, so it no longer gives up instantly. It actually sent the packet. So the packet got somewhere, and then died somewhere. Let's find out where.

Capture on both wires. In one terminal, watch the far side, `br1`, the wire toward `h2`:
```bash
tcpdump -i br1 -n
```
In another terminal, ping again:
```bash
ip netns exec h1 ping -c2 10.0.2.10
```
Nothing appears on `br1`. Now watch the near side, `br0`, and ping once more:
```bash
tcpdump -i br0 -n
```
On `br0` you *do* see the packet: `h1` sends it toward the router. So the packet reached the router. But it never came out the other side onto `br1`. The router received it and threw it away.

Why would the router drop a packet it clearly knows how to deliver? Because of a rule that is easy to miss:
> **A normal computer does not pass along packets that aren't addressed to it. If a packet arrives whose destination is some other machine, the computer's default reaction is: "not mine", and it drops it.**

This is the real difference between a plain computer and a router. Our `r1` has two legs and two addresses, but out of the box it still behaves like an ordinary computer: it only handles packets meant for itself. Passing along *other machines'* packets, taking a packet in on one leg and sending it out another, is a separate behaviour called **forwarding**, and it is switched off by default.

So the only thing standing between us and a working router is one setting. Let's check it:
```bash
ip netns exec r1 sysctl net.ipv4.ip_forward
```
`sysctl` reads and writes kernel settings. `net.ipv4.ip_forward` is the on-or-off switch for forwarding. On a normal Linux machine this reads `0`, meaning off. (Inside Docker you may find it already set to `1`, because Docker itself does forwarding; if so, set it to `0` first with command below and watch the ping fail, so you can see the effect for yourself.)

Turn it ON:
```bash
ip netns exec r1 sysctl -w net.ipv4.ip_forward=1
```
That single line is the entire difference between a host and a router. Now ping again:
```bash
ip netns exec h1 ping -c2 10.0.2.10
```
Replies come back. Two computers, in two separate networks, are finally talking, through the router.

# 8. The payoff: what changes as the packet crosses
We could stop here. But the most interesting part is *how* the packet crossed, and it teaches you something that holds for every packet on the internet.

Think about what the router actually did. A frame can only travel within one network, and its MAC addresses only make sense on that one wire (this is from the last two posts). So the router cannot simply pass the frame through. It **receives** the frame on `br0`, unwraps it to find the packet inside, decides where that packet should go next, and builds a **brand-new frame** to send out on `br1`. The journey is really two separate frame trips stiched together:
```
   trip 1: h1  ──►  router      (on br0, network 1)
   trip 2: router  ──►  h2      (on br1, network 2)
```
Let's watch both trips at once. The `-e` flag makes `tcpdump` show the MAC addresses, which is the whole point here. Open two terminals:
```bash
tcpdump -i br0 -n -e icmp
```
```bash
tcpdump -i br1 -n -e icmp
```
Then send a single ping from a third terminal:
```bash
ip netns exec h1 ping -c1 10.0.2.10
```
On `br0` (the near wire), the request looks like this (your MAC addresses will differ):
```
<h1 MAC> > <r1-eth0 MAC>, IPv4, 10.0.1.10 > 10.0.2.10: ICMP echo request
```
On `br1` (the far wire), the *same* request looks like this:
```bash
<r1-eth1 MAC> > <h2 MAC>, IPv4, 10.0.1.10 > 10.0.2.10: ICMP echo request
```
Put them side by side and read carefully. This is the whole lesson:
```
          source MAC     dest MAC       source IP    dest IP
br0       h1             r1-eth0        10.0.1.10    10.0.2.10
br1       r1-eth1        h2             10.0.1.10    10.0.2.10
```
Look at what changed and what did not:
* **The MAC addresses changed completely**. On `br0` the frame is "from `h1`, to the router." On `br1` it is "from the router, to `h2`." Different wire, different hardware conversation. The router tore off the old frame and built a fresh one for the second wire.
* **The IP addresses did not change at all.** Both wires show `10.0.1.10 > 10.0.2.10`. The packet always knows it came from `h1` and is going to `h2`, no matter how many routers it passes through.

That is the core idea of how the whole internet moves data:
> **The IP addresses say where the packet ultimately came from and where it is ultimately going. They stay the same the whole way. The MAC addresses say who is handling the frame to whom on this one wire, right now. They are rewritten at every hop.**

And this is *why* the IP addresses must stay frozen: `h2` sees the request as coming from `10.0.1.10`. That is how `h2` knows where to send the reply. If the router had rewritten the source IP, the reply would have no way home.

One small bonus. Every IP packet carries a number called **TTL** (time to live). It starts at a value (Linux uses 64) and every router that forwards the packet lowers it by one. Its purpose is to stop packets that get stuck in a loop from circling forever: eventually the number hits zero and the packet is discarded. If you add `-v` to your capture (`tcpdump -i br1 -n -e -v icmp`), you'll see the TTL on `br1` is one lower than on `br0`. You just caught the router in the act of decrementing it. It is a free hop counter: a reply that comes back with a TTL of 63 instead of 64 has crossed exactly one router.

# 9. A packet arriving is not the same as a reply coming back
Back in section 6, I made you add a second route, the one on `h2` pointing back at the router, and promised to explain why it matters. Here is the payoff, and it is one of the most useful things to understand in all of networking.

Let's deliberately break it. Remove `h2`'s return route:
```bash
ip netns exec h2 ip route del 10.0.1.0/24 via 10.0.2.1
```
Now capture the far wire and ping:
```bash
tcpdump -i br1 -n
```
```bash
ip netns exec h1 ping -c2 10.0.2.10
```
Watch what happens on `br1`: the echo *requests* arrive at `h2` just fine, but no replies ever come back. The traffic is one-way.

Here is why. The request reaches `h2` completely successfully. `h2` receives it, sees a ping addressed to itself, and wants to reply. But now `h2` has to send that reply back to `10.0.1.10`, and it runs the exact same check `h1` ran back in section 3.
* Is `10.0.1.10` in my own network? (`10.0.2.0/24`)? No.
* Do I have a route that covers it? No, we just deleted it.
* Result: `Network is unreachable`. The reply is never sent.

So the request arrives, but the reply cannot leave. From `h1`'s point of view, the ping just times out, exactly the same symptom as a packet that never arrived at all. But the truth is completely different: the packet arrived perfectly, and it was the *reply* that failed.

This is the lesson:
> **A packet reaching its destination does not mean a reply can get back. The forward path and the return path are two separate problems. Each end must independently know how to reach the other.**

This is the single most common routing bug in real life, and it is maddening precisely because "but the packet clearly arrives!" sends people looking in the wrong place. Now you know the fingerprint: if you capture the far wire and see requests arriving but no replies leaving, suspect a missing return route.

Put the route back so things work again:
```bash
ip netns exec h2 ip route add 10.0.1.0/24 via 10.0.2.1
```

# 10. What's next
Look back at what we did. We gave `h1` an instruction: "for network 2, go via the router." That worked. But it only works because we knew, in advance, exactly which network `h1` wanted to reach, and we wrote a specific route for it.

Now think about your own laptop. It talks to millions of different networks all over the world. You obviously did not write a route for each one. So how does a computer reach a network it has never heard of and has no specific instruction for?

The answer is one beautifully simple route, a catch-all that says "for anything I don't have a specific instruction for, just send it to my router and let it worry about the rest." That route is called the **default gateway**, and it is what turns a computer from "can reach the one network I was told about" into "can reach anywhere." That is the next post.

For now, the piece to remember:
```
Two networks, no connection
        │
        ▼
A router: one machine with a leg in each
        │
        ▼
Plus a route on each side pointing at it
        │
        ▼
The networks can finally talk
```