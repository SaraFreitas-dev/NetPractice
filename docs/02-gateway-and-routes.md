# 🚦 02 — Gateway & Static Routes

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
