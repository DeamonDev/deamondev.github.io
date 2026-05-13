+++
date = '2026-05-12T22:04:26+02:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: grow only counter (part 4)'
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
\,\rangle \end{aligned} \] is an element of \(\operatorname{SeqSpec}(\operatorname{stack})\). Formally we should denote
which process performed which invocation/response - but when thinking at the level of sequential history for given
object, we should narrow our mind to local (synchronous) reasoning.

Having formally defined all of these, we may give informal definition of distributed system as a *collection of
independent processes that communicate indirectly throught shared objects or directly throught messages*

Besides the sequential histories of the local objects, we consider the history of the whole system. For example, suppose
the system has two shared objects \[ X = \text{register}, \qquad S = \text{stack}. \] A whole system history could be:
\[ \begin{aligned} H = \langle\, &\operatorname{inv}_{P_1}(X.\mathrm{write}(1)), \\ &\operatorname{res}_{P_1}(
X.\mathrm{write} \to \mathrm{ok}), \\ &\operatorname{inv}_{P_2}(S.\mathrm{push}(a)), \\ &\operatorname{res}_{P_2}(
S.\mathrm{push} \to \mathrm{ok}), \\ &\operatorname{inv}_{P_1}(S.\mathrm{pop}()), \\ &\operatorname{res}_{P_1}(
S.\mathrm{pop} \to a)
\,\rangle . \end{aligned} \] This is history of the whole execution, because it includes operation on both \(X\) and
\(S\). We can then localize this history to \(S\) obtaining \[ \begin{aligned} H \mid S = \langle\, &\operatorname{inv}
_{P_2}(S.\mathrm{push}(a)), \\ &\operatorname{res}_{P_2}(S.\mathrm{push} \to \mathrm{ok}), \\ &\operatorname{inv}_{P_1}(
S.\mathrm{pop}()), \\ &\operatorname{res}_{P_1}(S.\mathrm{pop} \to a)
\,\rangle . \end{aligned} \]

So we may think of distributed system as a tuple of processes, events (operations), objects and history of the whole
system. Having such a description we may define some properties of such a system in terms of these component objects.
Shan't we?

### Sequential consistency and linearizability

### Compare and Swap (CAS)

## Setup

### Makefile

## Code

## Summary