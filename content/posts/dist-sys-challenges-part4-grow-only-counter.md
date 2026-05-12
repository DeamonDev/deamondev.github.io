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
Since sequential-consistency is closely related to a crucial guarantee which is *linearizability* I'll define that one
as well and provide a minimal example showing the difference. Finally, I'll describe *Compare and Swap* – an atomic CPU
instruction used in multithreaded systems to achieve synchronization.

But before even talking about consistency guarantees, I think we should define what we mean by a distributed system.

### What is a distributed system?

### Sequential consistency and linearizability

### Compare and Swap (CAS)

## Setup

### Makefile

## Code

## Summary