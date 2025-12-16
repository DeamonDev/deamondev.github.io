+++
date = '2025-12-14T12:57:29+01:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: unique-ids (part 2)'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems']
toc = true
+++

## Unique IDs Challenge

In this challenge we'll run three-node workload which should implement a globally-unique ID generation system.
Basically, this means that each of our nodes should respond with unique-id every time it is asked for, being totally available, meaning
that it can continue to operate even in the face of network partitions.

## Setup

Run these commands to bootstrap this part:

```shell
❯ mkdir unique-ids
unique-ids❯ go mod init github.com/deamondev/gossip-glomers-tutorial/unique-ids
❯ go work use ./unique-ids
```

### Makefile
Let's calibrate `MODULE` and `WORKLOAD` parameters to be `unique-ids`. Our new maelstrom command should be: 

```shell
MAELSTROM_CMD_unique-ids = maelstrom/maelstrom test -w unique-ids --bin $(BINARY) --time-limit 30 --rate 1000 --node-count 3 --availability total --nemesis partition
```

## Code

Let's create two new files:

```shell
unique-ids❯ touch main.go server.go
```

The `main.go` file remains exactly the same as in previous solution and it will remain the same up to `broadcast-3e` challenge, hence I'll skip it up to 
that moment.

### unique-ids/server.go

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"sync"

	maelstrom "github.com/jepsen-io/maelstrom/demo/go"
)

type Server struct {
	node    *maelstrom.Node
	nodeID  string
	mu      sync.Mutex 🄌
	counter uint64
}

type GenerateMessage struct {
	Type string `json:"type"`
}

type GenerateMessageResponse struct {
	Type string `json:"type"`
	Id   string `json:"id"`
}

func NewServer(n *maelstrom.Node) *Server {
	s := &Server{node: n, counter: 0}

	s.node.Handle("init", s.initHandler)
	s.node.Handle("generate", s.generateHandler)

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

func (s *Server) generateHandler(msg maelstrom.Message) error { ➊
	var body GenerateMessage
	if err := json.Unmarshal(msg.Body, &body); err != nil {
		return err
	}

	s.mu.Lock() ➋
	defer s.mu.Unlock()

	id := fmt.Sprintf("%s-%d", s.nodeID, s.counter) ❸
	generateMessageResponse := GenerateMessageResponse{
		Type: "generate_ok",
		Id:   id,
	}

	s.counter++ ❹
	log.Printf("Internal node counter incremented, current value: %d", s.counter)

	return s.node.Reply(msg, generateMessageResponse)
}

func (s *Server) Run() error {
	return s.node.Run()
}

```

There is now a shared mutable state stored inside the `Server` struct, which is just an internal counter 
that our node increments. 
We protect it via a `sync.Mutex`. The idea is that `i`-th node should return something like `node-i-42` when
asked for `42`-nd time for a unique id. The node prefix and the protected counter guarantee uniqueness. Let's
focus on the implementation of the handler for doing so. After locking the counter for the lifetime with
deferring for the lifetime of the function execution with deferring, we prepare aforementioned `id` returned 
from the handler. Finally, the counter is being incremented.

## Running workload

Let's see if our glomers don't repeat themselves:

```shell
❯ make run
go build -o ~/go/bin/maelstrom-unique-ids ./unique-ids
```

```clojure
...
INFO [2025-12-14 14:11:55,736] jepsen worker 1 - jepsen.util 1  :invoke :generate       nil
INFO [2025-12-14 14:11:55,736] jepsen worker 1 - jepsen.util 1  :ok     :generate       "n1-7399"
INFO [2025-12-14 14:11:55,736] jepsen worker 2 - jepsen.util 2  :invoke :generate       nil
INFO [2025-12-14 14:11:55,736] jepsen worker 2 - jepsen.util 2  :ok     :generate       "n2-7069"
INFO [2025-12-14 14:11:55,736] jepsen worker 0 - jepsen.util 0  :invoke :generate       nil
INFO [2025-12-14 14:11:55,736] jepsen worker 0 - jepsen.util 0  :ok     :generate       "n0-13731"
INFO [2025-12-14 14:11:55,737] jepsen worker 0 - jepsen.util 0  :invoke :generate       nil
INFO [2025-12-14 14:11:55,738] jepsen worker 0 - jepsen.util 0  :ok     :generate       "n0-13732"
INFO [2025-12-14 14:11:55,758] jepsen test runner - jepsen.core Run complete, writing
INFO [2025-12-14 14:11:56,027] jepsen node n2 - maelstrom.db Tearing down n2
INFO [2025-12-14 14:11:56,027] jepsen node n0 - maelstrom.db Tearing down n0
INFO [2025-12-14 14:11:56,027] jepsen node n1 - maelstrom.db Tearing down n1
INFO [2025-12-14 14:11:57,727] jepsen node n0 - maelstrom.net Shutting down Maelstrom network
INFO [2025-12-14 14:11:57,728] jepsen test runner - jepsen.core Analyzing...
INFO [2025-12-14 14:11:58,474] jepsen test runner - jepsen.core Analysis complete
INFO [2025-12-14 14:11:58,478] jepsen results - jepsen.store Wrote /home/deamondev/software_development/tutorials/gossip-glomers-tutorial/store/unique-ids/20251214T141124.781+0100/results.edn
INFO [2025-12-14 14:11:58,509] jepsen test runner - jepsen.core {:perf {:latency-graph {:valid? true},
        :rate-graph {:valid? true},
        :valid? true},
 :timeline {:valid? true},
 :exceptions {:valid? true},
 :stats {:valid? true,
         :count 28203,
         :ok-count 28203,
         :fail-count 0,
         :info-count 0,
         :by-f {:generate {:valid? true,
                           :count 28203,
                           :ok-count 28203,
                           :fail-count 0,
                           :info-count 0}}},
 :availability {:valid? true, :ok-fraction 1.0},
 :net {:all {:send-count 56412,
             :recv-count 56412,
             :msg-count 56412,
             :msgs-per-op 2.0002127},
       :clients {:send-count 56412,
                 :recv-count 56412,
                 :msg-count 56412},
       :servers {:send-count 0,
                 :recv-count 0,
                 :msg-count 0,
                 :msgs-per-op 0.0},
       :valid? true},
 :workload {:valid? true,
            :attempted-count 28203,
            :acknowledged-count 28203,
            :duplicated-count 0,
            :duplicated {},
            :range ["n0-0" "n2-999"]},
 :valid? true}


Everything looks good! ヽ(‘ー`)ノ
```

## Summary

I think it was a good warm-up round. The next challenges focus on developing the increasingly complex problem of *message broadcast*.





