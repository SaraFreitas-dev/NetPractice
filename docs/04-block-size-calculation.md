# 🧮 04 — Block Size Calculation

> This doc exists for one single skill: given a mask, figure out the valid blocks and which numbers inside them are usable hosts. Nothing else. Take it one section at a time.

## 🎯 The one formula this whole doc is about

```
block size = 256 - mask_value_in_that_octet
```

That's it. Everything else in this document is either *why* this works, or *how to use it* once you have the number.

## 🤔 Why 256?

Because a single octet can hold exactly 256 different values: `0` through `255`. That's the entire "universe" of numbers available in one octet. The mask value tells you how much of that universe is "used up" by the network part — subtracting from `256` tells you how much is left over for addressing within a block.

You confirmed this yourself already: for mask `252`, block size = `256 - 252 = 4`. For mask `248`, block size = `256 - 248 = 8`. That instinct is correct — this doc is just here to slow down and drill it until it's automatic.

## 📍 Step 1 — Find the ONE octet where the math applies

Look at the mask, left to right. You'll always see this pattern:

```
255 . 255 . 255 . 252
 ↑     ↑     ↑     ↑
copy  copy  copy  ← THIS is the only one you calculate
```

- Every `255` octet → just copy the given number exactly, no math
- Every `0` octet → completely free, no math
- The **one** octet that's neither `255` nor `0` → this is where block size applies

If a mask is something like `255.255.0.0`, there's no "mixed" octet at all — everything is a clean `255` or `0`, and you skip straight to "copy" / "free," no block-size math needed anywhere.

## 📍 Step 2 — Calculate the block size

```
block size = 256 - mask_value_in_that_octet
```

Practice this a few times, out loud if it helps:

| Mask value | Calculation | Block size |
|---|---|---|
| `128` | `256 - 128` | `128` |
| `192` | `256 - 192` | `64` |
| `224` | `256 - 224` | `32` |
| `240` | `256 - 240` | `16` |
| `248` | `256 - 248` | `8` |
| `252` | `256 - 252` | `4` |
| `254` | `256 - 254` | `2` |

## 📍 Step 3 — List the blocks

Once you have the block size, the blocks are just that number's multiplication table, starting at `0`:

**Example: block size = 8**
```
Multiples of 8:  0, 8, 16, 24, 32, 40...
```

Each multiple is where a new block *starts*. Each block *ends* exactly one number before the next multiple begins:

```
Block starting at 0:  ends at 8 - 1 = 7    →  block is 0–7
Block starting at 8:  ends at 16 - 1 = 15  →  block is 8–15
Block starting at 16: ends at 24 - 1 = 23  →  block is 16–23
```

Write this out by hand a few times with different block sizes until the pattern feels obvious — it's always "multiple, and one before the next multiple."

## 📍 Step 4 — Figure out which block a specific number belongs to

If you're given a number (say, an IP that's already fixed in the exercise), find which block it falls into:

```
Given number: 14
Block size: 8
Multiples of 8: 0, 8, 16, 24...
14 is between 8 and 16  →  it's in the block 8–15
```

**Quick shortcut without listing every multiple:** divide the number by the block size, round down, then multiply back:

```
14 ÷ 8 = 1.75  →  round down to 1  →  1 × 8 = 8
So the block starts at 8, and ends at 8 + 8 - 1 = 15
```

## 📍 Step 5 — Inside the block, identify network / hosts / broadcast

This rule never changes, regardless of block size:

```
First number in the block   → network address   (reserved, not a host)
Last number in the block    → broadcast address (reserved, not a host)
Everything in between       → valid hosts
```

**Example: block 8–15**
```
8  → network      ❌
9  → host         ✅
10 → host         ✅
11 → host         ✅
12 → host         ✅
13 → host         ✅
14 → host         ✅
15 → broadcast    ❌
```

## 🔁 Two scenarios — does the block choose itself, or do you choose it?

**You DON'T choose the block when:** you're already given an IP (locked field in NetPractice). The block is whatever block that given number happens to fall in — you calculate it, you don't pick it.

**You DO choose the block when:** there's no IP given yet — you're assigning addresses to a brand-new segment from scratch. Any block works, but every device on that segment must end up in the *same* block you picked.

---

## 🏋️ Worked examples, one at a time, increasing difficulty

### Example 1 — mask `255.255.255.128`, given number `.5`

1. Mixed octet: last one, value `128`
2. Block size: `256 - 128 = 128`
3. Multiples of 128: `0, 128, 256...` → only two blocks fit in one octet: `0–127` and `128–255`
4. `5` falls in `0–127`
5. Network = `0`, broadcast = `127`, hosts = `1`–`126`

### Example 2 — mask `255.255.255.192`, given number `.140`

1. Mixed octet: last one, value `192`
2. Block size: `256 - 192 = 64`
3. Multiples of 64: `0, 64, 128, 192`
4. `140` falls between `128` and `192` → block is `128–191`
5. Network = `128`, broadcast = `191`, hosts = `129`–`190`

### Example 3 — mask `255.255.255.252`, no IP given (from scratch), you pick Block 2

1. Mixed octet: last one, value `252`
2. Block size: `256 - 252 = 4`
3. Multiples of 4: `0, 4, 8, 12...` → Block 1 = `0–3`, Block 2 = `4–7`
4. You choose Block 2: `4–7`
5. Network = `4`, broadcast = `7`, hosts = `5` and `6` only (only 2 hosts — this is why `/30` is the standard for a router-to-router link)

### Example 4 — the exact case you tried: mask `255.255.255.252`, wanting hosts `.2` and `.4`

1. Block size = `4`, blocks are `0–3`, `4–7`, `8–11`...
2. `.2` falls in block `0–3`
3. `.4` falls in block `4–7` — **a different block!**
4. This is invalid: two hosts meant to be neighbors must be in the *same* block
5. Fix: either use `.1` and `.2` (both in block `0–3`, since `0`=network and `3`=broadcast leave only `1` and `2` as hosts), or `.5` and `.6` (both in block `4–7`)

---

## 📝 Practice — try these yourself, answers below

1. Mask `255.255.255.240`, given number `.20` — which block, and what's the valid host range?
2. Mask `255.255.255.224`, given number `.100` — which block, and what's the valid host range?
3. Mask `255.255.255.248`, from scratch, you pick Block 3 — what's the network, broadcast, and host range?

<details>
<summary>Click to check your answers</summary>

**1.** Block size = `256-240=16`. Multiples: `0,16,32...`. `20` falls in `16–31`. Network=`16`, broadcast=`31`, hosts=`17`–`30`.

**2.** Block size = `256-224=32`. Multiples: `0,32,64,96,128...`. `100` falls in `96–127`. Network=`96`, broadcast=`127`, hosts=`97`–`126`.

**3.** Block size = `256-248=8`. Multiples: `0,8,16,24...`. Block 1=`0-7`, Block 2=`8-15`, Block 3=`16-23`. Network=`16`, broadcast=`23`, hosts=`17`–`22`.

</details>

## ✅ One-paragraph summary

> Find the one mask octet that isn't `255` or `0` — that's where the math happens. Block size = `256 - that value`. Blocks are that number's multiples (`0, size, 2×size...`), each block running from one multiple up to (next multiple − 1). If you're given a number, find which block it falls in; if you're starting from scratch, pick any block, but keep every device on that segment inside the same one. Inside any block, the first number is always network (reserved) and the last is always broadcast (reserved) — everything between is a valid host.
