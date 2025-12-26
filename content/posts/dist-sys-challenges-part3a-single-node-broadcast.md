+++
date = '2025-12-18T09:22:19+01:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: single node broadcast (part 3a)'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems', 'broadcast']
toc = true
+++

## Single Node Broadcast Challenge

This is the very first challenge related to the *broadcast* system we need to implement. The idea is that our glomers (nodes) 
receive messages they need to propagate to another gossips. Each node stores messages it already seen and should be able
to return them if RPC `read` call was performed by some controller node. In this part we need to run one-node broadcast
system, so the only messages will be performed from controller nodes to our single node. In fact, there is no demand 
to perform any actual broadcasting at all, since the only specification which will be checked is that the `read` RPC
calls return an expected list of messages. The messages sent from the controller are unique (per node). 

There are two important metrics that we'll need to tune at later stages of the broadcast challenge: 

- `:net -> :msgs-per-op`
- `:workload -> :stable-latencies`

The first value measure shows the number of messages exchanged per logical operation. The second value are quantiles which show the broadcast 
latency for the minimum, median, 95th, 99th, and maximum latency request. These latencies are measured from the time a
broadcast request was acknowledged to when it was last missing from a read on any node. For example, here’s a system whose 
median latency was 452 milliseconds:

```clojure
:stable-latencies {0 0,
                   0.5 452,
                   0.95 674,
                   0.99 731,
                   1 794},
```
## Setup

Run these commands to bootstrap this part:

```shell
❯ mkdir broadcast-3a
broadcast-3a❯ go mod init github.com/deamondev/gossip-glomers-tutorial/broadcast-3a
❯ go work use ./broadcast-3a
```

### Makefile
Let's calibrate `MODULE` and `WORKLOAD` parameters to be `broadcast-3a`. Our new maelstrom command should be:

```shell
MAELSTROM_CMD_broadcast-3a = maelstrom/maelstrom test -w broadcast --bin $(BINARY) --node-count 1 --time-limit 20 --rate 10
```

## Code

### broadcast-3a/server.go

```go
package main

import (
	"encoding/json"
	"log"
	"sync"

	maelstrom "github.com/jepsen-io/maelstrom/demo/go"
)

type Server struct {
	node   *maelstrom.Node
	nodeID string

	mu       sync.Mutex 🄌
	messages map[int]struct{}
}

type BroadcastMessage struct {
	Type    string `json:"type"`
	Message int    `json:"message"`
}

type BroadcastMessageResponse struct {
	Type string `json:"type"`
}

type ReadMessage struct {
	Type string `json:"type"`
}

type TopologyMessage struct {
	Type     string              `json:"type"`
	Topology map[string][]string `json:"topology"`
}

type TopologyMessageResponse struct {
	Type string `json:"type"`
}

type ReadMessageResponse struct {
	Type     string `json:"type"`
	Messages []int  `json:"messages"`
}

func NewServer(n *maelstrom.Node) *Server {
	s := &Server{node: n, messages: make(map[int]struct{})}

	s.node.Handle("init", s.initHandler)
	s.node.Handle("broadcast", s.broadcastHandler)
	s.node.Handle("read", s.readHandler)
	s.node.Handle("topology", s.topologyHandler)

	return s
}

func (s *Server) initHandler(msg maelstrom.Message) error {
	var body maelstrom.InitMessageBody
	if err := json.Unmarshal(msg.Body, &body); err != nil {
		return err
	}

	s.nodeID = body.NodeID

	log.Printf("Node id set to: %s", s.nodeID)

	return nil
}

func (s *Server) broadcastHandler(msg maelstrom.Message) error { ➊
	var body BroadcastMessage
	if err := json.Unmarshal(msg.Body, &body); err != nil {
		return err
	}

	s.mu.Lock()
	defer s.mu.Unlock()

	s.messages[body.Message] = struct{}{} ➋

	broadcastMessageResponse := BroadcastMessageResponse{
		Type: "broadcast_ok",
	}

	return s.node.Reply(msg, broadcastMessageResponse)
}

func (s *Server) readHandler(msg maelstrom.Message) error { ➌
	var body ReadMessage
	if err := json.Unmarshal(msg.Body, &body); err != nil {
		return err
	}

	s.mu.Lock()
	defer s.mu.Unlock()

	messages := make([]int, 0, len(s.messages))
	for m := range s.messages {
		messages = append(messages, m)
	}

	readMessageResponse := ReadMessageResponse{
		Type:     "read_ok",
		Messages: messages,
	}

	return s.node.Reply(msg, readMessageResponse)
}

func (s *Server) topologyHandler(msg maelstrom.Message) error { ➍
	var body TopologyMessage
	if err := json.Unmarshal(msg.Body, &body); err != nil {
		return err
	}

	// We ignore topology sent from maelstrom's controller, at least for now
	topologyMessageResponse := TopologyMessageResponse{
		Type: "topology_ok",
	}

	log.Printf("Received topology information from controller: %v", body.Topology)

	return s.node.Reply(msg, topologyMessageResponse)
}

func (s *Server) Run() error {
	return s.node.Run()
}

```

The actual implementation is simplified since the only possibility of messages under this workload is as follows:

![Maelstrom](/images/broadcast3a.drawio.svg)

That is, there is no possibility our node will receive any message from another node (since we're running one-node cluster).
In our `Server` struct I add 🄌 `sync.Mutex` protected map of message our node already have seen. In ➊ I define handler 
for `broadcast` message. We simply analyze whether our node has already seen broadcasted message and eventually
we update our map of messages ➋. Plain and easy. In `read` handler we simply read and return messages from our map ➌. Finally,
in `topology` message handler ➍ we just ignore the topology *suggested* by maelstrom to our node. That is because in later
stages our topology will be as follows: neighbors of given node are all the nodes except itself (kind of discrete topology). 
In final stages we'll build more efficient topology, but still it will be different from the one proposed by maelstrom.

## Running workload

Let's see if our node collects all the messages from controllers:

```sh
❯ make run
go build -o ~/go/bin/maelstrom-broadcast-3a ./broadcast-3a
```


```clojure
...
INFO [2025-12-20 19:30:49,999] jepsen worker 0 - jepsen.util 0  :invoke :read   nil
INFO [2025-12-20 19:30:50,000] jepsen worker 0 - jepsen.util 0  :ok     :read   [96 23 48 54 60 67 74 98 4 7 9 14 43 51 55 5 22 25 31 35 52 85 86 0 15 64 68 72 88 92 3 6 12 79 10 13 28 46 57 75 84 90 30 37 41 47 50 94 16 19 45 91 95 21 33 56 1 24 49 62 73 78 83 89 17 32 34 36 40 44 69 76 8 59 66 70 80 81 87 93 11 18 27 39 42 53 61 65 2 29 58 71 77 82 97 20 26 38 63]
INFO [2025-12-20 19:30:50,019] jepsen test runner - jepsen.core Run complete, writing
INFO [2025-12-20 19:30:50,047] jepsen node n0 - maelstrom.db Tearing down n0
INFO [2025-12-20 19:30:51,045] jepsen node n0 - maelstrom.net Shutting down Maelstrom network
INFO [2025-12-20 19:30:51,045] jepsen test runner - jepsen.core Analyzing...
INFO [2025-12-20 19:30:51,280] jepsen test runner - jepsen.core Analysis complete
INFO [2025-12-20 19:30:51,285] jepsen results - jepsen.store Wrote /home/deamondev/software_development/tutorials/gossip-glomers-tutorial/store/broadcast/20251220T193018.978+0100/results.edn
INFO [2025-12-20 19:30:51,314] jepsen test runner - jepsen.core {:perf {:latency-graph {:valid? true},
        :rate-graph {:valid? true},
        :valid? true},
 :timeline {:valid? true},
 :exceptions {:valid? true},
 :stats {:valid? true,
         :count 200,
         :ok-count 200,
         :fail-count 0,
         :info-count 0,
         :by-f {:broadcast {:valid? true,
                            :count 99,
                            :ok-count 99,
                            :fail-count 0,
                            :info-count 0},
                :read {:valid? true,
                       :count 101,
                       :ok-count 101,
                       :fail-count 0,
                       :info-count 0}}},
 :availability {:valid? true, :ok-fraction 1.0},
 :net {:all {:send-count 404,
             :recv-count 404,
             :msg-count 404,
             :msgs-per-op 2.02},
       :clients {:send-count 404, :recv-count 404, :msg-count 404},
       :servers {:send-count 0,
                 :recv-count 0,
                 :msg-count 0,
                 :msgs-per-op 0.0},
       :valid? true},
 :workload {:worst-stale (),
            :duplicated-count 0,
            :valid? true,
            :lost-count 0,
            :lost (),
            :stable-count 99,
            :stale-count 0,
            :stale (),
            :never-read-count 0,
            :stable-latencies {0 0, 0.5 0, 0.95 0, 0.99 0, 1 0},
            :attempt-count 99,
            :never-read (),
            :duplicated {}},
 :valid? true}


Everything looks good! ヽ(‘ー`)ノ
```

## Summary

Hurray! We're running a one-node broadcast system! Isn't is awesome? It is, of course, but more interesting problems
are just waiting for us around the corner. Let us see in a [multi-node broadcast challenge](https://fly.io/dist-sys/3b/).
