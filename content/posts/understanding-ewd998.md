---
title: "Understanding EWD998: Shmuel Safra's version of termination detection"
description: >-
  Oh no, there are math symbols!
date: 2023-02-18T21:55:07-08:00
tags:
  - Distributed systems
  - TLA+
---

I’ll soon be attending [Markus Kuppe](https://twitter.com/lemmster)’s [workshop on TLA+](https://github.com/tlaplus-workshops/ewd998) and one of the pre-read materials is [Dijkstra’s EWD998 - Shmuel Safra's version of termination detection](https://www.cs.utexas.edu/users/EWD/ewd09xx/EWD998.PDF). I haven’t read a serious, academic paper since college like 3 years ago[^0], so it was quite an adventure getting back into the swing of things. As I was reading, I spent a lot of time going back and forth to make things make sense, because the hallmark of a real paper is that you can’t just consume it in one go if you actually want to understand the material.

One of the nice things about a college course/lecture is that you get the teacher’s notes edition of the paper/proof. You get the “why” behind statements in the paper, not just how. Whereas if I’m reading a paper by myself, I often have to pause and ask “Well, I see that you’ve gone from Point A to Point B, and I believe your logic was sound, but why did we do that? And how on earth did you come up with Point A in the first place?!”

So, after I spent a good chunk of my time unwrapping this paper, I figured I could throw something onto the Internet to make someone else’s reading a little easier. Read on for what is essentially EWD998 restated with longer explanations where Dijkstra's brevity forced me to stop and think :)

Be warned that the mathematical notation is still around (though I try to also explain it in English). It is a succinct yet clear way to express the ideas, which is rather neat![^1]

----------------

The premise of the paper is that there’s a circle of $N$ machines indexed from 0. Each machine is either active or passive. Active machines may send messages to other machines. These messages take time to travel between machines; however, no messages are ever lost. An active machine can become passive "spontaneously". Meanwhile, a passive machine only becomes active upon receipt of a message.

We want to figure out when the system is stable; that is, all machines are passive and there are no messages in transit. We’re going to devise a solution where the first machine, $m_0$[^2], can detect when the system has reached a stable state (i.e. termination). This termination detection algorithm is called "the probe".

The machines, regardless of their active/passive status, can always communicate such that:
1. Machine $m_0$ can kick off the probe by sending a signal (i.e. a _token_) to $m_{N - 1}$.
2. $m_{i + 1}$ can always propogate the probe by sending the token to $m_{i}$, even if $m_{i + 1}$ is passive.

Eventually, this token makes its way back to $m_0$. Based on information on the token and the state of $m_0$, we'll be able to conclude whether the stable state has been reached[^3].

----------

The strategy for solving this problem is to iteratively construct an invariant $P$ that is always true in the system and rules to maintain the invariant. This invariant will help us determine the conditions that are satisfied if and only if the system is stable when the token returns to $m_0$.

Concretely, we'll
1. Construct an invariant.
2. Devise a rule for system operation to maintain it.
3. Find an edge case that causes the invariant to break.

Each edge case can be addressed by adding a new condition to the invariant, so we loop back to 1. When we add a new condition, we have to be careful that the condition still allows us to detect termination.

Based on Dijkstra's closing[^4], this strategy seem to have been discussed for some time, which explains why EWD998 reads so matter-of-factly[^5]. But here I'm telling you up front what to expect, so you don't have to tear your hair out wondering what magic possessed Safra such that the logic moves so effortlessly from step to step!

-----

Let $t$ be the index of the machine holding the token and $B$ be the number of messages on their way. We would like to determine whether the system has reached stable state when the token returns to $m_0$ (i.e. $t = 0$), and our definitions mean that termination can be stated as[^6]
$$\forall i \in [0, N): m_i \text{ is passive} \land B = 0$$

In other words, the system is stable if all machines are passive and no messages are on their way (to wake up a machine).

It's clear we need to know $B$ to detect termination. If we knew how many messages had been sent and received by each machine, we can easily compute $B$. Thus, let's construct our first invariant, $P_0$, to help us keep track of $B$.

$$P_0: \quad B = \Sigma_{i = 0}^{N - 1}c_i$$

Intuitively, we want $c_i$ to be the net messages sent by machine $i$, and we enforce this by adding a rule to the system.

> Rule 0: Each machine maintains its own counter $c_i$, incrementing it by 1 when it sends a message and decrementing it by 1 when it receives a message[^7]. It follows that we should initialize all $c_i$ to 0 when the machines first start.

$P_0$ doesn't depend on $t$, so $m_0$ receiving the token after it has been sent around the ring doesn't help us determine anything. Consequently, let's say that the token has a value $q$ and add a condition $P_1$.

$$P_1: \quad \forall i \in (t, N): m_i \text{ is passive} \land \Sigma_{i = t + 1}^{N - 1}c_i = q$$

If $P_1$ is true, that means that all machines that have already handled the token are passive, and the sum of their counters $c_i$ is equal to the value of the token. The token being integer-valued and connected to the $c_i$ is critical to the termination detection algorithm as it encodes information about $B$, which is hard to know without an overall view of the system, into something that can be passed around[^8].

Now, our invariant is $P_0 \land P_1$. If the invariant holds, when the token returns at $t = 0$,

$$
\begin{aligned}
& P_0 \land P_1 \land t = 0 \\\\
\implies & B = \Sigma_{i = 0}^{N - 1}c_i \land \forall i \in (0, N): m_i \text{ is passive} \land \Sigma_{i = 1}^{N - 1}c_i = q \\\\
\implies & B = c_0 + q \land \forall i \in (0, N): m_i \text{ is passive}
\end{aligned}
$$

That is, all machines other than $m_0$ are passive (plus the little equation, $c_0 + q = B$, that follows from some arithmetic).

Recall that all machines are passive and $B = 0$ when the system is stable. Thus, given the invariant, the system is stable when $t = 0$ if $m_0$ is passive (it's the only machine we aren't certain about the state of) and $c_0 + q = 0$.

--------

How do we maintain $P_1$? Observe that the probe starts with $t = N - 1, q = 0$. The boundary conditions mean that at this point, there are no terms in the $\forall$ and $\Sigma$ operations, so while we aren't making a statement on which machines are passive, we _are_ stating that $q$ must be 0 since there's nothing being summed. That leads us to `Rule 1`.
> Rule 1: When the probe starts, $m_0$ sends the token with a value of 0 (i.e. $q = 0$) to $m_{N - 1}$.

When a machine transmits the token, due to the invariant it must be making a statement that it is passive. It also must update the value of the token. Thus, we introduce `Rule 2`.
> Rule 2: A machine $m_{i + 1}$ _only_ transmits the token to the next machine $m_i$ after $m_{i + 1}$ becomes passive, and the token's value is updated to $q = q + c_{i + 1}$.

------

When is $P_1$ false? Most directly, $P_1$ is false if, when the token is at index $t$, some machine $m_i$, $t < i < N$, is active _or_ $\Sigma_{i = t + 1}^{N - 1}c_i \neq q$. Let $m_i$ be the first machine to cause a violation of $P_1$.

Suppose $m_i$ violates $P_1$ by becoming active. Recall that a machine only forwards the token when it becomes passive, so if $m_i$ is now active, it must have received a message.

Meanwhile, suppose $\Sigma_{j = t + 1}^{N - 1}c_j \neq q$. Since $m_i$ is the _first_ machine to cause a violation of $P_1$, $\Sigma_{j = i + 1}^{N - 1}c_j = q_i$ (let $q_k$ be the value of the token when $m_k$ received it) must have been true for the whole period during which $m_i$ held the token. By `Rule 2`, $\Sigma_{j = i}^{N - 1} = q_{i - 1}$ must also be true when $m_i$ transmits the token, and remain true until the violation is caused by $m_i$. A violation can be caused by $m_i$ if $c_i$ changes. As $m_i$ is passive, $c_i$ can only change via receipt of a message.

We conclude that the only way for $m_i$ to cause a violation of $P_1$ is by receiving a message. In real world terms, this means that $m_i$ was woken up by a message that was either still in-transit or yet to be sent when it became passive. Further, in the instance _right before_ the violation, there must be a message on its way to $m_i$. This is expressed by $B \geq 1$.

Since $P_0 \land P_1$ is true the moment before $P_1$ is violated, we have

$$
\begin{aligned}
1 \leq B = & \Sigma_{i = 0}^{N - 1}c_i \\\\
B = & \Sigma_{i = 0}^{t}c_i + \Sigma_{i = t + 1}^{N - 1}c_i \\\\
B = & \Sigma_{i = 0}^{t}c_i + q \\\\
\therefore 0 < 1 &\leq \Sigma_{i = 0}^{t}c_i + q \\\\
\\\\
P_2: \quad \quad \quad & \Sigma_{i = 0}^{t}c_i + q > 0
\end{aligned}
$$

Observe that $P_2$ is true even if $m_i$, $t < i < N$, receives a message because no machine with $i > t$ is involved in the statement. So we can say either $P_1$ or $P_2$ is true, and our invariant becomes $P_0 \land (P_1 \lor P_2)$.

When the probe concludes at $t = 0$, $P_2$ evaluates as $c_0 + q > 0$. Recall that when the system is stable, we have $c_0 + q = 0$, so $P_2$ must be false when that occurs. Thus, the invariant returns to just $P_0 \land P_1$ when $c_0 + q = 0$, and we already know how to conclude termination when $P_0 \land P_1$. As a result, introducing $P_2$ does not make it any harder to conclude termination.

--------

It's getting gnarly now! Under $P_2$, $m_i$ where $0 \leq i \leq t$ is free to send as many messages as desired - doing so only increases $c_i$, so the invariant is maintained. However, if $c_i$ decreases due to $m_i$ receiving a message, we might falsify $P_2$! So, let's add a statement to the invariant to capture the receipt of a message by $m_i$.

$$
P_3: \quad \exists i \in [0, t]: \text{machine } i \text{ is black}
$$

We also add a corresponding `Rule 3`.
> Rule 3: When a machine receives a message, it changes its color to black.

Fortunately, this still doesn't make it any harder to detect termination. Our invariant is now $P_0 \land (P_1 \lor P_2 \lor P_3)$, and we already know that $P_2$ is false at termination, so we just need to make sure that $P_3$ is also false at termination.

At $t = 0$, when we can try to determine termination, $m_0$ must be black to satisfy $P_3$. If $P_3$ is false, we can apply our previously derived techniques for detecting termination; thus, if $m_0$ is white when the token returns, we have a shot at detecting termination.

-------

What, you thought we were going to stop at $P_3$? Nope! If a black machine propagates the token, we are in danger of violating $P_3$ since that machine falls out of the set of machines referenced in $P_3$. This doesn't necessarily mean the full invariant is invalidated as other parts of it can be true, so let's construct a scenario where $P_1$ and $P_2$ are also false when a black machine propagates the token.

Let $m_t$, $t > 0$ be the last black machine. Suppose that machines $m_i$, $i < t$ have $c_i = 0$ and are white. While $m_t$ holds the token, it sends a message to $m_x$, $x > t$ - this breaks $P_1$!

Then, $m_t$ passes the token to $m_{t - 1}$. Suddenly, $P_2$ is falsified as $\sum_{i = 0}^{t - 1}c_i = 0$. Meanwhile, no machines $0..t - 1$ received a message, so all machines within the set described in $P_3$ are white and $P_3$ is false too!

-------

Clearly, we have to introduce $P_4$ to rectify the situation. Let's color the token!

$$P_4: \quad \text{the token is black}$$

> Rule 4: If a black machine is to transmit the token, it colors the token black before sending it.

If the token is white when it makes it back to $m_0$, then $P_4$ is false, and we have a chance at detecting termination.

At this point, we are done modifying the invariant! The color of the token never changes back to white from black, so it can never be made untrue at some later step.

Overall, we constructed the invariant as follows:
1. Constructed $P_1$.
2. $P_1$ can be falsified by the receipt of a message by $m_i$, $t < i < N$, so we constructed $P_2$ which is true at least as early as the instant before $P_1$ becomes false.
3. $P_2$ can be falsified by receipt of a message by $m_i$, $0 \leq i \leq t$, so we constructed $P_3$, which becomes true the instant $m_i$ receives a message.
4. $P_3$ can be falsified by the last black machine transmitting the token, so we constructed $P_4$, which becomes true whenever any black machine transmits the token.
5. Once true, $P_4$ never becomes false.

Thus, by construction our invariant holds for all states in the system.

-------

Let us summarize what we know so far. The invariant $P_0 \land (P_1 \lor P_2 \lor P_3 \lor P_4)$ is always true. $P_0$ is true by design, and we know how to detect termination when $P_0 \land P_1$. By the invariant, $P_1$ must be true if $P_2$, $P_3$, and $P_4$ are all false. Thus, we have a chance at detecting termination when the token returns at $t = 0$ if:
1. $c_0 + q = 0 \implies \neg P_2$ (i.e. $P_2$ is false)
2. $m_0 \text{ is white} \implies \neg P_3$
3. $\text{The token is white} \implies \neg P_4$

When only $P_1$ is true when the token returns, all machines $1..N-1$ are passive, so the system is totally passive if $m_0$ is also passive.

Thus, points 1-3 and $m_0$ being passive at $t = 0$ are the conditions for concluding that the system has reached stable state.

-----

What happens if any of the conditions are false? Well, we can't make a conclusive determination, so we run the probe again - what, you thought we could only probe once?
> Rule 5: $m_0$ initiates another probe after an unsuccessful probe.

Running the probe again is insufficient though! If the token can't change back to white, the outcome of the next probe will be no different ($P_4$ will always be true). Further, even if the token color can change, if machine colors don't change, then either $m_0$ stays black or the token is turned back to black as it makes its way around the ring. Consequently, we need to be able to change colors for both the token and all machines.

While changing colors back to white, we must be careful to ensure the invariant $P_0 \land (P_1 \lor P_2 \lor P_3 \lor P_4)$ is maintained! Otherwise, all of our good work will have gone to waste.

Note that the invariant is true when the probe is first started. $P_0$ is always true thanks to `Rule 0`, and we established earlier that $P_1$ is trivially true when $m_0$ sends the token to $m_{N - 1}$. So, we can whiten $m_0$ and the token when initiating the probe again.
> Rule 6: At initiation of the probe, $m_0$ whitens itself and the token.

Additionally, note that $P_3$ only makes a statement about machines whose index is $t$ or less. As a result, it's always safe to whiten a machine whose index exceeds $t$.
> Rule 7: After transmitting the token to $m_i$, $m_{i + 1}$ whitens itself.

It follows that when no messages are in-flight ($B = 0$), no machines turn black anymore, all black machines whiten themselves eventually, and the token eventually stays white. When $B=0$ _and_ all machines are passive, every $c_i$ becomes constant because no machine is able to send messages and there are no messages in transit that might decrease some $c_i$. When a probe is started during such a state, $P_1$ stays true throughout the probe. When the probe ends, $c_0 + q = B = 0$. Therefore, all conditions for detecting termination eventually become true after the system becomes passive![^9]

Exhausted? That makes two of us!

-------

I started writing this post while I was reading EWD998, but I've already completed the workshop and have yet to publish it 😅

So, I'll take this opportunity to note that the workshop was terrific! Many thanks and kudos to Markus for putting together such a great experience. We implemented the termination detection algorithm - you can see my spec [here](https://github.com/lynshi/ewd998/blob/main/EWD998.tla) - and I learned a lot about TLA+! Hopefully I'll get a chance to write a spec for some of the systems I've designed to catch all the edge cases I undoubtedly have missed.

<!--- Footnotes -->

[^0]: Ackchyually, I read the [Raft paper](https://raft.github.io/raft.pdf) when I first started working (technically still almost 3 years ago though!) and [Paxos vs Raft: Have we reached consensus on distributed consensus?](https://arxiv.org/pdf/2004.05074.pdf) sometime last year. But they didn’t have any mathematical notation, so my brain didn’t really get fried and therefore it doesn't count.

[^1]: After reading this article, Leslie Lamport wrote to say that "math isn't a hinderance to understanding but rather a requirement." I try to be a bit gentler, but I do agree! Words can be ambiguous, but symbols are not.

[^2]: Dijkstra uses the term `nr.i` to refer to machine `i`, but that doesn't look good and is even a little confusing in LaTeX.

[^3]: This concludes the easy part! If I were a student, I'd be worried about plagiarism because I'm almost quoting the paper verbatim so far. Thankfully, we don't have to be so rigorous here.

[^4]: "Neither the algorithm nor its variations are the point of the note, which is about the derivation strategy, which worked again."

[^5]: EWD998 references [EWD840](https://www.cs.utexas.edu/users/EWD/ewd08xx/EWD840.PDF), which solves a similar problem with the assumption of instantaneous message delivery. EWD840 feels a bit more accessible - maybe because I read (\*cough\* skimmed) it after EWD998, but it also seems to contain a bit more color on why certain steps are taken.

[^6]: Dijkstra uses $\underline{A}$, $\underline{S}$, and $\underline{E}$ for $\forall$ (for all), $\Sigma$ (sum), and $\exists$ (exists), respectively. I'll use the math symbols since they're more familiar to me. All ranges are over integers, so $[0, 5) = \\{ 0, 1, 2, 3, 4 \\}$.

[^7]: Dijkstra remarks that because the value of $B$ changes in a distributed fashion (i.e. based on the individual sending/receiving actions of each machine), our only option to compute $B$ is to try to compute it in a distributed fashion too. Sounds reasonable to me!

[^8]: This explains why Dijkstra's previous solution in EWD840 only uses a colored token. In that scenario, messages are delivered instantaneously, so we don't have to worry about keeping track of how many messages are in-flight. This also suggests that an integer-valued token (as well as formulation of the problem) was Shmuel Safra's key contribution.

[^9]: "Eventually" is doing a lot of work for us here! As Markus clarified, this derivation only ensures that the rules for system operation and termination detection are guaranteed to be correct. No statements are made about non-correctness properties (e.g. efficiency - whether there are opportunities to reduce the lag between termination and detection). Similarly, the spec constructed in the workshop only satisfies safety and liveness; that is, it shows that the algorithm eventually detects termination if the system terminates, and termination is correctly detected. It turns out that [TLA+ can also be used to study non-correctness properties](https://github.com/tlaplus/Examples/tree/master/specifications/ewd998#statistics) too!
