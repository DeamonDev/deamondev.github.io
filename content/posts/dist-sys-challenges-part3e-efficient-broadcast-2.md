+++
date = '2026-01-22T21:17:29+01:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: efficient broadcast 2 (part 3e)'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems', 'broadcast', 'fault tolerance']
toc = true
+++

## Efficient Broadcast Challenge (Part II)

Here we are - the last part of the whole broadcast series. This is second part of the efficiency challenge, which builds
on the previous part. With the same node count of `25` and message delay of `100ms`, the challenge is to achieve
the following performance metrics:

* `msgs-per-op` should be below `20`
* `median latency` should be below `1` second
* `maximum latency` should be below `2` seconds

In other words, we handled median and maximum latencies for lower messages per op during execution of the same workflow.
Let me think for a moment...Since I can sacrifice latency in exchange for sending fewer messages, maybe it would be a good
idea to batch messages before sending them and send them in the aforementioned batches?

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

As I mentioned above, the whole idea is to add batching mechanism to our node. For doing so I'll define new component which
I call *batcher*. Its whole role is just batching messages sent by our node to another nodes. The batcher has embedded ticker
which ticks every configured time duration. After tick, the batched messages are transformed into `FlushEvent`'s which are then
handled by the node internal handler.

### broadcast-3e/batcher.go

```go
package main

import (
	"log"
	"sync"
	"time"
)

type Batcher struct { ⓿
	mu        sync.Mutex
	batches   map[string][]int
	ticker    *time.Ticker
	flushChan chan FlushEvent
}

type FlushEvent struct { ❶
	PeerID   string
	Messages []int
}

func NewBatcher(batchTimeout time.Duration) *Batcher { ❷
	return &Batcher{
		ticker:    time.NewTicker(batchTimeout),
		batches:   make(map[string][]int),
		flushChan: make(chan FlushEvent),
	}
}

func (b *Batcher) Run() {
	for range b.ticker.C {
		b.mu.Lock()
		for peerID, messages := range b.batches {
			if len(messages) > 0 { ❸
				b.flushChan <- FlushEvent{
					PeerID:   peerID,
					Messages: messages,
				}
			}
		}
		b.batches = make(map[string][]int) ❹
		b.mu.Unlock()
	}
}

func (b *Batcher) Add(peerID string, message int) {
	b.mu.Lock()
	b.batches[peerID] = append(b.batches[peerID], message) ❺
	b.mu.Unlock()
}

func (b *Batcher) Close() { ❻
	log.Printf("Closing batcher")

	b.ticker.Stop()
	close(b.flushChan)
}
```

The `Batcher` struct holds internal map of batched messages, where we batch array of messages (thst is, numbers)
per each peer id of the node in question. There is also `ticker` and channel of `FlushEvent`'s which is channel of 
communication of the batcher with the external node (in which the batcher is embedded on).

The `FlushEvent` contains only info about the id of the peer and the batched messages. It is just plain data, which
is meant to be *interpreted* somehow in consumers of the `flushChan`  channel. Such a separation of concerns is solid
design choice, since it is easier to reason and test the code.

In the constructor function we just configure the `batchTimeout` - the intuition is that when increasing this timeout, 
we minimize the `msgs-per-op` cluster characteristic and increase overall latency of the system (and vice versa). I'll
summarize the experiments at the very end of this article.

Below is a diagram illustrating the described system:

![Maelstrom](/images/broadcast3e-batcher.drawio(3).svg)


### broadcast-3e/server.go

## Running workload

Let us see if messages propagate in the cluster properly:
```sh
```


```clojure
```

## Summary
