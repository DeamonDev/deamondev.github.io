+++
date = '2026-05-12T22:04:26+02:00'
draft = false
title = 'Introduction to sequential consistency and linearizability'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems', 'sequential-consistency', 'linearizability', 'fault tolerance']
toc = true
[params]
math = true
+++

## Introduction

Recently, I’ve been enjoying a brilliant read of *Database Internals: A Deep Dive Into How Distributed Data Systems
Work* by Alex Petrov, which I was inspired to pick up by a colleague I had the pleasure of working with at a start-up (a
big hello to you, Raphael). This book is an exceptionally good introduction to storage engines, though I felt that some
rather important topics related to the theory of distributed systems were given rather short shrift (inevitably, given
that it is a book of moderate length). So I decided to expand on one of the topics that was only briefly covered there,
and which is often described in a rather convoluted manner in various tutorials or documentation pages for a given
database. I therefore decided to describe two key concurrency guarantees, leaving out other, presumably equally
important guarantees (such as casual consistency). However, I hope that after reading my notes, it will be a little
easier for someone to delve into the other guarantees and, above all, that it will clarify vague definitions and help to
understand the proofs of correctness [by waving one’s hands around](https://en.wikipedia.org/wiki/Hand-waving).

**The plan.** In this article I plan to define (formally) what sequential consistency is. Since sequential-consistency
is closely related to another crucial guarantee which is *linearizability* I'll define that one as well and provide a
minimal example showing the difference. Finally, I'll describe *Compare and Swap* – an atomic CPU instruction used in
multithreaded systems to achieve synchronization.

But before even talking about consistency guarantees, I think we should define what we mean by a distributed system.

## What is a distributed system?

I would like to start by saying that the level of formality used to answer the question of what a distributed system is
should be appropriately calibrated. There are many excellent works and publications on the subject, which are very good
sources of information on the topic. Links to the most important publications are provided in the [Links](#links)
section below.

Nevertheless, I believe it is important to maintain an appropriate level of formality, especially if we want to
understand exactly what the guarantees of concurrency are. Let's start with some simple definitions.

### Processes, objects and operations

A *process* is n execution thread of control that invokes *operations* on *shared objects*. We denote them by
\[P_1,P_2,\dots\] An *object* is shared data structure accessed by processes. Examples being \[\operatorname{register},
\operatorname{queue}, \operatorname{stack}, \operatorname{counter}\] Each object \(X\) has *sequential specification*,
\(\operatorname{SeqSpec}(X)\) which is a set of legal sequential histories over a given object. By *history* we mean
**sequence** of *events*.

An *operation* is a single method call performed by a process on an object. Formally it is pair of events: \[op=\langle
\operatorname{inv}_{P_i}(op), \operatorname{res}_{P_i}(op)\rangle\]

So each event is either invocation of response of an operation by some process \(P_i\).

Here is example of legal sequential history (all of these adjectives are formally defined in the very next section) for
stack: \[ \begin{aligned} \Sigma = \langle\, &\operatorname{inv}(
\mathrm{push}(a)), \\ &\operatorname{res}(\mathrm{push}(a) \to \mathrm{ok}), \\ &\operatorname{inv}(\mathrm{push}(
b)), \\ &\operatorname{res}(
\mathrm{push}(b) \to \mathrm{ok}), \\ &\operatorname{inv}(\mathrm{pop}()), \\ &\operatorname{res}(\mathrm{pop}() \to
b), \\ &\operatorname{inv}(\mathrm{pop}()), \\ &\operatorname{res}(\mathrm{pop}() \to a)
\,\rangle \end{aligned} \] It is not hard to see that \(\Sigma\) __is an element of__ \(\operatorname{SeqSpec}(
\operatorname{stack})\). Formally we should denote which process performed which invocation/response - but when thinking
at the level of sequential history for given object, we should narrow our mind to local (synchronous) reasoning.

We may give informal definition of distributed system as a *collection of independent processes that communicate
indirectly throught shared objects or directly throught messages*

Besides the sequential histories of the local objects, we consider the history of the whole system. For example, suppose
the system has two shared objects \[ X = \text{register}, \qquad S = \text{stack}. \] A whole system history could be:
\[ \begin{aligned} \Sigma = \langle\, &\operatorname{inv}_{P_1}(X.\mathrm{write}(1)), \\ &\operatorname{res}_{P_1}(
X.\mathrm{write} \to \mathrm{ok}), \\ &\operatorname{inv}_{P_2}(S.\mathrm{push}(a)), \\ &\operatorname{res}_{P_2}(
S.\mathrm{push} \to \mathrm{ok}), \\ &\operatorname{inv}_{P_1}(S.\mathrm{pop}()), \\ &\operatorname{res}_{P_1}(
S.\mathrm{pop} \to a)
\,\rangle . \end{aligned} \] This is history of the whole execution, because it includes operation on both \(X\) and
\(S\). We can then localize this history to \(S\) getting \[ \begin{aligned} \Sigma \mid S = \langle\,
&\operatorname{inv}
_{P_2}(S.\mathrm{push}(a)), \\ &\operatorname{res}_{P_2}(S.\mathrm{push} \to \mathrm{ok}), \\ &\operatorname{inv}_{P_1}(
S.\mathrm{pop}()), \\ &\operatorname{res}_{P_1}(S.\mathrm{pop} \to a)
\,\rangle . \end{aligned} \]

So we may think of distributed system as a tuple of processes, events, operations, objects and history of the whole
system. Having such a description we may define some properties of such a system in terms of these component objects.
Shan't we?

## Various types of orderings imposed by distributed system

We assume we have some history \(\Sigma\) of events \(e_1,e_2,\dots\). I'll also assume our history is *complete*, that
is every event is uniquely associated to some operation, which means that there is no invocation which was not
returned (and vice versa!). That assumption may be relaxed by lifting the history in question to its *completion* - but
I think it might overcomplicate things.

### Ordering of events

If \(\Sigma=e_1,e_2,\dots\) then we define \[e_i <_\Sigma^{ev} e_j \iff i < j\]

That is, left to right order of symbols in \(\Sigma\).

Ordering events is quite easy. The forecoming orders are related for operations. So formally we're defining these orders
on the set of operations (which formally may be realized as a subset of product \(\Sigma \times \Sigma\)).

### Real-time order

For operations (which are completed by our assumption on history) \(a\) and \(b\) we define *real time ordering*
\[a <_\Sigma^{rt} b \iff \operatorname{res}(a) <_\Sigma^{ev} \operatorname{inv}(b)\]

Hence it means that the response of event \(a\) appears before the invocation of event \(b\) in \(\Sigma\). Note that it
is not important which process performed invocation or response.

### Process order

For operations \(a\) and \(b\) we define *process ordering* \[a <_\Sigma^{process} b \iff \operatorname{proc}(a)
=\operatorname{proc}(b)
\;\land\; \operatorname{res}(a) <^{\mathrm{ev}}_\Sigma \operatorname{inv}(b). \]

**Remark.** Note that the real-time and process order are only partial orderings in general. For example given
\[\Sigma=\langle \operatorname{inv}_{P_i}(a)
,\operatorname{inv}_{P_j}(b),\dots \rangle\] Then operations \(a\) and \(b\) are not comparable in real-time order. If
\(i \neq j\) then they are also not comparable in the process order.

## Sequential consistency and linearizability

### Sequential history

We say a history \(S\) is *sequential* if operations in it does not overlap. Namely it means every operation completes
before the next operation begins \[ \operatorname{inv}(op_1), \operatorname{res}(op_1), \operatorname{inv}(op_2),
\dots \]

**Remark.** Note that for sequential history \(S\), the induced real-time order \(<_S^{rt}\) is *total order*.

### Legal sequential history

A sequential history \(S\) is *legal*, if for every object \(X\), we have \(S|X \in \operatorname{SeqSpec}(X)\).

Here is an example of legal sequential history \[\operatorname{inv}(\operatorname{write}(x,1)),\operatorname{res}(
\operatorname{write}(x,1)),\operatorname{inv}(\operatorname{read}(
x)),\operatorname{res}(\operatorname{read}(x)→1). \]

On the other hand, this sequential history is not legal \[\operatorname{inv}(\operatorname{write}(x,1))
,\operatorname{res}(
\operatorname{write}(x,1)),\operatorname{inv}(\operatorname{read}(
x)),\operatorname{res}(\operatorname{read}(x)→0). \]

because after writing 1, a later read should return 1, not 0.

### Sequential consistency

A history \(\Sigma\) is *sequentially consistent* iff there exists a legal sequential history \(S\) such that \[ <_
\Sigma^{proc} \subseteq <_S^{rt}\] that is, for every two operations \(a\) and \(b\), if \(a <_\Sigma^{proc} b\), then
\(a <_S^{rt} b\).

Sequential consistency says that the execution must be explainable as some single-threaded execution that preserves the
order of operations made by each individual process. It does not require that this explanation respect the actual
wall-clock order between operations from different processes.

### Linearizability

A history \(\Sigma\) is *linearizable* iff there exists a legal sequential history \(S\) such that \[ <_\Sigma^{rt}
\subseteq <_S^{rt}\] that is, for every two operations \(a\) and \(b\), if \(a <_\Sigma^{rt} b\), then \(a <_S^{rt} b\).

Linearizability says that the execution must be explainable as some single-threaded execution that also respects
real-time order: if one operation finishes before another starts, they must appear in that order. Equivalently, each
operation appears to take effect atomically at some point between its invocation and response (we'll come back to that
in [Linearization points](#linearization-points) section below).

### Example of sequential history which is not linearizable

I think what we need at this point is a simple example to clarify the above definitions. Navigating the maze of abstract
concepts, whilst very enjoyable and rewarding, only makes sense when accompanied by a good example. So let’s not build
an ivory tower; let’s move on to a minimal example of a system that is sequentially consistent but not linearizable.

In our example we consider only one register (object) \(x\) with its initial value being \(0\). Lets consider history \[
\Sigma = \langle \operatorname{inv}_{P_1}(\operatorname{write}(x, 1)), \operatorname{res}_{P_1}(\operatorname{write}(x,
1)), \operatorname{inv}_{P_2}(\operatorname{read}(x)), \operatorname{res}_{P_2}(\operatorname{read}(x) \rightarrow 0))
\rangle\]

Let us denote operations in \(\Sigma\) to be \[ w := \langle \operatorname{inv}_{P_1}(\operatorname{write}(x, 1)),
\operatorname{res}_{P_1}(\operatorname{write}(x, 1)) \rangle \qquad r := \langle \operatorname{inv}_{P_2}(
\operatorname{read}(x)), \operatorname{res}_{P_2}(\operatorname{read}(x) \rightarrow 0))
\rangle\]

Hence, by the very definition of real-time order, we have \(w <_\Sigma^{rt} r\). Also note that since each process has
only one operation, it follows that the process-order is actually empty: \[<_\Sigma^{proc} = \emptyset \] In particular
we have \[ r \not <_\Sigma^{proc} w \qquad \text{and} \qquad w \not <_\Sigma^{proc} r\]

**Proving \(\Sigma\) is sequentially consistent**. Sequential consistency requires that there exists a legal sequential
history \(S\) such that: \[ <_\Sigma^{proc} \subseteq <_S^{rt}\] and \(S\) respects the sequential specification of the
register \(x\). Choose \[ S := \langle r, w \rangle \] That is \(r <_S^{rt} w\). Now let us check that aforementioned
conditions. Since empty set is contained in every other set, we have indeed \[ \emptyset = <_
\Sigma^{proc} \subseteq <_S^{rt} \] So it is left to check \(S\) is legal according to \(\operatorname{SeqSpec}(x)\).
The register is initially equal to \(0\) and in \(S\) the read happens before the write. So when \(r\) executes, no
write to \(x\) has happened yet. Therefore the most recent value of \(x\) is still initial value and it shows that
\(S|x\) is an element of \(\operatorname{SeqSpec}(x)\).

**Proving \(\Sigma\) is not linearizable**. Linearizability requires that there exists a legal sequential history \(S'\)
such that \[ <_\Sigma^{rt} \subseteq <_{S'}^{rt} \]

From the actual history, we have \(w <_\Sigma^{rt} w\), therefore any linearization \(S'\) must satisfy \(w <_{S'}^{rt}
r\). Since there are only two operations, the only possible sequential order satisfying it is \[ S' := \langle w,r
\rangle \] It is obvious that \(S'|x \not \in \operatorname{SeqSpec}(x)\) so \(S'\) is not legal, hence \(S'\) cannot be
a linearization of \(\Sigma\).

### Locality of linearizability

One of the most important properties of linearizability is that it is a *local property*. Suppose a history \(\Sigma\)
contains operations on finitely many objects \(X_1,X_2,\dots, X_N\). Then

\[ \Sigma \text{ is linearizable} \iff \forall X_i \space \Sigma|_{X_i} \text{ is linearizable}\]

This is important because it makes linearizability modular. For example, if a system has a linearizable register \(X\)
and a linearizable stack \(S\), then the combined system using both \(X\) and \(S\) is linearizable as a whole.

### Linearization points

There is a well-known technique of baking legal sequential history \(S\) to prove the system is linearizable, which in
fact conveys to us an intuition why we call such a system "linearizable". The idea is to take existing ordering
\(\Sigma\) of operations and define *linearization points* by using this ordering. Then, having these points, we cook
another ordering of operations from \(\operatorname{Ops}(\Sigma)\) which *may happen to be* linearization of \(\Sigma\).
Let me jump into formalities for a moment.

#### Step 1. Introducting linearization points into existing system

Let us consider history \(\Sigma\). For every operation \(\operatorname{op} \in \operatorname{Ops}(\Sigma)\) let us
choose a (kinda formal) point \(t_{\operatorname{op}}\) satisfying \[\operatorname{inv}(\operatorname{op}) \leq_
\Sigma^{ev} t_
{\operatorname{op}} \leq_\Sigma^{ev} \operatorname{res}(\operatorname{op})\] where \(\leq_\Sigma^{ev}\) is an event
order generated by \(\Sigma\) [defined above](#ordering-of-events).

**Mathematical remark.** To be completely precise, to add points we mean to expand our set \(\Sigma\) by adding formal
points \[ \widetilde{\Sigma} := \Sigma \cup \{ t_{op} | op \in \operatorname{Ops}(\Sigma)\}\] and then expanding the
given event order \(\leq_\Sigma^{ev}\) to \(\leq_{\widetilde{\Sigma}}^{ev}\). It is just fancy way of saying that, we
just need to order events (old events from \(\Sigma\) and new formal points) in \(\widetilde{\Sigma}\) with the
constraints being the aforementioned inequalities on formal points.

#### Step 2. Ordering of operations by linearization points

For two operations \(a,b \in \operatorname{Ops}(\Sigma)\) let us define new order \(\leq_S\) by putting \(a \leq_S b\)
whenever \(t_a \leq t_b\). Having such an ordering of *operations* we may *smear it down* into ordering of *events* in a
trivial way (do you see how?) and we get order \(S=\langle S, \leq_S \rangle\) of events. By the very construction, we
see that \(S\) is sequential history. The question now is if such a produced ordering \(S\) is legal history?

#### Step 3. Checking that generated sequential history is legal

The last step is to forget about linearization points and just check that \(S\) is legal history for \(\Sigma\).
Business as usual, they say.

#### Intuition

So what did we discover? The intuition is linearization points explain why the name “linearizability” makes sense. A
concurrent history caughted in the wild is probably far from being linear: operations may overlap in time.
Linearizability says that, despite this overlap, we can *choose one abstract point inside each operation interval and
then order operations by these points*.

Thus, a linearization point is the moment where an operation is treated as *taking effect atomically*. Once every
operation has such a point, the concurrent execution becomes a single line of operations:
\[ \operatorname{op}_1,\operatorname{op}_2,\dots, \operatorname{op}_n\] That line is the “linearized” view of the
execution.

#### Example

Consider the following system

![Maelstrom](/images/linearizability.drawio.svg)

Then we have \[\Sigma=\langle \operatorname{inv}_{P_1}(\operatorname{op_1}), \operatorname{inv}_{P_2}(
\operatorname{op_2}), \operatorname{res}_{P_1}(\operatorname{op_1}), \operatorname{res}_{P_2}(\operatorname{op_2}),
\operatorname{inv}_{P_3}(\operatorname{op_3}), \operatorname{res}_{P_3}(\operatorname{op_3})
\rangle \]

We see this history is not even sequential one. This choice of linearization points \(t_2 \leq_S t_1 \leq_S t_3\) yields
ordering of operations \[\operatorname{op}_2 \leq_S \operatorname{op}_1 \leq_S
\operatorname{op}_3 \] And finally smearing it down we obtain abstract re-ordering of \(\Sigma\) \[S=\langle
\operatorname{inv}_{P_2}(\operatorname{op_2}), \operatorname{res}_{P_2}(\operatorname{op_2}), \operatorname{inv}_{P_1}(
\operatorname{op_1}), \operatorname{res}_{P_1}(\operatorname{op_1}), \operatorname{inv}_{P_3}(\operatorname{op_3}),
\operatorname{res}_{P_3}(\operatorname{op_3}) \rangle \] As promised \( \langle S, \leq_S \rangle\) is being sequential.

**Remark**. At such level of generality we of course cannot prove it is legal history. In this example I wanted to show
you just how the choice of linearization points induces re-ordering of events and produces sequential history which is a
*candidate*
for linearization. The natural question is now: is there some natural choice for linearization point inside operation
completion inteval? The next section may serve as an example of such a choice for some systems.

## Compare and Swap (CAS)

There is one piece we still need in order to connect the abstract definition of linearizability with real
implementations: an atomic update primitive.
**Compare-And-Swap** usually written as \[ \operatorname{CAS}(x, \operatorname{expected}, \operatorname{new}) \] is an
atomic operation on shared memory location \(x\).

More explicitly, CAS performs the following logic atomically:

```python
if x == expected:
    x = new
    return success
else:
    return failure
```

The crucial word is *atomically*. No other process can observe CAS halfway through. It either sees the old value or the
new value, but not some intermediate state.

CAS is closely related to hardware support for atomic read-modify-write instructions. Modern CPUs usually provide
instructions that can implement CAS or similar primitives. For example, x86 provides compare-and-exchange instructions,
commonly known as `CMPXCHG`. The implementation (on x86) relies on the fact that the expected value is stored in a
special accumulator register: \[ \operatorname{AL}, \operatorname{AX}, \operatorname{EAX},\text{ or}
\operatorname{RAX}\] depending on the operand size.

### Relation of CAS instruction to linearization points

CAS naturally gives linearization points because a successful CAS is the exact instant when a shared state change
becomes visible.

For example, in a lock-free stack, a push operation may first allocate and prepare a new node privately. None of that
changes the abstract stack yet. The operation takes effect only when it successfully executes:

\[ \operatorname{CAS}(\operatorname{stack[top], \operatorname{oldTop}, \operatorname{newTop}})\]

At that moment, the top pointer changes from \(\operatorname{oldTop}\) to \(\operatorname{newTop}\), so the new element
becomes visible as part of the stack. Therefore the successful CAS is the natural linearization point of the push.

Similarly, a pop operation often linearizes at the successful CAS that changes the top of the stack from the removed
node to the next node.

## Industry standards

I have prepared table consisting of five commonly used distributed systems with their guarantees and links to relevant
documentation.

| System              | Vendor / common guarantee                                             | Compact meaning                                                                                                                                                                                                              |
|---------------------|-----------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Amazon S3**       | **Strong read-after-write consistency**                               | After a successful object write, later `GET` and `LIST` requests observe the new state. This was not always true historically; S3 [changed this][6] guarantee in 2020. ([Amazon Web Services, Inc.][1])                      |
| **DynamoDB**        | **Eventual reads by default; strongly consistent reads optionally**   | A normal read may briefly return stale data. A strongly consistent read returns the most up-to-date data, but only for tables and local secondary indexes, not global secondary indexes or streams. ([AWS Documentation][2]) |
| **Google Spanner**  | **External consistency / strict serializability**                     | Transactions behave as if they execute in a serial order that respects real-time commit order. This is a transaction-level analogue of linearizability plus serializability. ([Google Cloud Documentation][3])               |
| **etcd**            | **Linearizable operations by default; serializable reads optionally** | Default reads reflect the current consensus state. For performance, etcd also offers “serializable” reads that can be served locally and may be stale. ([etcd][4])                                                           |
| **Azure Cosmos DB** | **Tunable consistency levels**                                        | The application chooses among levels such as Strong, Bounded Staleness, Session, Consistent Prefix, and Eventual, trading freshness for latency/availability. ([Microsoft Learn][5])                                         |

[1]: https://aws.amazon.com/s3/consistency "Amazon S3 Strong Consistency"

[2]: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html "DynamoDB read consistency"

[3]: https://docs.cloud.google.com/spanner/docs/true-time-external-consistency "Spanner: TrueTime and external consistency"

[4]: https://etcd.io/docs/v3.5/learning/api_guarantees "etcd API guarantees"

[5]: https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels "Consistency level choices - Azure Cosmos DB"

[6]: https://aws.amazon.com/blogs/aws/amazon-s3-update-strong-read-after-write-consistency/

## Links

To dig deeper into this hole I encourage you to read Lessie Lamport's [[Lam79]] as a old good introduction to
distributed systems theory. He touches more deeply the topic events ordering in distributed system. In particular he
introduces logical clocks and vector clocks which are used to these days in systems like concurrent document editions.
There is also Herlihy-Wing paper [[HW90]] on linearizability which is very readable and formal. If you want to get your
hands dirty with formal evidence and technical arguments, go for it.

That's all from me for today. See you around!

[Lam79]: https://lamport.azurewebsites.net/pubs/time-clocks.pdf

[HW90]: https://cs.brown.edu/people/mph/HerlihyW90/p463-herlihy.pdf