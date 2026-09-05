# 🚦 02 — Gateway & Static Routes

## 🔀 Before anything else: are you even looking at a router?

Everything in this document assumes you've already identified that you're dealing with a **router** and not just a switch or a direct link. This distinction matters enormously, because the two devices look similar at first glance but need completely different treatment:

- A **switch** connects devices that are **already on the same network**. You never configure a gateway or route for it — it doesn't have IP/Mask fields of its own, and it's invisible to the addressing logic covered in doc 01. If two hosts are linked through a switch, your only job is making sure their IP + mask put them in the same network block.
- A **router** connects **different** networks together. Each of its interfaces has its own IP/Mask (one per network it touches), and this is exactly where gateways and routes come into play.

**How to tell them apart on screen:** a router shows up as a single device with **multiple numbered interfaces** (e.g., `interface R11`, `interface R12` both belonging to `router R1`), each with editable IP/Mask fields. A switch either isn't shown as a configurable box at all, or shows up as a simple passthrough between two devices' interface boxes with nothing to edit on it directly.

📖 **For a full, detailed walkthrough of recognizing switches vs. routers in the actual NetPractice diagrams** — including a worked example of reading a multi-hop diagram piece by piece, and the most common mix-up (pointing a gateway at the wrong-side interface) — see **doc 05: Switches vs. Routers**. It's worth reading *before* this document if you're not yet 100% sure you can spot a router on sight, since everything below assumes that skill.

Once you've confirmed you're looking at a router (not a switch), the rest of this document explains what to actually configure on it and around it.

## 🔌 Why a switch isn't enough

A switch only forwards frames between devices **already on the same network** (doc 01). It has zero concept of "other networks" — it doesn't even look at IP addresses, just hardware addresses on its own segment. 🤷

A **router** 🌉 is fundamentally different: it connects two or more *different* networks, and for every packet it decides where to send it next based on the destination IP. Each interface on a router sits on a different network and acts as the **gateway** — the "door out" — for that network.

## 🚪 What "gateway" means in practice

The **default gateway** of a device is the IP it hands traffic to whenever the destination is **not** on its own local network. Two conditions, both required:
- ✅ It must be an IP in the **same network** as the device itself (otherwise the device can't even reach it directly to hand off the packet!)
- ✅ It must be a router interface — the thing that actually knows what to do next

> 📬 **Analogy:** destination on your street → deliver directly, no post office needed. Destination elsewhere → hand it to the post office (gateway), and *your job is done* — routing onward is the post office's problem, not yours.

🧠 **Why this trips people up:** a host with **no gateway configured** can still talk to devices on its *own* network fine — the mistake only shows up in the logs once you try to reach something outside. If a level's log shows "packet accepted" but then nothing forwards, check whether the *sending host* even has a gateway set at all.

## 🗺️ How a router actually decides — the routing table

Every router keeps a routing table: a list of rules, each shaped like this:

```
Destination network / mask   →   Next hop (gateway)
      (where we're going)          (hand it to...)
```

For every incoming packet, the router compares the destination IP against every entry in its table and picks the entry whose network best matches. If the destination is on a **directly connected** network (one of the router's own interfaces), it delivers straight there — no lookup needed, no static route required. Directly-connected networks are "free" knowledge. Anything else requires you to add a rule.

## 🔍 Worked example — building a route by hand

Router **R1**:
- Interface A: `10.0.0.1 /24` → directly connects to network `10.0.0.0/24`
- Interface B: `10.0.1.1 /24` → directly connects to network `10.0.1.0/24`

Router **R2**:
- Interface C: `10.0.1.2 /24` (same network as R1's interface B — they're neighbors on that link) 🔗
- Interface D: `10.0.2.1 /24` → directly connects to network `10.0.2.0/24`

**Goal:** a device on `10.0.0.0/24` 🖥️ needs to reach a device on `10.0.2.0/24` 🖥️.

Trace the path in your head first: `Host A → R1 → R2 → Host B`. R1 isn't directly connected to `10.0.2.0/24` — it only knows A's network and the link to R2. So R1 needs to be told:

```
10.0.2.0/24  →  10.0.1.2
```

Read this as: *"if the destination matches `10.0.2.0/24`, I don't have it directly, so forward the packet to `10.0.1.2` (R2's side of our shared link) and trust R2 to finish the job."* R2, in turn, already has `10.0.2.0/24` directly connected — no extra route needed on that end.

⚠️ **Now the part people forget:** the *reply* from Host B back to Host A has to travel the same path in reverse. R2 needs a route back toward `10.0.0.0/24`, pointing at R1's interface (`10.0.1.1`). Without it, Host A's request arrives fine, but the reply vanishes, and the level looks "half-working" — one goal shows OK, the paired direction doesn't.

## 🔍 Worked example 2 — one router, three hosts on one side (switch + router combined)

This is a setup you'll see very often: several hosts sharing a switch, all needing to reach something across a router. Let's go slower this time, with the full picture.

### The topology, drawn out

```
🖥️ Host A ------\
🖥️ Host B -------+---- [switch] ---- interface R1a ---- 🌉 router R1 ---- interface R1b ---- 🖥️ Host C
🖥️ Host C2 ------/
```

Three hosts on the left all connect to the same switch, which connects to one router interface (`R1a`). The router's *other* interface (`R1b`) leads to a completely separate network where Host C lives.

### The full address table (this is what you'd actually see filled into the interface)

| Device | IP | Mask | Gateway |
|---|---|---|---|
| Host A | `192.168.5.10` | `255.255.255.0` | *(to fill in)* |
| Host B | `192.168.5.11` | `255.255.255.0` | *(to fill in)* |
| Host C2 | `192.168.5.12` | `255.255.255.0` | *(to fill in)* |
| interface R1a | `192.168.5.1` | `255.255.255.0` | — (this IS the gateway, routers don't need one for their own directly-connected side) |
| interface R1b | `172.16.0.1` | `255.255.255.0` | — |
| Host C | `172.16.0.10` | `255.255.255.0` | *(to fill in)* |

**Goal:** every host on the left (A, B, C2) needs to reach Host C on the right.

### Step by step, slowly

**Step 1 — confirm A, B, and C2 are genuinely on the same network as R1a.**
Mask `255.255.255.0` → the network is the first 3 numbers. `192.168.5` matches for A, B, C2, **and** R1a. ✅ This is just doc 01 logic applied to 4 devices instead of 2 — the switch is what physically allows all 4 to share this one network at once.

**Step 2 — every host needs a gateway, and it's the SAME gateway for all three.**
Because A, B, and C2 all live on the exact same network, they all reach the "door out" of that network through the exact same door: `R1a`'s IP, `192.168.5.1`.

```
Host A  gateway → 192.168.5.1
Host B  gateway → 192.168.5.1
Host C2 gateway → 192.168.5.1
```

🧠 **This is the single most common mistake in this exact scenario:** configuring Host A's gateway, testing it, seeing the goal for A pass, and stopping — while B and C2's gateway fields are still empty or wrong. Each host is configured **independently** in NetPractice; fixing one never automatically fixes its neighbors on the same switch, even though conceptually "they're all in the same situation."

**Step 3 — does R1 need any static route for this?**
No. `172.16.0.0/24` (where Host C lives) is directly connected to R1 via `R1b` — that's "free" knowledge for R1, per the routing table rule earlier in this doc. No static route entry needed on R1 at all for this particular goal.

**Step 4 — the return trip.**
Host C also needs a gateway, pointing back at `R1b` (`172.16.0.1`), so that replies addressed to any of A/B/C2 have somewhere to go. Without this, requests from the left side would arrive at Host C fine, but Host C would have no idea how to send anything back — the goal would show as reaching C but never completing round-trip.

### 🔁 Variant — what if Host C2 needs to reach Host C, but A and B don't (yet)?

Same network, same router — but suppose the level's *goals* only mention Host C2 talking to Host C (not A or B). Does that change anything?

**No, and this is worth understanding why:** you still only configure what the goals actually require, but if C2 needs a gateway, it's still `192.168.5.1` — nothing about *which* host is asking changes what the correct gateway IP is. The only difference is you wouldn't necessarily need to fix A and B's gateways *for this specific goal* to pass, since they're not part of what's being tested. In practice, though, it's often worth setting them all correctly anyway, since many levels have multiple goals and hosts sharing a network usually all need the same gateway sooner or later.

### 🕵️ A log specific to this scenario

Suppose Host B's gateway is still unset (empty or wrong) while A and C2 are correctly configured:

```
Goal: host B needs to communicate with host C — Status: KO
--- log ---
on B: packet accepted
on B: destination does not match any interface
pass through routing table
on B: destination does not match any route
```

This is exactly the "missing gateway" signature from Worked example 3 further down — but notice the *other* goals (for A and C2, if they exist) would show `OK` at the same time, because each host's configuration is checked independently. **A green goal for one host on a shared switch never guarantees the others are configured too** — always check every host's field individually, even when they "should" all be the same.

🧠 **Takeaway:** a switch multiplies how many hosts need the *same* gateway — it doesn't change what that gateway should be.

## 🔍 Worked example 3 — diagnosing from the log alone

Imagine you're given this topology and this log after clicking "Check again":

```
Goal: host A needs to communicate with host C — Status: KO
--- log ---
on A: packet accepted
on A: destination does not match any interface
pass through routing table
on A: destination does not match any route
```

Walk through what this tells you, line by line:
- `packet accepted` → the packet left Host A fine, nothing wrong on A's side yet
- `destination does not match any interface` → totally normal, Host C isn't on A's own network — expected
- `pass through routing table` → A is now checking if it knows how to reach C
- `destination does not match any route` → **this is the actual problem** — but this is happening right on Host A, before it even reaches a router. In practice this means **A's gateway field is empty or wrong** — a host with no valid gateway behaves exactly like this in the log.

**Diagnosis:** the log stopping at "does not match any route" right after leaving a host (not a router) almost always means the gateway itself is missing or points somewhere invalid. Fix: set A's gateway to the correct router interface on A's own network.

Compare this to a similar-looking but different log:
```
on A: packet accepted
on A: sent to gateway 192.168.5.1
on R1: packet accepted
on R1: destination does not match any interface
pass through routing table
on R1: destination does not match any route
```

Here A's gateway **is** working (packet reached R1 successfully) — but **R1** is the one missing a route. Same final line, completely different fix, because it's happening on a different device. Always check *which device* the log is complaining about, not just *what* it says.

## 🔍 Worked example 4 — when it's NOT actually a missing route (half-green trap)

```
Router R1: iface A (10.1.1.1/24) -- Host A's network
Router R1: iface B (10.1.2.1/24) -- Host B's network

Configured so far:
  Host A gateway: 10.1.1.1  ✅
  Host B gateway: 10.1.2.1  ✅
```

With only **one** router directly touching both networks, no static route is even needed — both networks are already directly connected to R1. If this still shows KO for a goal needing both A → B and B → A, and both gateways are correctly set, the problem probably isn't a route at all — it's likely something from doc 01: check that both networks are genuinely different blocks, and that neither host is accidentally using a network or broadcast address.

**This example exists to make a point:** not every KO is a missing route. With just one router bridging two directly-connected networks, routes usually aren't the issue — step back to doc 01's addressing checks first before assuming you need to add more configuration.

---

## 📝 Practice — try these yourself, answers below

**1.** Router R1 has `iface X: 172.20.1.1/24` and `iface Y: 172.20.2.1/24`. Host M is on X's network, Host N is on Y's network. What gateway does each host need, and does R1 need any static routes?

**2.** Two routers: R1 (`iface A: 192.168.1.1/24`, `iface B: 10.0.0.1/30`) and R2 (`iface C: 10.0.0.2/30`, `iface D: 192.168.2.1/24`). Host P is on R1's A-side, Host Q is on R2's D-side. Write out every gateway and every route needed, both directions.

**3.** A log shows `on R2: destination does not match any route`, right after a line showing `on R1: sent to gateway 10.0.0.2`. Which device is missing configuration, and what kind?

<details>
<summary>Click to check your answers</summary>

**1.** Host M's gateway = `172.20.1.1` (iface X, its own network). Host N's gateway = `172.20.2.1` (iface Y, its own network). No static route needed on R1 — one router directly connects both networks, so both are "free" knowledge already.

**2.**
- Host P gateway → `192.168.1.1` (iface A)
- Host Q gateway → `192.168.2.1` (iface D)
- R1 route (forward): `192.168.2.0/24 → 10.0.0.2` (hand off to R2)
- R2 route (return): `192.168.1.0/24 → 10.0.0.1` (hand back to R1)

**3.** The packet successfully reached R2 (R1's gateway hop worked fine — that's what "sent to gateway 10.0.0.2" confirms). R2 is the device missing configuration — specifically, **R2 is missing a static route** for whatever network the destination belongs to.

</details>

## 🌍 The default route (`0.0.0.0/0`)

`/0` mask = **zero** network bits required to match → *literally every* destination matches it. It's the catch-all rule 🕸️, exactly like your home router not knowing every website's IP — it just forwards anything unrecognized to your ISP.

```
0.0.0.0/0  →  <ISP / upstream gateway IP>
```

NetPractice calls the router interface that plays this role an **internet interface**. Practically: instead of writing one static route per possible destination network, you write one route that matches *anything not more specifically matched elsewhere*. Routers always prefer a specific route over the default one if both would match — but for NetPractice's simple topologies, you usually only need one or the other per direction, not both stacked.

## 🕵️ Reading the NetPractice logs — line by line

| Log message | What it actually means | What to fix |
|---|---|---|
| `on X: packet accepted` | The packet reached this device / interface fine so far | Nothing yet — keep reading the next line ✅ |
| `destination does not match any interface` | Not a directly-connected network for this router — completely normal | Should fall through to the routing table next |
| `pass through routing table` | The router is now checking its static/default routes | Watch the next line for the verdict |
| `destination does not match any route` | No static route *and* no default route covers this destination | 🚨 **You're missing a route** on this exact router |
| Log stops right after leaving a host, destination never reached | Host has no gateway set, or the wrong gateway | 🚨 **Missing/wrong gateway** on that host |

## 🛠️ How to actually solve this in the interface

1. Read the **Goal** at the top carefully — note both endpoints (host A talking to host C, etc.) and whether it's one-directional or needs a reply path.
2. Trace the topology visually: which routers sit between the two endpoints?
3. For every host involved, check its **gateway field** — it must equal a router interface IP on its *own* segment.
4. For every router along the path, check whether the destination network is directly connected. If not, that router needs a route entry: destination network/mask on the left, the *next router's* interface IP (on the shared link) on the right.
5. Don't stop at "one goal passes" — check if there's a second goal for the reply direction, and repeat step 4 backwards.
6. Re-run `Check again` after each change — the log tells you exactly which hop is still broken.

## 🚨 Common mistakes to watch for

- Setting a host's gateway to an IP that isn't actually on its own network (won't be reachable at all)
- Adding a route on the *wrong* router (the one closer to the destination instead of the one that actually needs it)
- Forgetting the return-path route entirely — one direction works, the paired goal doesn't
- Using a host's own IP as its gateway, or leaving the gateway field as the network/broadcast address by mistake

## ⭐ The rule that matters for NetPractice

> Every device reaching outside its own network needs a 🚪 **gateway** on its own segment, and every router along the way needs a 🛣️ **route** (static or default) covering the destination — in **both directions** if the goal requires two-way communication.
