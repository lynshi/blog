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

To store values, we'll use an array of integers. Since we may have multiple handlers running concurrently, we'll use a channel to synchronize access. This channel will have a buffer size of 1 and we'll initialize it with an empty integer array. To add to the array, the `broadcast` handler will read the array to consume the single copy, append to it, and write it back to the channel. Similarly, the `read` handler will read the array from the channel and return a copy in the response.

In order to make it easy to have a distinct implementation for each part, we'll wrap the data structures used in a struct. For example, for Part A we'll define `SingleNodeNode`.

```go
type SingleNodeNode struct {
  mn *maelstrom.Node

  messages chan []int
}

func NewSingleNodeNode(ctx context.Context, n *maelstrom.Node) {
  messages = make(chan []int, 1)
  messages <- make([]int, 0, 1)

  n := SingleNodeNode{
    mn:       mn,
    messages: messages,
  }

  go func() {
    <-ctx.Done()
    close(messages)
  }()

  n.addBroadcastHandle()
  n.addReadHandle()
  n.addTopologyHandle()

  return &n
}
```

```go
func (n *SingleNodeNode) broadcastSingleNodeBuilder() maelstrom.HandlerFunc {
  broadcast := func(req maelstrom.Message) error {
    // ...

    message, _ := getMessage(body)

    msgs := <-n.messages
    msgs = append(msgs, int(message))
    n.messages <- msgs

    // ...
  }

  return broadcast
}
```

```go
func (n *SingleNodeNode) readBuilder() maelstrom.HandlerFunc {
  read := func(req maelstrom.Message) error {
    msgs := <-n.messages
    // Now that we have a local copy, we can immediately return it to the channel so that other
    // goroutines are unblocked.
    n.messages <- msgs

    resp := make(map[string]any)
    resp["type"] = "read_ok"
    resp["messages"] = msgs

    return n.mn.Reply(req, resp)
  }

  return read
}
```

My code for Part A is at [`internal/broadcast/3a.go`](https://github.com/lynshi/gossip-glomers/blob/main/internal/broadcast/3a.go).

## A note on topology
The `topology` message type is odd. The problem statement says that we can ignore the provided neighbors and build our own topology from Maelstrom's list of all nodes, as all nodes can communicate with each other. At first, I was confused about this as there doesn't seem to be a point of this message then, and based on a glance at [community.fly.io](https://community.fly.io/tag/dist-sys-challenge) I wasn't the only one[^1]. However, someone explained that ["the topology is just a way to logically arrange nodes" and that Maelstrom allows you to select a topology](https://community.fly.io/t/using-a-own-topology/11057/6), so my conclusion is that the topology can be interpreted as a recommendation for inter-node communication[^2] and also provides consistency with Maelstrom's problem formulation, which causes the Maelstrom controller to send a `topology` message when starting up the nodes. Ultimately, I chose to ignore this message for all sections of this challenge as I constructed my own topology later on.

---

# 3b: Multi-Node Broadcast
In [Part B](https://fly.io/dist-sys/3b/), we introduce multiple nodes, and upon receiving a `broadcast` message a node must distribute that message to all other nodes within a few seconds. Because all messages are unique, I decided to store the messages in a `map[int]interface{}` instead so that saved messages are automatically deduplicated [^3]. Similarly to previous section, we initialize a `MultiNodeNode` by adding an empty map to the `messages` channel.

```go
type MultiNodeNode struct {
  mn *maelstrom.Node

  messages chan map[int]interface{}
}

func NewMultiNodeNode(ctx context.Context, mn *maelstrom.Node) *MultiNodeNode {
  messages := make(chan map[int]interface{}, 1)
  messages <- make(map[int]interface{})

  n := &MultiNodeNode{
    mn:       mn,
    messages: messages,
  }

  // ...
  return &n
}
```

Since we have to forward messages received from the controller to other nodes as soon as possible, upon receipt of a `broadcast` message that did not originate from another node, the node initiates a goroutine to send the message to every other node. We use the Maelstrom-provided method `Send`, which is a fire-and-forget method that sends a message to the specified destination, as there aren't network failures in this scenario.

```go
broadcast := func(req maelstrom.Message) error {
  // ...

  // Only forward if the message did not come from another node.
  if strings.HasPrefix(req.Src, "n") {
    return nil
  }

  go func() {
    for _, neighbor := range n.mn.NodeIDs() {
      req := make(map[string]any)
      req["type"] = "broadcast"
      req["message"] = message

      go n.mn.Send(neighbor, req)
    }
  }()

  resp := make(map[string]any)
  resp["type"] = "broadcast_ok"

  return n.mn.Reply(req, resp)
}
```

The `read` handler is very similar to before, except we now must convert the map into an array.
```go
read := func(req maelstrom.Message) error {
  messages := <-n.messages
  n.messages <- messages

  resp := make(map[string]any)
  resp["type"] = "read_ok"
  resp_messages := make([]int, 0, len(messages))

  for v, _ := range messages {
    resp_messages = append(resp_messages, v)
  }

  resp["messages"] = resp_messages

  return n.mn.Reply(req, resp)
}
```

The full code for Part B is at [`internal/broadcast/3b.go`](https://github.com/lynshi/gossip-glomers/blob/main/internal/broadcast/3b.go).

---

# 3c: Fault Tolerant Broadcast
[Part C](https://fly.io/dist-sys/3c/) introduces network partitions to temporarily prevent inter-node communication. To accommodate, we use `RPC` instead of `Send`, as `RPC` checks for a successful response. `RPC` takes a callback handler, which we use to set a local `success` variable to `true` to prevent further retries[^4]. Otherwise, the network call is retried.

To reduce latency, we'll have every node forward received `broadcast` messages even if they weren't the first node to get it. This means we can detour around partitions when possible at the cost of message duplication. Of course, if a node finds that it has already received the message, we skip forwarding as it must have already done so earlier; this avoids infinite forwarding cycles. This resulted in tiny latencies (milliseconds): `:stable-latencies {0 0, 0.5 0, 0.95 0, 0.99 3, 1 3}`. Without this optimization — that is, if only the first node forwards — the latency was `:stable-latencies {0 0, 0.5 1022, 0.95 10504, 0.99 11563, 1 12205}`.

```go
func (n *FaultTolerantNode) forward_to_all(message int) {
  for _, neighbor := range n.mn.NodeIDs() {
    if neighbor == n.mn.ID() {
      continue
    }

    req := make(map[string]any)
    req["type"] = "broadcast"
    req["message"] = message

    go n.forward(neighbor, req)
  }
}

func (n *FaultTolerantNode) forward(neighbor string, body map[string]any) {
  for {
    success := false
    err := n.mn.RPC(neighbor, body, func(resp maelstrom.Message) error {
      success = true
      return nil
    })
    if err == nil && success {
      return
    }

    // Let's not bother with fancy backoffs since we know the partition
    // heals eventually.
    time.Sleep(500 * time.Millisecond)
  }
}
```

```go
broadcast := func(req maelstrom.Message) error {
  // ...

  messages := <-n.messages
  _, val_exists := messages[message]
  messages[message] = nil
  n.messages <- messages

  if !val_exists {
    go n.forward_to_all(message)
  }

  // ...
}
```

For the complete implementation for Part C, see [`internal/broadcast/3c.go`](https://github.com/lynshi/gossip-glomers/blob/main/internal/broadcast/3c.go).

<!--- Footnotes -->
[^0]: Note that nodes never crash so we don't have to worry about persisting data to disk.
[^1]: [maelstrom challenge: request to implement topology and then ignore it is very confusing.](https://community.fly.io/t/maelstrom-challenge-request-to-implement-topology-and-then-ignore-it-is-very-confusing/11337)
[^2]: This is relevant for later challenges, where efficiency requirements mean you can't have a node talk to every other node.
[^3]: Go doesn't have sets, so a `map[int]interface{}` is a workaround for creating a set of integers as the value in each key-value pair is ignored and usually set to `nil`.
[^4]: I didn't look at the documentation, but from brief experimentation I hypothesize that `Send` and `RPC` only return errors if there was an issue sending the message (which is probably rare). However, `Send` doesn't guarantee receipt, while `RPC` will call the callback handler once the message is received, even if the recipient node doesn't actually reply with anything.
