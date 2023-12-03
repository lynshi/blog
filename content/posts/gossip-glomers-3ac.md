---
title: "Gossip Glomers 3 (a-c): Single-Node, Multi-Node, and Fault Tolerant Broadcast"
date: 2023-11-29T14:57:01-08:00
draft: true
description: >-
  It gets more involved at 3d, so let's talk about a-c first.
categories:
  - Academic
tags:
  - Distributed systems
  - fly.io
  - Gossip Glomers
---

[Last time](https://lynshi.github.io/posts/gossip-glomers-intro-and-unique-id-generation/) we introduced the Gossip Glomers challenge from [Fly.io](https://fly.io/) and discussed our approach to [Challenge #2: Unique ID Generation](https://fly.io/dist-sys/2/).

This time, we'll talk about the first three parts of [Challenge #3: Broadcast](https://fly.io/dist-sys/3a/). Parts D and E are saved for a separate post as they're a bit more involved.

The overall theme of Challenge 3 is to build a broacast system to propagate messages to all nodes[^0]. We iteratively build up our system, from a single-node cluster that simply stores and returns received messages, to a multi-node cluster that shares received messages, to a fault-tolerant multi-node cluster that can operate even during network partitions by Part C (D and E are about efficiency).

As before, my code is on GitHub at [`lynshi/gossip-glomers`](https://github.com/lynshi/gossip-glomers) under [`internal/broadcast`](https://github.com/lynshi/gossip-glomers/tree/main/internal/broadcast).

---
# 3a: Single-Node Broadcast
We start off implementing only one node to ensure that we receive and save broadcasted messages correctly. There are three types of messages our node needs to handle:
* `broadcast`: Store the value received.
* `read`: Return all values received by the node.
* `topology`: Receive information about neighbors.

## broadcast


<!--- Footnotes -->
[^0]: Note that nodes never crash, so we don't have to worry about persisting data to disk.
