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
