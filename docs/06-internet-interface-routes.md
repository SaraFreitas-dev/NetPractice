# 🌐 06 — Upstream Nodes & Return Routes (Calculating the Destination Side)

> This doc explains one specific pattern: when a node representing an external network (often labeled "Internet," an ISP, or any upstream connection) needs a route back into your topology, how do you figure out exactly what to write? Same slow pace as doc 04, with many worked examples.

## 🎯 The situation this doc covers

In most of docs 02 and 03, hosts and routers used a `default` route (`0.0.0.0/0`) — "send anything unrecognized to my one gateway," which works because there's only one way out for a host or a simple router.

But sometimes a node sits on the *outside* of your topology, representing a larger network that could reach many different internal segments through the same connection point. That kind of node can't just say "default, send everything the same way" — it needs to know, **network by network**, which internal segment is reachable through which entry point. This shows up as a route shaped like:

```
??? → next_hop
```

Where `next_hop` (the right side) is usually already given to you, fixed. Your job is figuring out the `???` (the left side) — the specific destination network.

## 🧠 Why a specific network, and not just "default," in this case

Think about the difference in position. A host or a simple router only has **one exit** — so "send everything unrecognized the same way" makes sense as a rule. A node representing an external/upstream network, though, sits in a position where it could potentially need to distinguish between *multiple* internal networks reachable through the same router — it needs to know exactly which internal network corresponds to which path, rather than lumping everything into one generic exit. So instead of `default`, this kind of route needs the internal network spelled out explicitly.

## 📍 The core method — 4 steps, every time

1. **Identify which internal device or segment this route needs to make reachable.** This comes from the goal you're solving, or from tracing which host sits behind which interface.
2. **Find that device's IP and mask.** It's almost always given somewhere in the diagram, usually in a locked/grayed field.
3. **Calculate that device's network address** — this is exactly the block-size math from doc 04: find which block the IP falls into, and take the *start* of that block (the network address), not the IP itself.
4. **Write the route as `network/mask → next_hop`**, keeping whatever next-hop value was already given to you on the right side unchanged.

That's the whole method. Every example below just applies these same 4 steps to different numbers.

---

## 🏋️ Worked example 1 — a clean /24, no tricky block math

**Given:**
- An internal interface: `192.168.44.10` / `255.255.255.0`
- The next hop (already given): `78.90.12.1`

**Step 1 — what needs to be reachable?** Whatever host or segment sits behind this interface.

**Step 2 — IP and mask:** `192.168.44.10 /24`

**Step 3 — calculate the network:** `/24` means the entire last octet is host, and the first 3 octets are network, copied exactly as given. Network = `192.168.44.0`. No block-size math needed here at all — this is the simplest case.

**Step 4 — write the route:**
```
192.168.44.0/24  →  78.90.12.1
```

---

## 🏋️ Worked example 2 — a /25, needing an actual block calculation

**Given:**
- An internal interface: `100.119.229.227` / `255.255.255.128`
- The next hop (already given): `163.172.250.12`

**Step 1 — what needs to be reachable?** The host/segment behind this interface.

**Step 2 — IP and mask:** `100.119.229.227 /25`

**Step 3 — calculate the network:**
```
Mask /25 → mixed octet is the last one, value 128, block size = 256-128 = 128
Blocks: 0-127, 128-255
227 falls in 128-255
Network = 100.119.229.128
```

**Step 4 — write the route:**
```
100.119.229.128/25  →  163.172.250.12
```

🧠 **Why this one needs care:** it's tempting to just copy the given IP wholesale into the route, but the network address is the *start of the block* the IP falls in — not the IP itself. Always run the block calculation rather than assuming the given number is already the network address.

---

## 🏋️ Worked example 3 — a /30 link network (small block, easy to miscalculate)

**Given:**
- An internal interface: `10.5.5.2` / `255.255.255.252`
- The next hop (already given): `55.66.77.1`

**Step 1 — what needs to be reachable?** Whatever's behind this small link (often a router-to-router segment).

**Step 2 — IP and mask:** `10.5.5.2 /30`

**Step 3 — calculate the network:**
```
Mask /30 → last octet, value 252, block size = 256-252 = 4
Blocks: 0-3, 4-7, 8-11...
2 falls in 0-3
Network = 10.5.5.0
```

**Step 4 — write the route:**
```
10.5.5.0/30  →  55.66.77.1
```

---

## 🏋️ Worked example 4 — mixed octet NOT in the last position

**Given:**
- An internal interface: `172.16.14.200` / `255.255.252.0`
- The next hop (already given): `9.9.9.1`

**Step 1 — what needs to be reachable?** The segment behind this interface.

**Step 2 — IP and mask:** `172.16.14.200 /22`

**Step 3 — calculate the network:**
```
Mask /22 → mixed octet is the 3rd one, value 252, block size = 256-252 = 4
Blocks in 3rd octet: 0-3, 4-7, ..., 12-15...
14 falls in 12-15
Network = 172.16.12.0   (4th octet resets to 0 — the network address always has every host bit at 0, and here the 4th octet is entirely host)
```

**Step 4 — write the route:**
```
172.16.12.0/22  →  9.9.9.1
```

🧠 **Why this one trips people up:** the temptation is to write `172.16.14.0/22` — just zeroing the last octet while keeping the 3rd octet as given. That's wrong. You have to find the actual **start of the block** in the mixed octet (`12`, not `14`), because `14` is just some value that happens to fall inside the `12–15` block — it isn't the network address itself.

---

## 🏋️ Worked example 5 — two internal networks, two separate route entries needed

Sometimes the outside node needs routes to more than one internal network — for instance, if two different hosts on two different segments both need to be reachable from outside.

**Given:**
- Network A: `192.168.10.0/24`, reachable via next hop `41.42.43.1`
- Network B: `192.168.20.0/24`, reachable via the *same* next hop `41.42.43.1` (same router, different internal segment)

You need **two separate lines**, not one combined entry — each internal network gets its own route, even when they share the same next hop:

```
192.168.10.0/24  →  41.42.43.1
192.168.20.0/24  →  41.42.43.1
```

🧠 **Why not just one route covering both?** These are two genuinely *different* blocks — there's no single mask that covers exactly these two networks and nothing outside of them. Every distinct internal network in your topology needs its own line in this kind of table, even if several of them point to the same next hop.

---

## 📝 Practice — try these yourself, answers below

**1.** An interface shows `203.0.113.55` / `255.255.255.192`, next hop given as `88.77.66.1`. Write the full route.

**2.** An interface shows `10.20.30.4` / `255.255.255.248`, next hop given as `12.13.14.1`. Write the full route.

**3.** An interface shows `172.30.99.5` / `255.255.240.0`, next hop given as `1.2.3.4`. Write the full route. (Careful — the mixed octet isn't the last one here.)

<details>
<summary>Click to check your answers</summary>

**1.** Mask `/26` → block size `256-192=64`. Blocks: `0-63,64-127,128-191,192-255`. `55` falls in `0-63`. Network = `203.0.113.0`. Route: `203.0.113.0/26 → 88.77.66.1`

**2.** Mask `/29` → block size `256-248=8`. Blocks: `0-7,8-15,16-23,24-31...`. `4` falls in `0-7`. Network = `10.20.30.0`. Route: `10.20.30.0/29 → 12.13.14.1`

**3.** Mask `/20` → mixed octet is the 3rd, value `240`, block size `256-240=16`. Blocks in 3rd octet: `0-15,16-31,32-47...,96-111...`. `99` falls in `96-111`. Network = `172.30.96.0`. Route: `172.30.96.0/20 → 1.2.3.4`

</details>

## ✅ One-paragraph summary

> When a node needs a *specific* route instead of a plain `default`, the right side (next hop) is usually already given — the task is finding the left side (destination network). Identify which internal device or segment needs to be reachable, find its IP and mask, run the block-size calculation from doc 04, and use the **network address** you land on (the start of the block, not the given IP itself) as the left side of the route. If more than one internal network needs to be reachable, each gets its own separate route line, even when they share the same next hop.
