# 🏠 00 — IP & Mask Explained From Zero (the Street & House analogy)

> This doc exists for someone who has genuinely never seen this before. No jargon, no binary knowledge assumed — just the logic, step by step, with lots of examples.

## 🤔 First: what even is an IP address?

Every device on a network (a computer, a router, a phone) needs an "address" so other devices know how to reach it — just like your house needs a postal address so people can send you mail.

That address is the **IP address**. It looks like this:

```
104.99.23.12
```

Four numbers, separated by dots. Each number can be anything from `0` to `255` (never higher — more on why later).

On its own, this number tells you **nothing** about which other devices it can talk to. That's the whole reason the **subnet mask** exists — it's the missing piece of context.

## 🛣️🚪 The core idea: Street name + House number

Here's the one idea that unlocks everything else in this document:

> An IP address is made of two parts glued together: a **"street name"** part and a **"house number"** part. Two devices can only talk to each other **directly** (without a router helping) if they live on the **same street**.

The **mask** is what tells you where the "street name" ends and the "house number" begins.

| Part | Analogy | What it actually does |
|---|---|---|
| 🛣️ Network portion | Street name | Identifies which network the device belongs to |
| 🚪 Host portion | House number | Identifies that one specific device on the network |

## 🔢 How to actually read this: number by number, same position

Both the IP and the mask are written as 4 numbers. Line them up, one on top of the other:

```
IP:   104 . 99 . 23 . 12
Mask: 255 . 255 . 255 . 0
```

Now look at each **position** (1st number, 2nd number, 3rd number, 4th number) and compare the IP to the mask **at that same position**:

| Position | IP value | Mask value | What the mask is telling you |
|---|---|---|---|
| 1st | `104` | `255` | "this number is part of the street" |
| 2nd | `99` | `255` | "this number is part of the street" |
| 3rd | `23` | `255` | "this number is part of the street" |
| 4th | `12` | `0` | "this number is part of the house" |

### The rule in one sentence

> Wherever the mask shows **255**, that same position in the IP is the **street**. Wherever the mask shows **0**, that position is the **house**.

So for `104.99.23.12` with mask `255.255.255.0`:
- **Street** = `104.99.23` (the first 3 numbers — they must match exactly for two devices to be neighbors)
- **House** = `12` (the last number — this is what makes this specific device unique on its street)

## 🧠 But why does 255 mean "street" and 0 mean "house"?

You don't need to understand binary to get this intuitively — just remember:

- `255` is the **maximum** possible value at that position → the mask is saying "this has to match 100%, no wiggle room, no exceptions"
- `0` is the **minimum** possible value → the mask is saying "this position doesn't matter for deciding the street, it can be literally anything"

If you ever do want the deeper mechanical reason (totally optional to understand at this stage): both the IP and mask are actually stored as 32 individual `1`s and `0`s (bits). Wherever the mask has a `1`-bit, that bit of the IP counts as "street." Wherever it has a `0`-bit, that bit is "house." `255` happens to be 8 `1`-bits in a row (`11111111`), and `0` is 8 `0`-bits in a row (`00000000`) — which is exactly why they mean "full street" and "full house" respectively.

## 📖 Worked example #1 — same mask, walk through it slowly

```
Device A:  IP 104.99.23.5    Mask 255.255.255.0
Device B:  IP 104.99.23.200  Mask 255.255.255.0
```

Step 1 — read the mask position by position: `255, 255, 255, 0` → first 3 positions are street, last is house.

Step 2 — extract the street for each device:
- Device A's street: `104.99.23`
- Device B's street: `104.99.23`

Step 3 — compare: **same street!** ✅ These two devices CAN talk to each other directly.

Step 4 — their houses are different (`5` vs `200`), which is exactly what makes them two separate, distinguishable devices on the same street.

## 📖 Worked example #2 — different street, same-looking IPs

```
Device C:  IP 104.99.23.5    Mask 255.255.255.0
Device D:  IP 104.99.24.5    Mask 255.255.255.0
```

Street for C: `104.99.23`
Street for D: `104.99.24`

These look almost identical at a glance (only the 3rd number differs, by 1!) — but they are **different streets**. ❌ These two devices CANNOT talk to each other directly. This is the single most common beginner mistake: assuming IPs that "look similar" must be on the same network. What matters is not visual similarity — it's whether the street portion (as defined by the mask) matches exactly.

## 📖 Worked example #3 — a different mask changes everything

```
Device E:  IP 211.191.56.75   Mask 255.255.0.0
Device F:  IP 211.191.200.10  Mask 255.255.0.0
```

Step 1 — read the mask: `255, 255, 0, 0` → only the **first 2** positions are street this time. The last 2 positions are both house.

Step 2 — street for E: `211.191`
Step 2 — street for F: `211.191`

Step 3 — same street! ✅ These two CAN talk directly, **even though their 3rd numbers are completely different** (`56` vs `200`) — because with this mask, the 3rd number is part of the house, not the street.

🧠 **Takeaway:** the mask completely changes what counts as "the street." The exact same pair of IPs could be on the same network or different networks depending only on which mask you're using. Always check the mask before assuming anything about who can talk to whom.

## 🚪 How do you pick the house number?

Once you know the street (fixed by whoever/whatever you need to match), you need to pick a house number for your own device. Three simple rules:

1. ❌ **It can't be `0`** — a house number of all-zeros is reserved; it actually refers to the *entire street* (the "network address"), not a single house.
2. ❌ **It can't be `255`** (or, more precisely, the maximum possible value for that many house-digits) — this is reserved for "broadcast," a special message sent to *every* house on the street at once, not a real device.
3. ❌ **It can't already be used by another device on the same street** — every house needs a unique number.

Outside those 3 rules? **Pick anything.** There's no "correct" house number — it's like choosing any free house number on a street. `1`, `2`, `3`, `50`, `199` — all equally valid, as long as they're not already taken and not one of the two reserved values.

### Quick example

Street: `104.99.23` (mask `255.255.255.0`, so the house number is just the last single position)
- ❌ `104.99.23.0` — reserved (means "the whole street")
- ❌ `104.99.23.255` — reserved (means "broadcast to everyone")
- ✅ `104.99.23.1`, `104.99.23.2`, `104.99.23.87`, `104.99.23.254` — all valid choices, pick any free one

## 🧩 A full walkthrough — connecting two devices for real

Let's use an actual NetPractice-style situation. Your PC needs to talk to your little brother's computer.

**What's already given (his computer, host B):**
```
IP:   104.99.23.12
Mask: 255.255.255.0
```

Reading it: street = `104.99.23`, house = `12`.

**What you need to fill in (your PC, host A):**
```
Mask: 255.255.255.0   (already correct — matches his)
IP:   ???
```

To be on the same street as him, your IP needs to start with `104.99.23`. For your house number, you just need something that isn't `0`, isn't `255`, and isn't `12` (already taken by him).

✅ A valid answer: `104.99.23.1`

Full picture once done:
```
Host A (you):        104.99.23.1
Host B (your brother): 104.99.23.12
```
Same street (`104.99.23`) → they can talk directly. Different houses (`1` vs `12`) → they're recognized as two separate devices.

## 🗂️ Naming things properly (useful for your README later)

What we've been doing this whole time has an actual name:

| Term | What it means |
|---|---|
| **Topology** | The diagram itself — which devices are connected to which (the drawing you see on screen) |
| **IP addressing / network addressing** | Choosing the right IP + mask for each device |
| **Subnetting** | The technique of calculating the correct ranges/masks for each "street" (network) |

So this document is really about learning **IP addressing** — figuring out, for a topology that's already drawn for you, which numbers each device needs so they can actually talk to the right neighbors.

## ✅ One-paragraph summary

> An IP address has two glued-together parts: a street (network) and a house (host). The mask tells you, position by position, which part is which — `255` means "this is street, must match exactly," `0` means "this is house, can be almost anything." Two devices can only talk directly if their street portions are identical. To pick a house number, choose anything that isn't the reserved "all zeros" or "all max value" address, and isn't already used by another device on that same street.
