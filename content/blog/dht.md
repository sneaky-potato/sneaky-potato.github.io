---
title: "Distributed Hash Table"
date: 2025-06-05
description: "fault tolerant p2p hash table"
tags: ["tech"]
math: true
---

{{< lead >}}
Data spread across nodes, each knowing whereabouts of the other
{{< /lead >}}

## Distributed Key Value Store
Imagine data so large that one server cannot store it, we need to split the data into multiple nodes.
So, say there are `n` nodes storing some key values as shown in this table

| Node  | Key                 |
| ----  | ---                 |
| $n_1$ | $k_a$, $k_b$, $k_c$ |
| $n_2$ | $k_d$               |
| $n_3$ | $k_a$, $k_e$        |

Now, a new node `n4` comes and establishes contact with `n2`. This new node `n4` wants `ke`, but `n2` unfortunately does not have it.
We should be able to handle this very case.

So _any_ **new** node connecting to _any_ node can ask for _any_ key.

## Problem: Node discovery
A single node cannot store information about all the available nodes (it might but we want to make it scalable to some extent).

### Solution: Kademlia
A solution to pure p2p distributed key value store, wikipedia article [here](https://en.wikipedia.org/wiki/Kademlia#:~:text=Kademlia%20is%20a%20distributed%20hash,and%20David%20Mazi%C3%A8res%20in%202002.).
Nice video explanation is [here](https://www.youtube.com/watch?v=_kCHOpINA5g)

### Unique identifiers
Each node is assigned a random 160 bit identifier. Same goes for the keys as well.
This can be achieved by passing the node ids and key ids through SHA1 to produce 160 bit hashes for these.

## Data Ownership
Given a key `k` and the set of nodes, how to decide on which node this key should reside?

**Idea**: Calculate some sort of distance between the key and nodes and select the node which is _closest_ to key.

How to define the distance then? It should follow the follwoing three properties:

- $d\(x, x\) = 0$; distance of some entity from itself should be zero
- $d\(x, y\) > 0,  x \neq y$; distance of two different entities must be positive
- $d\(x, y\) + d\(y, z\) >= d\(x, z\)$; triangle inequality

Since `x` and `y` here would be 160 bit ids, we need a function which would work well with this. One such funciton which would work is the XOR function.
***XOR*** satisfies all three properties which we need in this case:
- $d\(x, x\) = x \oplus x = 0$
- $d\(x, y\) = x \oplus y > 0$
- $d\(x, y\) + d\(y, z\) = \(x \oplus y\) \oplus \(y \oplus z\) = x \oplus z = d\(x, z\)$

An example showing what we mean with this: say we are working with 4 bit ids to keep things simple

| Entity    | ID | Bit representation |
| ----      | ---| ---                |
| node $n_1$ | 15 | 1111               |
| node $n_2$ |  5 | 0101               |
| key  $k_a$ |  6 | 0110               |
| key  $k_b$ | 13 | 1101               |

$d\(k_a, n_1\) = 0110 \oplus 1111 = 1001 = 9$

$d\(k_a, n_2\) = 0110 \oplus 0101 = 0011 = 3$

since 3 < 9, key $k_a$ should reside with node $n_2$

This can be visualised with a binary tree as follows
```kroki{type=d2}
direction: down

root -> b0: "0"
root -> b1: "1"

b0 -> b00: "0"
b0 -> b01: "1"
b1 -> b10: "0"
b1 -> b11: "1"

b01 -> b010: "0"
b01 -> b011: "1"
b11 -> b110: "0"
b11 -> b111: "1"

b010 -> n2: "1"
b011 -> ka: "0"
b111 -> n1: "1"

root: "Root"
b0: "0"
b00: "00"
b1: "1"
b10: "10"
b01: "01"
b11: "11"
b010: "010"
b011: "011"
b110: "110"
b111: "111"
n2: "n2 (0101)"
ka: "ka (0110)"
n1: "n1 (1111)"

n2.style.fill: "#e0f2fe"
n1.style.fill: "#e0f2fe"
ka.style.fill: "#fef08a"
```

We can see how $k_a$ is closer to $n_2$ in that case

## Routing
Nodes must know some other nodes so that information can be passed. But we want to avoid storing information of all nodes. This is where the tree structure would come handy.

**Idea**: Every node knows at least one node in each **subtree** that it is _not_ part of
![subtree_routing](./subtree_routing.png)
So node `n1` must store contact in the 4 subtrees 1, 00, 011, 0101. This ensures any node can get information on any other node easily without storing all node addresses.

