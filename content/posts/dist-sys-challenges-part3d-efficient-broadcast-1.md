+++
date = '2025-12-29T17:45:24+01:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: efficient broadcast 1 (part 3d)'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems', 'broadcast', 'fault tolerance']
toc = true
+++

## Efficient Broadcast Challenge

Things are getting interesting! This is the first part of the efficiency challenge, which builds on the fault-tolerant broadcast
challenge. The workload is becoming more rigorous:

* node count is increased to `25`
* there is `100ms` delay to each message to simulate slow network

Our challenge is to achieve:

* `msgs-per-op` should be below `30`
* `median latency` should be below `400ms`
* `maximum latency` should be below `600ms`

**Remark**. We should still ignore the topology proposed by maelstrom, since it is just 2 dimensional grid of nodes. Such
a topology duplicates a lot of messages and add latencies of order `2*sqrt(n)`. In fact, we may simply remove some connections
from such a grid to improve its characteristics. I will investigate this further in the corresponding topology section of this article.

## Setup

Run these commands to bootstrap this part:

```shell
❯ mkdir broadcast-3d
broadcast-3d❯ go mod init github.com/deamondev/gossip-glomers-tutorial/broadcast-3d
❯ go work use ./broadcast-3d
```

### Makefile
Let's calibrate `MODULE` to `broadcast-3d` and `WORKLOAD` parameter to `broadcast-3de` (since we'll reuse it in last part).
Our new maelstrom command should be:

```shell
MAELSTROM_CMD_broadcast-3de = maelstrom/maelstrom test -w broadcast --bin $(BINARY) --node-count 25 --time-limit 20 --rate 100 --latency 100
```

## Code

That time we create new file with our custom topology:

### broadcast-3d/topology.go

```go
package main

var topology = map[string][]string{
	"n0":  {},
	"n1":  {"n0"},
	"n2":  {"n1", "n3"},
	"n3":  {"n4"},
	"n4":  {},
	"n5":  {},
	"n6":  {"n5"},
	"n7":  {"n6", "n2", "n8"},
	"n8":  {"n9"},
	"n9":  {},
	"n10": {},
	"n11": {"n10"},
	"n12": {"n7", "n11", "n13", "n17"},
	"n13": {"n14"},
	"n14": {},
	"n15": {},
	"n16": {"n15"},
	"n17": {"n16", "n18", "n22"},
	"n18": {"n19"},
	"n19": {},
	"n20": {},
	"n21": {"n20"},
	"n22": {"n21", "n23"},
	"n23": {"n24"},
	"n24": {},
}

// It is hardcoded, in a real system it should be dynamic
var masterNode = "n12"
```

The idea behind that is we simply remove all the vertical arrows from full 2d grid topology. Thanks to central place of `n12` node, 
by routing through it we see that we may reach any node from any node in at most `5` steps. For example, reaching `n4` from `n21` is just
sequence of steps: `n21-->n12-->n7-->n2-->n3-->n4`. This is true in general, since any node from `n12` is reachable within at most `4` steps.
We declare `n12` to be *master* node, altough maybe better name might be *central* node?

![Maelstrom](/images/broadcast3d-network.drawio.svg)



### broadcast-3d/server.go

```go

```

## Running workload

Let us see if...
```sh
❯ make run
go build -o ~/go/bin/maelstrom-broadcast-3d ./broadcast-3d
```


```clojure
...

```

## Summary
