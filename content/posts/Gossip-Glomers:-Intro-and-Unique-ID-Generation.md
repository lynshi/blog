---
title: "Gossip Glomers: Intro and Unique ID Generation"
description: >-
  Who knew you could practice implementing distributed systems for fun?
date: 2023-11-26T21:06:12-08:00
categories:
  - Academic
tags:
  - Distributed systems
  - fly.io
  - Gossip Glomers
draft: true
---

One of the challenges for practicing implementation distributed systems is that it is not easy to simulate the various situations a distributed system might find itself in. Moreover, I previously could not even come up with an easy way to deploy a toy setup; the only thing I could think of is to use [minikube](https://minikube.sigs.k8s.io/docs/), but frankly at that point it is too much investment for me[^0].

Fortunately, I recently came across a series of distributed systems challenges created by [fly.io](https://fly.io/) and [Kyle Kingsbury](https://aphyr.com/about) (author of [Jepsen](https://jepsen.io/)): [Gossip Glomers](https://fly.io/dist-sys/). The challenges use [`Maelstrom`](https://github.com/jepsen-io/maelstrom), a framework for running and testing toy implementations of distributed systems, so that you only have to implement the individual nodes and not worry about anything else. Even better, Maelstrom provides a [Go library](https://pkg.go.dev/github.com/jepsen-io/maelstrom/demo/go) containing the boilerplate for creating Maelstrom Nodes, leaving you to focus on the fun stuff!

With this being so accessible, I guess I'll be working through the Gossip Glomers as I get time. As I complete challenges, I'll also write a post about my thought process[^1]! My code can be found at [https://github.com/lynshi/gossip-glomers](https://github.com/lynshi/gossip-glomers).

<!--- Footnotes -->

[^0]: We haven't even gotten to how to create and run proper tests yet!
[^1]: The [first challenge](https://fly.io/dist-sys/1/) is just a "Hello, World!" for getting you set up, so I'll skip it.
