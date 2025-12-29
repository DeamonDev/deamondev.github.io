+++
date = '2025-12-29T17:45:24+01:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: efficient broadcast 1 (part 3d)'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems', 'broadcast', 'fault tolerance']
toc = true
+++

## Efficient Broadcast Challenge

Things are 

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
