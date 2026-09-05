# 🔀 05 — Switches vs. Routers: Recognizing Them and Knowing What To Do

> This doc exists to fill a gap: docs 02 and 03 explain gateway/route *logic* assuming you already know whether you're looking at a switch or a router. This doc teaches you to recognize which one you're looking at, on screen, before you even start reasoning about IPs.

## 🧠 The conceptual difference, one more time, in plain terms

| | 🔌 Switch | 🌉 Router |
|---|---|---|
| **What it connects** | Devices on the **same** network | **Different** networks to each other |
| **How many networks does it "know"** | Doesn't think in terms of networks at all — just wires | One network per interface, minimum 2 interfaces |
| **Does it have an IP/mask you configure?** | No — it's invisible to addressing, purely a physical/electrical relay | Yes — every interface has its own IP + mask |
| **Do you ever add a "gateway" or "route" to it?** | Never — the concept doesn't apply | Yes — this is exactly what docs 02/03 are about |
| **Analogy** | An extension cord / power strip — just extends the same electrical circuit | A border crossing between two countries — decides what crosses over and how |

The single most important sentence to remember: **a switch never appears as an editable box with IP/Mask fields in NetPractice.** If you see a box with `IP:` and `Mask:` fields you can edit, you are looking at a **router interface**, never a switch.

## 👀 How to recognize a switch in the NetPractice diagram

A switch typically shows up as:
- A small icon (often drawn as a box with a few dots/antenna-like marks, sometimes labeled generically or not labeled with an IP at all)
- **No `IP:` / `Mask:` fields attached to it directly**
- It sits *between* hosts (or between a host and a router interface), just passing the connection through
- Multiple hosts connecting to the same switch are really just multiple hosts on the **same single network** — the switch doesn't split anything

**Visual rule of thumb:** if you trace a line from a host and it reaches *another device's IP/Mask box directly* (as if the switch weren't even there, functionally), that's the signature of a switch — it's transparent to addressing. You still need the hosts on either side of it to share the same network (same mask, same block) — but you never touch the switch itself.

## 👀 How to recognize a router in the NetPractice diagram

A router typically shows up as:
- A box (often drawn with a WiFi-style icon or a small device icon) with a name like `router R1` or `R2`
- **Multiple `interface` boxes hanging off it**, each labeled distinctly (e.g., `interface R11`, `interface R12`) — this is the dead giveaway. One router, several interfaces, each interface is its own little network membership card.
- Each of those interface boxes **does** have editable `IP:` and `Mask:` fields
- Often (from level 5 onward in the typical progression) there's also a `Routes:` field near the router box itself — this is where static routes live (see doc 02)

**Visual rule of thumb:** count the interfaces. **One device, multiple numbered interfaces, each with its own IP/Mask = router.** A device with just one IP/Mask pair and nothing else = probably a host (a PC/Mac, not a networking device at all).

## 🖼️ Worked example — reading a real diagram

Picture a diagram like this (this mirrors an actual NetPractice level structure):

```
        host A ---- interface A1 ---- [switch, invisible] ---- interface R11 ---- router R1 ---- interface R12 ---- [switch] ---- interface C1 ---- host C
```

How to read this, piece by piece:
1. `host A` and `interface A1` are the same device really — the interface box just shows A's IP/Mask
2. The connection from A1 to R11 passes through what's *visually* a switch/wire — no extra box to configure, so you don't touch anything there beyond making sure A1 and R11 share the same network (same mask, same block)
3. `interface R11` and `interface R12` both hang off the **same router**, `router R1` — this is one physical device wearing two different network "hats"
4. R11 is on A's network; R12 is on a *different* network, the one leading toward C
5. `interface C1` (host C's side) needs to match R12's network, the same way A1 matched R11's

The key insight: **A1 and R11 must be on the same network** (that's the switch-connected pair — same street, doc 01 logic). **R11 and R12 belong to different networks** (that's the router's whole job — bridging two streets). **R12 and C1 must be on the same network** (another switch-connected pair, on the other side).

## 🧩 What actually changes in your task, depending on which one you're looking at

### If you're configuring devices connected via a switch (same network)
Your only job is **doc 01 logic**: make sure IP + mask put both devices in the same block, with valid (non-network, non-broadcast, non-duplicate) host addresses. No gateway, no route, nothing else to think about for that pair.

### If you're configuring a router's interfaces
Each interface needs its own valid IP + mask for **its own** side's network (same doc 01 logic, applied per-interface, independently — R11's network has nothing to do with R12's network; they're deliberately different).

### If traffic needs to cross a router to reach a different network
Now doc 02 kicks in:
1. Every **host** on either side needs its **gateway** field set to the router interface on *its own* network (never the interface on the *other* side!)
2. The **router** needs a **route** for any network it isn't directly connected to (if there's more than one router in the chain, see doc 03 for the full multi-hop logic)

## 🚨 The most common mix-up

Setting a host's gateway to the **wrong-side** router interface — e.g., pointing host A's gateway at `R12` instead of `R11`. This fails immediately because a gateway has to be reachable *directly*, meaning it must be on the **same network** as the host. `R12` is on a completely different network from A — A literally cannot send a packet to it without going through a gateway first, which is circular and invalid.

**How to catch this yourself:** before setting any gateway, ask "is this router interface's network the *exact same block* as my host's network?" (apply doc 01's block-checking method). If not, you've picked the wrong interface.

## 🔁 What about a switch connecting more than 2 hosts?

Nothing changes conceptually — a switch with 3, 4, or more hosts attached just means **all of them share the same single network**. Every one of those hosts needs an IP in the same block, all with the same mask, all different from each other. No new rules — it's the same doc 01 logic, just applied to more devices at once instead of a pair.

## ✅ Quick recognition checklist

Before touching any field in a level, scan the diagram and ask:
1. 📦 How many separate devices with editable IP/Mask boxes do I see?
2. 🔢 Do any of them share a device name/box with multiple numbered interfaces? → that's a **router**
3. 🔌 Are two IP/Mask boxes connected with nothing else in between (no separate device)? → they're on the **same network via a switch** (or a direct link) — just match their addressing, no gateway/route needed for that pair
4. 🌉 Does the goal require reaching a device on the *other side* of a router? → now you need gateways (doc 02) and possibly chained routes (doc 03)
