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

To store values, we'll use an array of integers. Since we may have multiple handlers running concurrently, we'll use a channel to synchronize access. This channel will have a buffer size of 1, and we'll initialize it with an empty integer array. To add to the array, the `broadcast` handler will read the array to consume the single copy, append to it, and write it back to the channel. Similarly, the `read` handler will read the array from the channel and return a copy in the response.

```go
var messages chan []int

func AddSingleNodeBroadcastHandle(ctx context.Context, n *maelstrom.Node) {
  messages = make(chan []int, 1)
  messages <- make([]int, 0, 1)
}
```

```go
broadcast := func(req maelstrom.Message) error {
  var body map[string]any
  if err := json.Unmarshal(req.Body, &body); err != nil {
    return err
  }

  message, _ := getMessage(body)

  msgs := <-messages
  msgs = append(msgs, int(message))
  messages <- msgs
}
```

```go
read := func(req maelstrom.Message) error {
  msgs := <-messages
  // Now that we have a local copy, we can immediately return it to the channel so that other
  // goroutines are unblocked.
  messages <- msgs

  resp := make(map[string]any)
  resp["type"] = "read_ok"
  resp["messages"] = msgs

  return n.Reply(req, resp)
}
```

My code for Part A is at [`internal/broadcast/3a.go`](https://github.com/lynshi/gossip-glomers/blob/main/internal/broadcast/3a.go).

## A note on topology
The `topology` message type is odd. The problem statement says that we can ignore the provided neighbors and build our own topology from Maelstrom's list of all nodes, as all nodes can communicate with each other. At first, I was confused about this as there doesn't seem to be a point of this message then, and based on a glance at [`community.fly.io`](https://community.fly.io/tag/dist-sys-challenge) I wasn't the only one[^1]. However, someone explained that ["the topology is just a way to logically arrange nodes" and that Maelstrom allows you to select a topology](https://community.fly.io/t/using-a-own-topology/11057/6), so my conclusion is that the topology can be interpreted as a recommendation for inter-node communication[^2] and also provides consistency with Maelstrom's problem formulation, which causes the Maelstrom controller to send a `topology` message when starting up the nodes. Ultimately, I chose to ignore this message for all sections of this challenge as I constructed my own topology later on.

---

# 3b: Multi-Node Broadcast
In Part B, we introduce multiple nodes, and upon receiving a `broadcast` message a node must distribute that message to all other nodes within a few seconds. Because all messages are unique, I decided to store the messages in a `map[int]interface{}` instead so that saved messages are automatically deduplicated [^3].

As the problem is getting more complicated, I also introduced a `MultiNodeNode` type to encapsulate the implementation for Part B. This also helps me easily write distinct implementations for each section so that I can link to standalone files for the blog 😆.

```go
type MultiNodeNode struct {
	mn *maelstrom.Node

	// Keeps track of received messages.
	messages chan map[int]interface{}

	// Queues up messages yet to be sent to other nodes.
	queue chan int
}
```

<!--- Footnotes -->
[^0]: Note that nodes never crash, so we don't have to worry about persisting data to disk.
[^1]: [maelstrom challenge: request to implement topology and then ignore it is very confusing.](https://community.fly.io/t/maelstrom-challenge-request-to-implement-topology-and-then-ignore-it-is-very-confusing/11337)
[^2]: This is relevant for later challenges, where efficiency requirements mean you can't have a node talk to every other node.
[^3]: Go doesn't have sets, so a `map[int]interface{}` is a workaround for creating a set of integers as the value in each key-value pair is ignored and usually set to `nil`.
