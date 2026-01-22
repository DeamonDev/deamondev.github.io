+++
date = '2026-01-22T21:17:29+01:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: efficient broadcast 2 (part 3e)'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems', 'broadcast', 'fault tolerance']
toc = true
++++++

## Efficient Broadcast Challenge (Part II)

## Setup

Run these commands to bootstrap this part:

```shell
❯ mkdir broadcast-3e
broadcast-3e❯ go mod init github.com/deamondev/gossip-glomers-tutorial/broadcast-3e
❯ go work use ./broadcast-3e
```

### Makefile
At this point you only should set `MODULE` to be `broadcast-3e` and `WORKLOAD` parameter to be `broadcast-3e` (see previous part
for the details).

## Code

### broadcast-3e/batcher.go

### broadcast-3e/server.go

## Running workload

Let us see if messages propagate in the cluster properly:
```sh
```


```clojure
```

## Summary
