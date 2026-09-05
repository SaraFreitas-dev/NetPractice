# 🔗 03 — Multiple Routers & Topology Rules

## 🧩 Why this level of complexity exists

Docs 01–02 covered a single link (same network) and a single router bridging two networks. Real topologies — and the later NetPractice levels — chain **several routers together** 🌉🌉🌉. Each router still only knows what doc 02 said: its own directly-connected networks, plus whatever routes you explicitly add. Getting traffic across the *whole* chain means **every router on the path** needs the right instructions — not just the ones nearest the two endpoints.

## 1️⃣ Rule — Every segment needs its own, non-overlapping network

Every link (router-to-router) and every LAN (router-to-switch-to-hosts) needs a **distinct** network range (own block, per doc 01's math). Two different segments can never legitimately share the same range unless they're actually meant to be the same segment.

🚨 **Why this actually breaks things (not just "looks messy"):** a router decides where to forward a packet by matching the destination against its known networks/routes. If two *different* segments both claim `192.168.1.0/24`, the router can't tell which physical interface a packet for that range should go out — traffic can get delivered to the wrong segment, or the router can pick the first match and silently misroute. This is one of the sneakiest bugs because the level can look "almost right" — masks all valid, IPs all in-range — and still fail.

**Checklist before finalizing any new segment:**
- Does its range overlap any other segment already used anywhere in the topology (not just the neighboring one)?
- Is the mask sized right for the segment? A router↔router link only ever needs 2 usable hosts → `/30` is the natural choice 🔌; a LAN with several hosts needs a bigger block 🏘️.

## 2️⃣ Rule — Each router only knows what it's told, one hop at a time

A router's table starts with its **directly connected networks** for free (one per interface). Anything beyond that needs an explicit static route, or a default route as a catch-all (doc 02). Crucially, a route only needs to point at the **next router in the chain**, not the final destination — the next router repeats the same logic for its own next hop.

```
🖥️ Host A -- 🌉 R1 -- 🌉 R2 -- 🌉 R3 -- 🖥️ Host B
```

For A → B to work end-to-end, every hop needs to know how to move the packet *one step closer*:
- **R1** needs a route toward whatever comes after it (R2's side), or a default route
- **R2** needs a route toward R3's side
- **R3** needs a route toward B's network — likely already directly connected, since B sits on R3's own segment ✅

🔁 **The return path is a completely separate problem.** R3, R2, and R1 each need a route *back* toward A's network too. It's easy to configure the forward path, watch one goal turn green, and stop — but if the goal requires two-way communication, the reply needs its own chain of routes, hop by hop, in the opposite direction. In the logs this shows up as: the forward goal passes, but a paired goal (or the same goal checked from the other side) reports the packet never arriving back.

## 3️⃣ Rule — Think "next hop," not "final destination"

A static route's right-hand side is **never** the destination host itself — it's the next router interface **on the path toward it**, always on a network the current router can reach directly (its own connected link to that neighbor). This is exactly what makes long chains tractable: each router only solves "who do I hand this to next," and responsibility passes one hop at a time until it reaches a router that has the destination directly connected.

## 🔍 Worked example — 3-router chain, both directions

```
🖥️ Host A (192.168.10.10/24)
        |
   R1 iface1 (192.168.10.1/24)
   R1 iface2 (10.0.0.1/30)  ---- 10.0.0.2/30  R2 iface1
                                  R2 iface2 (10.0.1.1/30) ---- 10.0.1.2/30  R3 iface1
                                                                R3 iface2 (192.168.20.1/24)
                                                                        |
                                                                🖥️ Host B (192.168.20.10/24)
```

**Forward direction (A → B):**
- Host A's gateway → `192.168.10.1` (R1 iface1, same segment ✅)
- R1 needs a route: `192.168.20.0/24 → 10.0.0.2` (hand off to R2)
- R2 needs a route: `192.168.20.0/24 → 10.0.1.2` (hand off to R3)
- R3: `192.168.20.0/24` is directly connected — nothing to add

**Return direction (B → A):**
- Host B's gateway → `192.168.20.1` (R3 iface2, same segment ✅)
- R3 needs a route: `192.168.10.0/24 → 10.0.1.1` (hand back to R2)
- R2 needs a route: `192.168.10.0/24 → 10.0.0.1` (hand back to R1)
- R1: `192.168.10.0/24` is directly connected — nothing to add

Notice the pattern: **6 route entries total** across 3 routers for full two-way communication (2 per router, one each direction) — plus the two host gateways. Miss any single one and exactly one direction of one segment stops working, which is usually visible as "one goal OK, the paired one KO."

## 🔍 Worked example 2 — a BRANCHING topology (not just a straight line)

Chains aren't the only shape you'll see. Sometimes one router connects to *two* others, forming a branch:

```
                          🖥️ Host A (192.168.1.10/24)
                                   |
                          R1 iface1 (192.168.1.1/24)
                          R1 iface2 (10.0.0.1/30) ---- 10.0.0.2/30 R2 iface1 -- R2 iface2 (192.168.2.1/24) -- 🖥️ Host B (192.168.2.10/24)
                          R1 iface3 (10.0.1.1/30) ---- 10.0.1.2/30 R3 iface1 -- R3 iface2 (192.168.3.1/24) -- 🖥️ Host C (192.168.3.10/24)
```

R1 has **three** interfaces this time — one toward Host A, one toward R2 (which leads to Host B), one toward R3 (which leads to Host C). This is the same logic as the linear chain, just applied with R1 as a hub instead of a link in a straight line.

**Goal: Host A needs to reach both Host B and Host C.**

- Host A's gateway → `192.168.1.1` (R1 iface1) — same as always, only one gateway needed even though there are two destinations
- R1 needs **two separate routes**, one per branch:
  - `192.168.2.0/24 → 10.0.0.2` (toward R2, for Host B)
  - `192.168.3.0/24 → 10.0.1.2` (toward R3, for Host C)
- R2 needs nothing extra for `192.168.2.0/24` (directly connected) — but for the **return** trip to Host A, R2 needs: `192.168.1.0/24 → 10.0.0.1` (back to R1)
- R3, similarly, needs for its return trip: `192.168.1.0/24 → 10.0.1.1` (back to R1)

🧠 **Takeaway:** a router with 3+ interfaces doesn't change any single rule — it just means you repeat "add a route for every network I'm not directly connected to" **once per branch**, not just once total. R1 ends up with 2 routes instead of 1, simply because it has 2 "elsewhere" networks to reach instead of 1.

⚠️ **A trap specific to branching topologies:** R2 and R3 only know about their *own* branch. If Host B ever needed to reach Host C (not just Host A), R1 would need to relay between R2 and R3 — but R2 doesn't automatically know R3 exists, and vice versa. Every router still only knows what doc 02 said: its direct connections plus whatever you explicitly add. A branch doesn't create automatic awareness between the branches themselves.

## 🔍 Worked example 3 — spotting a broken route in a chain from the log

Given the same 3-router chain as Worked example 1 (Host A ↔ Host B through R1-R2-R3), suppose you've configured everything *except* you forgot R2's return-direction route. Here's what checking the reverse goal (B → A) would show:

```
Goal 2: host B needs to communicate with host A — Status: KO
--- log ---
on B: packet accepted
on B: sent to gateway 192.168.20.1
on R3: packet accepted
on R3: sent to gateway 10.0.1.1
on R2: packet accepted
on R2: destination does not match any interface
pass through routing table
on R2: destination does not match any route
```

Reading this line by line: B successfully reached R3, R3 successfully forwarded to R2 (both of those hops are configured correctly) — but **R2** doesn't know how to get to `192.168.10.0/24` from here. The log names R2 explicitly right where it breaks down. This is exactly the "6 route entries, miss one and exactly one direction breaks" scenario from Worked example 1 — the log tells you precisely which of the 6 is the missing one, so you never have to guess-and-check all of them.

---

## 📝 Practice — try these yourself, answers below

**1.** A linear chain: Host A — R1 — R2 — Host B (2 routers only, not 3). R1's networks: `iface1 172.16.1.1/24` (toward A), `iface2 10.5.5.1/30` (toward R2). R2's networks: `iface1 10.5.5.2/30` (toward R1), `iface2 172.16.2.1/24` (toward B). Write every gateway and every route needed, both directions.

**2.** In the branching topology from Worked example 2, if Host B tried to reach Host C directly (not through Host A), would it work with the routes already configured? Why or why not?

**3.** A log for a 4-router chain shows the packet successfully passing through the first 2 routers, then stopping with `destination does not match any route` on the 3rd router. How many total route entries would you expect to need for full 2-way communication across all 4 routers, and which single one is confirmed missing by this log?

<details>
<summary>Click to check your answers</summary>

**1.**
- Host A gateway → `172.16.1.1`
- Host B gateway → `172.16.2.1`
- R1 route (forward): `172.16.2.0/24 → 10.5.5.2`
- R2 route (return): `172.16.1.0/24 → 10.5.5.1`

**2.** No, it would fail. R2 (Host B's router) has no route toward `192.168.3.0/24` (Host C's network) — it only knows its own directly-connected network and, at most, a route back toward Host A via R1. R3 is a completely separate branch that R2 has never been told about. To make B ↔ C work, you'd need to add routes on R2 (toward R3, via R1) and on R3 (toward R2, via R1), plus R1 would need to relay between both branches — R1 already can, since it directly connects to both, but R2 and R3 individually cannot skip over R1.

**3.** For a 4-router chain (R1-R2-R3-R4), full 2-way communication needs 2 routes per router × 4 routers = **8 total route entries** (each router needs one route toward "further along" and one toward "back the way it came," except the routers at each end only need one direction each toward their own directly-connected host network — so in practice it's slightly fewer, but conceptually budget for up to 2 per router). The log confirms specifically that **the 3rd router is missing its forward-direction route** — the first 2 routers' forwarding is confirmed working since the packet reached the 3rd router successfully.

</details>

## 🧠 Mental model for solving a multi-router level

1. 🗺️ **Map the topology fully first** — every segment, every router interface, before touching any field. Sketch it on paper if the diagram is dense.
2. 🔍 **Validate addressing on each segment** (doc 01) — non-overlapping, right-sized mask, valid host IPs.
3. 🚪 **Set every host's gateway** to its own segment's router interface.
4. 🛣️ **For every router, for every network it's NOT directly connected to**, add a route: destination network → next hop toward it.
5. 🔁 **Explicitly re-check the reverse direction** — don't assume it's automatically covered by the forward routes; it needs its own set.
6. 📋 **Use the log after every `Check again`** — it names the exact router/hop where the packet got stuck, so you don't have to guess which of several routers is missing an entry.

## 🚨 Common mistakes to watch for

- Fixing the forward path, seeing one goal turn green, and declaring the level done while a second (return-path) goal is still red
- Pointing a route at the *final destination's* IP instead of the *next router's* interface on the shared link
- Reusing a `/30` (or any) range on two different router-to-router links by accident
- Adding a route on a router that's actually already directly connected to that destination (redundant, and sometimes flagged as a config the checker doesn't expect)

## ⭐ The rule that matters for NetPractice

> A working multi-router topology needs: 🚫 non-overlapping addressing on every segment, 🚪 a gateway on every host, and 🛣️ a route on every router for every network it isn't directly connected to — repeated in **both directions** for every pair of endpoints that must communicate. 🔁
