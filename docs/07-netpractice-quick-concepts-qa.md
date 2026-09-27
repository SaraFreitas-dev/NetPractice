# 🌐 NetPractice — Quick Concepts Q&A

A short revision guide with the main networking concepts used in **NetPractice**.

The goal is simple: read the question, answer it quickly, and check the definition.

---

## 🧠 Basic Networking Concepts

### What is an IP address?
An address used to identify a device or interface on a network.

### What is a network?
A group of devices that can communicate with each other.

### What is a host?
A device connected to a network, such as a computer or server.

### What is an interface?
A network connection point on a host or router.

### What is a subnet?
A smaller network created by dividing a larger network.

### What is a subnet mask?
It defines which part of an IP address represents the **network** and which part represents the **host**.

### Why do devices need a subnet mask?
To know whether another IP is inside the same network or must be reached through a router.

### What is a network address?
The first address of a subnet. It identifies the subnet itself.

### Can the network address be assigned to a host?
No.

### What is a broadcast address?
The last address of a subnet. It is used to reach all devices in that subnet.

### Can the broadcast address be assigned to a host?
No.

### Which addresses in a subnet cannot be assigned to hosts?
A: The first address is the **network address**, and the last is the **broadcast address**.

Example: `192.168.1.0 → 192.168.1.127`  
- `.0` = Network  
- `.1 → .126` = Available Hosts  
- `.127` = Broadcast

---

## 🔢 Subnet Masks

### What does `255.255.255.0` mean?
The last octet is available for host addresses, giving one block from `0` to `255`.

### What block size does `255.255.255.128` create?
Blocks of **128**.

### What block size does `255.255.255.192` create?
Blocks of **64**.

### What block size does `255.255.255.224` create?
Blocks of **32**.

### What block size does `255.255.255.240` create?
Blocks of **16**.

### What block size does `255.255.255.248` create?
Blocks of **8**.

### What block size does `255.255.255.252` create?
Blocks of **4**.

### How can I quickly calculate the block size?
Use:

```text
256 - relevant mask value
```

Example:

```text
256 - 224 = 32
```

So `255.255.255.224` creates blocks of 32 addresses.

---

## 📦 Subnet Ranges

### How do I know which subnet an IP belongs to?
Find the block created by the subnet mask and check where the IP falls.

Example:

```text
IP:   192.168.1.70
Mask: 255.255.255.192
```

The mask creates blocks of 64:

```text
0–63
64–127
128–191
192–255
```

`70` belongs to the `64–127` block.

### What is subnet overlap?
It happens when two different subnets use part of the same IP range.

### Why is subnet overlap a problem?
A router may not know which interface should be used to reach an address.

### How do I check for subnet overlap?
Compare the complete ranges of both subnets.

Example:

```text
Subnet A: 0–127
Subnet B: 64–127
```

They overlap.

Example:

```text
Subnet A: 0–63
Subnet B: 64–127
```

They do not overlap.

---

## 🚪 Gateway

### What is a gateway?
The router used to reach devices outside the local network.

### What is a default gateway?
The router a host uses when the destination is not inside its own subnet.

### Does a host's gateway need to be in the same subnet?
Yes.

### If a host is `192.168.1.2`, can its gateway be `10.0.0.1`?
Not normally. The host must be able to reach its gateway directly.

### What gateway should a host usually use in NetPractice?
The router interface directly connected to the host's subnet.

---

## 🧭 Routing

### What is a routing table?
A list of rules that tells a router where to send packets.

### What is a route?
A rule that maps a destination network to a next hop.

### What goes on the left side of a route?
The **destination network**.

### What goes on the right side of a route?
The **next hop** or gateway.

### What is a next hop?
The next router that should receive the packet.

### If R1 wants to reach a network behind R2, what should the next hop be?
The IP of the R2 interface directly connected to R1.

### Should the next hop be the final host?
Usually no. It should be the next router on the path.

### Does a router need a route for a network directly connected to one of its interfaces?
Usually no. It already knows that network.

---

## 🌍 Default Route

### What is a default route?
The route used when no more specific route matches the destination.

### What does `0.0.0.0/0` mean?
Any destination.

### Is `default` the same idea as `0.0.0.0/0`?
Yes.

### When is a default route useful?
When many unknown destinations should all be sent to the same router.

### Can two routers point their default routes at each other?
They can, but it may create a routing loop for unknown destinations.

---

## ↔️ Forward and Reverse Paths

### What is the forward path?
The path from the source to the destination.

### What is the reverse path?
The path from the destination back to the source.

### Why does NetPractice check both directions?
Because successful communication requires both devices to be able to reply.

### What does `No forward way` mean?
The destination cannot be reached from the source.

### What does `No reverse way` mean?
The destination was reached, but the reply cannot return to the source.

---

## 🔀 Routers and Switches

### What does a switch do?
It connects devices inside the same local network.

### Does a switch route traffic between different networks?
No.

### What does a router do?
It connects different networks and forwards packets between them.

### Can two router interfaces belong to overlapping subnets?
They should not. It can create ambiguous routing decisions.

### What does `multiple interface match` usually mean?
More than one interface appears valid for the same destination, often because of overlapping subnets.

---

## 🌐 Internet Routes

### Why does the Internet need a route back to internal hosts?
Because reaching the Internet is only half of the communication. The reply also needs a route back.

### If the Internet has only one route field, what may be necessary?
A larger route that covers several smaller internal subnets.

### What is route aggregation?
Using one larger network route to represent several smaller subnets.

### Why is route aggregation useful?
It reduces the number of routing entries required.

### If the Internet has a fixed route, what should I check first?
Which IP range that route covers.

### If the Internet route is fixed, what may I need to change?
The internal subnet layout so the required hosts fit inside the reachable range.

---

## ⚠️ Special IP Ranges

### Can every IPv4 address be assigned to a normal host?
No.

### What is the multicast range?
`224.0.0.0` to `239.255.255.255`.

### Can `233.x.x.x` be used as a normal host address in NetPractice?
No. It belongs to the multicast range.

---

## ⚡ Quick Mental Checklist

Before pressing **Check again**, ask:

1. Are directly connected interfaces in the same subnet?
2. Are different links using different, non-overlapping subnets?
3. Is any host using a network or broadcast address?
4. Does each host point to its local router as gateway?
5. Does each route use the correct destination network?
6. Does each route point to the correct next hop?
7. Is there a valid forward path?
8. Is there a valid reverse path?
9. Can the Internet route back to the internal networks?

---

## 🎯 One-Line Rules to Remember

> **Subnet mask = where the network ends and the host part begins.**

> **Gateway = where I send traffic that is outside my local network.**

> **Route left side = destination network.**

> **Route right side = next hop.**

> **No reverse way = the reply cannot come back.**

> **Overlap = two subnets share part of the same address range.**

> **Always think in ranges, not only individual IP addresses.**
