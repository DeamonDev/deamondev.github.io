+++
date = '2025-12-26T20:25:58+01:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: multi node broadcast (part 3b)'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems', 'broadcast']
toc = true
+++

## Multi Node Broadcast Challenge

...

## Setup

Run these commands to bootstrap this part:

```shell
❯ mkdir broadcast-3b
broadcast-3b❯ go mod init github.com/deamondev/gossip-glomers-tutorial/broadcast-3b
❯ go work use ./broadcast-3b
```

### Makefile
Let's calibrate `MODULE` and `WORKLOAD` parameters to be `broadcast-3b`. Our new maelstrom command should be:

```shell
MAELSTROM_CMD_broadcast-3b = maelstrom/maelstrom test -w broadcast --bin $(BINARY) --node-count 5 --time-limit 20 --rate 10
```

## Code

### broadcast-3b/server.go

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
	peers  []string ⓿

	mu       sync.Mutex
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

	// no-op handlers
	s.node.Handle("broadcast_ok", s.noOpHandler) ❹

	return s
}

func (s *Server) initHandler(msg maelstrom.Message) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	var body maelstrom.InitMessageBody
	if err := json.Unmarshal(msg.Body, &body); err != nil {
		return err
	}

	s.nodeID = body.NodeID
	log.Printf("Node id set to: %s", s.nodeID)

	var peers []string ❶
	for _, peerID := range body.NodeIDs {
		if peerID != s.nodeID {
			peers = append(peers, peerID)
		}
	}

	s.peers = peers
	log.Printf("Discovered cluster peers: %v", s.peers)

	return nil
}

func (s *Server) broadcastHandler(msg maelstrom.Message) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	var body BroadcastMessage
	if err := json.Unmarshal(msg.Body, &body); err != nil {
		return err
	}

	// To avoid cycles: n0->n1->n2->n0
	if _, exists := s.messages[body.Message]; exists { ❷
		broadcastMessageResponse := BroadcastMessageResponse{
			Type: "broadcast_ok",
		}

		return s.node.Reply(msg, broadcastMessageResponse)
	}

	s.messages[body.Message] = struct{}{}

	// To avoid: n0->n0
	for _, peerID := range s.peers { ❸
		err := s.node.Send(peerID, body)
		if err != nil {
			log.Printf("Failed to broadcast message to node: %s", peerID)
			continue
		}
	}

	broadcastMessageResponse := BroadcastMessageResponse{
		Type: "broadcast_ok",
	}

	return s.node.Reply(msg, broadcastMessageResponse)
}

func (s *Server) noOpHandler(maelstrom.Message) error {
	return nil
}

func (s *Server) readHandler(msg maelstrom.Message) error {
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

func (s *Server) topologyHandler(msg maelstrom.Message) error {
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

We expand our `Server` struct with `peers` field ⓿, which we simply initialize with all the other node id's in the cluster
in our `init` method ❶.

In `broadcast` message handler we add guard which cheks if our node has seen the incoming message ❷. If it has seen this
particular message, then we simply respond with `broadcast_ok`. This is important, because since we use quite an 
aggressive topology, we need to avoid possible cycles: 

![Maelstrom](/images/broadcast3b-peers.drawio.svg)

In the opposite case ❸, our node just broadcast the same broadcast message to its peers. Because of that, our node need
to be able to deal with `broadcast_ok` message (because peers of node will respond with that one). I just ignore it using
custom `noOpHandler` ❹ which I'll reuse in later stages as well.

## Running workload

Let's see if our nodes propagate messages correctly:

```sh
❯ make run
go build -o ~/go/bin/maelstrom-broadcast-3b ./broadcast-3b
```


```clojure
...
INFO [2025-12-26 20:43:57,463] jepsen test runner - jepsen.core Analysis complete
INFO [2025-12-26 20:43:57,470] jepsen results - jepsen.store Wrote /home/deamondev/software_development/tutorials/gossip-glomers-tutorial/store/broadcast/20251226T204324.913+0100/results.edn
INFO [2025-12-26 20:43:57,493] jepsen test runner - jepsen.core {:perf {:latency-graph {:valid? true},
        :rate-graph {:valid? true},
        :valid? true},
 :timeline {:valid? true},
 :exceptions {:valid? true},
 :stats {:valid? true,
         :count 203,
         :ok-count 203,
         :fail-count 0,
         :info-count 0,
         :by-f {:broadcast {:valid? true,
                            :count 110,
                            :ok-count 110,
                            :fail-count 0,
                            :info-count 0},
                :read {:valid? true,
                       :count 93,
                       :ok-count 93,
                       :fail-count 0,
                       :info-count 0}}},
 :availability {:valid? true, :ok-fraction 1.0},
 :net {:all {:send-count 4826,
             :recv-count 4826,
             :msg-count 4826,
             :msgs-per-op 23.7734},
       :clients {:send-count 426, :recv-count 426, :msg-count 426},
       :servers {:send-count 4400,
                 :recv-count 4400,
                 :msg-count 4400,
                 :msgs-per-op 21.674877},
       :valid? true},
 :workload {:worst-stale (),
            :duplicated-count 0,
            :valid? true,
            :lost-count 0,
            :lost (),
            :stable-count 110,
            :stale-count 0,
            :stale (),
            :never-read-count 0,
            :stable-latencies {0 0, 0.5 0, 0.95 0, 0.99 0, 1 0},
            :attempt-count 110,
            :never-read (),
            :duplicated {}},
 :valid? true}


Everything looks good! ヽ(‘ー`)ノ
```

## Summary

This is something. In my taste, this is the first challenge in which we gently touch the surface of building a distributed
system.

