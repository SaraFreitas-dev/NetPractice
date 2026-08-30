# 🌐 01 — IP Addressing & Subnet Mask

## 📦 What an IPv4 address actually is

An IPv4 address is 32 bits, written as 4 decimal octets separated by dots:

```
192.168.1.10
```

Each octet is 8 bits (`0`–`255`). On its own, an IP is just a number — it tells you nothing about who can talk to whom. That's entirely the **subnet mask**'s job. 🔑

## 🏠 Network portion vs. host portion

The mask splits the 32 bits into two contiguous chunks:

| Part | Analogy | Role |
|---|---|---|
| 🛣️ **Network portion** | Street name | Which network the device belongs to |
| 🚪 **Host portion** | House number | The specific device on that network |

> 🧠 **Why this matters:** two devices can only talk *directly* (no router) if their street name is identical. It's not about the IPs "looking similar" — it's about what the mask says is the network portion. `10.0.0.5` and `10.0.0.9` can be on completely different networks if the mask says so (see the `/30` example below).

Mechanically, the mask works via **bitwise AND**: `IP AND mask = network address`. Everywhere the mask has a `1`, that bit of the IP is kept (network); everywhere it has a `0`, that bit is zeroed out (host portion, ignored for network identity).

## 🔢 Reading a mask in binary

```
255.255.255.0   = 11111111.11111111.11111111.00000000   → /24  (8 host bits  → 256 addrs)
255.255.255.128 = 11111111.11111111.11111111.10000000   → /25  (7 host bits  → 128 addrs)
255.255.255.192 = 11111111.11111111.11111111.11000000   → /26  (6 host bits  → 64 addrs)
255.255.255.224 = 11111111.11111111.11111111.11100000   → /27  (5 host bits  → 32 addrs)
255.255.255.240 = 11111111.11111111.11111111.11110000   → /28  (4 host bits  → 16 addrs)
255.255.255.248 = 11111111.11111111.11111111.11111000   → /29  (3 host bits  → 8 addrs)
255.255.255.252 = 11111111.11111111.11111111.11111100   → /30  (2 host bits  → 4 addrs)
```

📐 **The core formula:** `/n` → `32 - n` host bits → `2^(32-n)` total addresses in that block. Memorize the last-octet values (`0,128,192,224,240,248,252,254,255`) — you'll use them constantly, and in the eval you have no calculator beyond `bc`.

### 🎯 The "only 9 valid values" rule

A subnet mask, read left to right in binary, is always a run of `1`s followed by a run of `0`s — **never** mixed (`11110101` is not a valid mask octet). This means **only 9 values are ever possible** in any mask octet:

```
00000000 → 0
10000000 → 128
11000000 → 192
11100000 → 224
11110000 → 240
11111000 → 248
11111100 → 252
11111110 → 254
11111111 → 255
```

⚠️ **The moment one octet isn't `255`, every octet after it must be `0`.** So `255.255.240.0` is a valid mask, but `255.255.240.128` is never valid — once the run of `1`s stops, it can't restart later. If you ever see a mask field that doesn't fit this pattern, something's wrong before you even get to IPs.

### 🧠 Deriving the 9 values under exam pressure (no memorizing 9 random numbers)

You don't need to memorize `0, 128, 192, 224, 240, 248, 252, 254, 255` as a random list — each value is the previous one **plus half of what's left to 256**:

```
0
128  (0 + 128)
192  (128 + 64)
224  (192 + 32)
240  (224 + 16)
248  (240 + 8)
252  (248 + 4)
254  (252 + 2)
255  (254 + 1)
```

The additions are always `128, 64, 32, 16, 8, 4, 2, 1` — powers of 2, halving each time. If you know that sequence (it never changes), you can rebuild the whole table in seconds instead of recalling 9 loose numbers.

### 🧮 If you blank on it: the `bc` formula (allowed in the eval)

The PDF explicitly allows a simple calculator like `bc` during evaluation. The formula:

```
mask_octet = 256 - 2^(8 - network_bits_in_that_octet)
```

**Worked example — deriving `/30`:**
1. `/30` = 30 network bits total
2. The first 3 octets are always fully `255` (that's `8+8+8 = 24` bits used up)
3. That leaves `30 - 24 = 6` network bits in the last octet
4. `256 - 2^(8-6) = 256 - 4 = 252` → so `/30` = `255.255.255.252` ✅

```bash
$ bc
256 - 2^(8-6)
252
```

**Shortcut for any CIDR:** count how many full `255` octets fit first (each is worth 8 bits), then apply the formula (or the doubling table) to whatever bits remain in the next octet. It's the same calculation every time — practice it a few times and it becomes automatic.

### 📋 Full CIDR reference table

| CIDR | Subnet Mask | # Addresses | # Usable Hosts |
|---|---|---|---|
| /32 | 255.255.255.255 | 1 | 1 |
| /31 | 255.255.255.254 | 2 | 2 |
| /30 | 255.255.255.252 | 4 | 2 |
| /29 | 255.255.255.248 | 8 | 6 |
| /28 | 255.255.255.240 | 16 | 14 |
| /27 | 255.255.255.224 | 32 | 30 |
| /26 | 255.255.255.192 | 64 | 62 |
| /25 | 255.255.255.128 | 128 | 126 |
| /24 | 255.255.255.0 | 256 | 254 |
| /23 | 255.255.254.0 | 512 | 510 |
| /22 | 255.255.252.0 | 1,024 | 1,022 |
| /21 | 255.255.248.0 | 2,048 | 2,046 |
| /20 | 255.255.240.0 | 4,096 | 4,094 |
| /19 | 255.255.224.0 | 8,192 | 8,190 |
| /18 | 255.255.192.0 | 16,384 | 16,382 |
| /17 | 255.255.128.0 | 32,768 | 32,766 |
| /16 | 255.255.0.0 | 65,536 | 65,534 |

💡 For NetPractice specifically, you'll almost never need anything smaller than `/16` — most levels live comfortably in the `/24`–`/30` range. The full table down to `/0` exists in theory, but memorize `/24` through `/30` cold; everything bigger is rare in these exercises.

## 🧮 Calculating a network's range — full method

Given an IP and a mask, three things exist in every block:

1. 🏳️ **Network address** — all host bits = `0` → identifies the block itself, never assigned to a device
2. 📢 **Broadcast address** — all host bits = `1` → "send to everyone on this network," never assigned to a single device
3. ✅ **Usable range** — everything strictly between network and broadcast

**Step-by-step method (works for any mask):**
1. Find the "block size" in the octet where the mask stops being `255`: `block size = 256 - mask_value_in_that_octet`.
2. Find which multiple of the block size the given IP falls into — that multiple (and the one right after it minus 1) bound your block.
3. Network = start of block, broadcast = end of block, usable = everything in between.

### 🔍 Worked example: `104.198.212.129 /25`

- `/25` touches the last octet: mask there = `128` → block size = `256 - 128 = 128`
- Blocks in the last octet: `0–127` and `128–255`. `129` falls in `128–255`.

| | |
|---|---|
| 🏳️ Network | `104.198.212.128` |
| ✅ Usable range | `104.198.212.129` – `104.198.212.254` (126 usable hosts) |
| 📢 Broadcast | `104.198.212.255` |

### 🔍 Worked example: `92.168.1.1 /30`

- Mask in last octet = `252` → block size = `256 - 252 = 4`
- Blocks: `0–3, 4–7, 8–11, ...`. `1` falls in `0–3`.

| | |
|---|---|
| 🏳️ Network | `92.168.1.0` |
| ✅ Usable range | `92.168.1.1` – `92.168.1.2` (only 2 hosts — perfect for a router-to-router link! 🔗) |
| 📢 Broadcast | `92.168.1.3` |

### 🔍 Worked example: `172.16.14.200 /22`

- `/22` means the mask spans into the *third* octet: `255.255.252.0` → block size in 3rd octet = `256 - 252 = 4`
- Blocks in 3rd octet: `0–3, 4–7, ..., 12–15, ...`. The IP's 3rd octet is `14`, which falls in `12–15`.
- So the block spans the *whole 4th octet* for 3rd-octet values `12` through `15`.

| | |
|---|---|
| 🏳️ Network | `172.16.12.0` |
| ✅ Usable range | `172.16.12.1` – `172.16.15.254` (1022 usable hosts) |
| 📢 Broadcast | `172.16.15.255` |

⚠️ This is the case that trips people up: when the mask doesn't stop in the last octet, the "block size" math applies to an *earlier* octet, and everything after it is along for the ride.

## 🚫 Private IP ranges (you'll see these constantly in NetPractice)

| Range | Typical use |
|---|---|
| `10.0.0.0 – 10.255.255.255` | Large private networks |
| `172.16.0.0 – 172.31.255.255` | Medium private networks |
| `192.168.0.0 – 192.168.255.255` | Small/home private networks |

These aren't routable on the public internet — but NetPractice's simulated topologies use them freely, exactly like a real company's internal network would.

## ⭐ The rule that matters for NetPractice

> Any two interfaces on the same switch (no router between them) must:
> 1. Use the **same subnet mask** 🎭
> 2. Have IPs that fall in the **same network block** 📍 (same result after applying the mask)

## 🛠️ How to actually solve this in the interface

1. Look at which field is **already filled and locked (shaded)** vs. **editable (unshaded)** — the locked one usually tells you the mask or one IP you must match.
2. Apply the block-size method to the locked IP+mask to get the valid range.
3. Fill the unshaded IP with any address in that range that isn't the network or broadcast address, and isn't already taken by another device on the same segment.
4. Match the mask on the unshaded field to the locked one — **the mask must be identical for every device on the same switch.**
5. Hit `Check again` — if it's still red, re-read the log: it usually says exactly whether it's a mask mismatch or an out-of-range IP.

## 🚨 Common mistakes to watch for

- Assigning the **network address** or **broadcast address** to a host by accident (classic off-by-one on the block boundaries)
- Copying a mask without checking it actually matches the *other* device's mask on the same segment
- Forgetting that two devices can have IPs that "look close" (`10.0.0.5` vs `10.0.0.9`) but be on *different* networks once you actually apply a small mask like `/30`

## ⚡ Quick mental shortcut

Block size = `256 - mask_octet_value`. Whichever multiple of that block size the IP sits just above = your network address. Add `block_size - 1` = your broadcast. Everything between = usable. Practice this until it's instant — it's the single most-repeated calculation in every level. 💡
