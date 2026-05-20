+++
date = '2026-05-12T22:04:26+02:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: grow only counter (part 4) with introduction to sequential consistency and linearizability'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems', 'sequential-consistency', 'linearizability', 'fault tolerance']
toc = true
[params]
math = true
+++

## Grow Only Counter Challenge

...

## Theoretical background

While solving this challenge, we'll need to use a sequentially consistent key value store provided by maelstrom (this is
demanded by the workload). As long as I am convinced that the general reader is aware of what the key-value store is, I
am not quite sure about the adjective part. In this section I plan to define (formally) what sequential consistency is.
Since sequential-consistency is closely related to another crucial guarantee which is *linearizability* I'll define that
one as well and provide a minimal example showing the difference. Finally, I'll describe *Compare and Swap* – an atomic
CPU instruction used in multithreaded systems to achieve synchronization.

But before even talking about consistency guarantees, I think we should define what we mean by a distributed system.

### What is a distributed system?

I would like to start by saying that the level of formality used to answer the question of what a distributed system is
should be appropriately calibrated. There are many excellent works and publications on the subject, which are very good
sources of information on the topic. Links to the most important publications are provided in the 'Links' section below.

Nevertheless, I believe it is important to maintain an appropriate level of formality, especially if we want to
understand exactly what the guarantees of concurrency are. Let's start with some simple definitions.

#### Processes, objects and operations

A *process* is n execution thread of control that invokes *operations* on *shared objects*. We denote them by
\[P_1,P_2,\dots\] An *object* is shared data structure accessed by processes. Examples being \[\operatorname{register},
\operatorname{queue}, \operatorname{stack}, \operatorname{counter}\] Each object \(X\) has *sequential specification*,
\(\operatorname{SeqSpec}(X)\) which is a set of legal sequential histories over a given object. By *history* we mean
**sequence** of *events*.

An *operation* is a single method call performed by a process on an object. Formally it is pair of events: \[op=\langle
\operatorname{inv}(op), \operatorname{res}(op)\rangle\]

So each event is either invocation of response of an operation.

Here is example of legal sequential history for stack: \[ \begin{aligned} \Sigma = \langle\, &\operatorname{inv}(
\mathrm{push}(a)), \\ &\operatorname{res}(\mathrm{push}(a) \to \mathrm{ok}), \\ &\operatorname{inv}(\mathrm{push}(
b)), \\ &\operatorname{res}(
\mathrm{push}(b) \to \mathrm{ok}), \\ &\operatorname{inv}(\mathrm{pop}()), \\ &\operatorname{res}(\mathrm{pop}() \to
b), \\ &\operatorname{inv}(\mathrm{pop}()), \\ &\operatorname{res}(\mathrm{pop}() \to a)
\,\rangle \end{aligned} \] It is not hard to see that \(\Sigma\) __is an element of__ \(\operatorname{SeqSpec}(
\operatorname{stack})\). Formally we should denote which process performed which invocation/response - but when thinking
at the level of sequential history for given object, we should narrow our mind to local (synchronous) reasoning.

Having formally defined all of these, we may give informal definition of distributed system as a *collection of
independent processes that communicate indirectly throught shared objects or directly throught messages*

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

So we may think of distributed system as a tuple of processes, events (operations), objects and history of the whole
system. Having such a description we may define some properties of such a system in terms of these component objects.
Shan't we?

### Various types of orderings imposed by distributed system

We assume we have some history \(\Sigma\) of events \(e_1,e_2,\dots\). I'll also assume our history is *complete*, that
is every event is an operation, which means there is no invocation which was not returned. That assumption may be
relaxed, by lifting to *completion of history* – but I think it might overcomplicate things.

#### Ordering of events

If \(\Sigma=e_1,e_2,\dots\) then we define \[e_i <_\Sigma^{ev} e_j \iff i < j\]

That is, left to right order of symbols in \(\Sigma\).

Ordering events is quite easy. The forecoming orders are related for operations. So formally we're defining these orders
on the set of operations (which formally may be realized as a subset of product \(\Sigma \times \Sigma\)).

#### Real-time order

For operations (which are completed by our assumption on history) \(a\) and \(b\) we define *real time ordering*
\[a <_\Sigma^{rt} b \iff \operatorname{res}(a) <_\Sigma^{ev} \operatorname{inv}(b)\]

Hence it means that the response of event \(a\) appears before the invocation of event \(b\) in \(\Sigma\). Note that it
is not important which process performed invocation or response.

#### Program order

For operations \(a\) and \(b\) we define *process ordering* \[a <_\Sigma^{process} b \iff \operatorname{proc}(a)
=\operatorname{proc}(b)
\;\land\; \operatorname{res}(a) <^{\mathrm{ev}}_\Sigma \operatorname{inv}(b). \]

**Remark.** Note that the real-time and program order are only partial orderings in general. For example given
\[\Sigma=\langle \operatorname{inv}_{P_i}(a)
,\operatorname{inv}_{P_j}(b),\dots \rangle\] Then operations \(a\) and \(b\) are not comparable in real-time order. If
\(i \neq j\) then they are also not comparable in the process order.

### Sequential consistency and linearizability

#### Sequential history

We say a history \(S\) is *sequential* if operations in it does not overlap. Namely it means every operation completes
before the next operation begins \[ \operatorname{inv}(op_1), \operatorname{res}(op_1), \operatorname{inv}(op_2),
\dots \]

**Remark.** Note that for sequential history \(S\), the induced real-time order \(<_S^{rt}\) is *total order*.

#### Legal sequential history

A sequential history \(S\) is *legal*, if for every object \(X\), we have \(S|X \in \operatorname{SeqSpec}(X)\).

Here is an example of legal sequential history \[\operatorname{inv}(\operatorname{write}(x,1)),\operatorname{res}(
\operatorname{write}(x,1)),\operatorname{inv}(\operatorname{read}(
x)),\operatorname{res}(\operatorname{read}(x)→1). \]

On the other hand, this sequential history is not legal \[\operatorname{inv}(\operatorname{write}(x,1))
,\operatorname{res}(
\operatorname{write}(x,1)),\operatorname{inv}(\operatorname{read}(
x)),\operatorname{res}(\operatorname{read}(x)→0). \]

because after writing 1, a later read should return 1, not 0.

#### Sequential consistency

A history \(\Sigma\) is *sequentially consistent* iff there exists a legal sequential history \(S\) such that \[ <_
\Sigma^{program} \subseteq <_S^{rt}\] that is, for every two operations \(a\) and \(b\), if \(a <_\Sigma^{program} b\),
then \(a <_S^{rt} b\).

Sequential consistency says that the execution must be explainable as some single-threaded execution that preserves the
order of operations made by each individual process. It does not require that this explanation respect the actual
wall-clock order between operations from different processes.

#### Linearizability

A history \(\Sigma\) is *linearizable* iff there exists a legal sequential history \(S\) such that \[ <_\Sigma^{rt}
\subseteq <_S^{rt}\] that is, for every two operations \(a\) and \(b\), if \(a <_\Sigma^{rt} b\), then \(a <_S^{rt} b\).

Linearizability says that the execution must be explainable as some single-threaded execution that also respects
real-time order: if one operation finishes before another starts, they must appear in that order. Equivalently, each
operation appears to take effect atomically at some point between its invocation and response (we'll come back to that
in [Linearization points](#linearization-points) section below).

#### Example of sequential history which is not linearizable

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
only one operation, it follows that the program-order is actually empty: \[<_\Sigma^{program} = \emptyset \] In
particular we have \[ r \not <_\Sigma^{program} w \qquad \text{and} \qquad w \not <_\Sigma^{program} r\]

**Proving \(\Sigma\) is sequentially consistent**. Sequential consistency requires that there exists a legal sequential
history \(S\) such that: \[ <_\Sigma^{program} \subseteq <_S^{rt}\] and \(S\) respects the sequential specification of
the register \(x\). Choose \[ S := \langle r, w \rangle \] That is \(r <_S^{rt} w\). Now let us check that
aforementioned conditions. Since empty set is contained in every other set, we have indeed \[ \emptyset = <_
\Sigma^{program} \subseteq <_S^{rt} \] So it is left to check \(S\) is legal according to \(\operatorname{SeqSpec}(x)\).
The register is initially equal to \(0\) and in \(S\) the read happens before the write. So when \(r\) executes, no
write to \(x\) has happened yet. Therefore the most recent value of \(x\) is still initial value and it shows that
\(S|x\) is an element of \(\operatorname{SeqSpec}(x)\).

**Proving \(\Sigma\) is not linearizable**. Linearizability requires that there exists a legal sequential history \(S'\)
such that \[ <_\Sigma^{rt} \subseteq <_{S'}^{rt} \]

From the actual history, we have \(w <_\Sigma^{rt} w\), therefore any linearization \(S'\) must satisfy \(w <_{S'}^{rt}
r\). Since there are only two operations, the only possible sequential order satisfying it is \[ S' := \langle w,r
\rangle \] It is obvious that \(S'|x \not \in \operatorname{SeqSpec}(x)\) so \(S'\) is not legal, hence \(S'\) cannot be
a linearization of \(\Sigma\).

##### Locality of linearizability

##### Linearization points

### Compare and Swap (CAS)

### Links

## Setup

### Makefile

## Code

## Summary