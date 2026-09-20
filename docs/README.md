# 📚 Index — Reading Order

> Start here. These docs build on each other — reading out of order will leave gaps.

| # | Doc | What it covers | Read this if... |
|---|---|---|---|
| 00 | `00-ip-basics-eli5.md` | The street/house analogy for IP + mask, from absolute zero | You've never touched networking before |
| 01 | `01-ip-and-subnet-mask.md` | Full IP/mask theory: CIDR, the 9 valid mask values, block math, private ranges | You get the analogy and want the real mechanics |
| 04 | `04-block-size-calculation.md` | Just the block-size calculation, drilled in isolation with many examples | You keep getting the block math wrong under pressure |
| 05 | `05-switches-vs-routers.md` | How to tell a switch from a router on screen, and what changes your task | You're not 100% sure what device you're looking at in a diagram |
| 02 | `02-gateway-and-routes.md` | Gateways, static routes, reading NetPractice's logs | You're ready to configure a router, not just addressing |
| 03 | `03-multiple-routers.md` | Chains and branches of routers, return-path routes, non-overlapping segments | Your topology has more than one router |
| 06 | `06-internet-interface-routes.md` | Calculating specific routes for an "Internet" node (not `default`) | A level has an Internet/upstream node needing a real network on the left side |
| 07 | `07-quick-qa-reference.md` | Fast, plain-language Q&A — one-liners for every concept above | You just need a quick reminder, not the full explanation |
| 08 | `08-common-mistakes-review.md` | The recurring concepts behind the errors that come up most, explained again | You want a refresher without re-reading everything |

## Recommended order for a first read-through

```
00 → 01 → 04 → 05 → 02 → 03 → 06 → 09
```
