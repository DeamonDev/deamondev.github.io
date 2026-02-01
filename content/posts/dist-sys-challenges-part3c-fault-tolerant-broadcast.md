+++
date = '2025-12-27T11:54:46+01:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: fault tolerant broadcast (part 3c)'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems', 'broadcast', 'fault tolerance']
toc = true
+++

## Fault Tolerant Broadcast Challenge

In this section, we extend the multi-node broadcast challenge by introducing network partitions between the nodes, meaning
they will be unable to communicate with each other for certain periods of time.

## Setup

Run these commands to bootstrap this part:

```shell
❯ mkdir broadcast-3c
broadcast-3c❯ go mod init github.com/deamondev/gossip-glomers-tutorial/broadcast-3c
❯ go work use ./broadcast-3c
```

### Makefile
Let's calibrate `MODULE` and `WORKLOAD` parameters to be `broadcast-3c`. Our new maelstrom command should be:

```shell
MAELSTROM_CMD_broadcast-3c = maelstrom/maelstrom test -w broadcast --bin $(BINARY) --node-count 5 --time-limit 20 --rate 10 --nemesis partition
```

This will run a 5-node cluster like before, but this time with a failing network!

## Code

I am posting here only the relevant part that has changed to deal with network partition.
The rest of the code remains identical to a multi-node broadcast.

### (part of) broadcast-3c/server.go

```go
func (s *Server) broadcastHandler(msg maelstrom.Message) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	var body BroadcastMessage
	if err := json.Unmarshal(msg.Body, &body); err != nil {
		return err
	}

	// To avoid cycles: n0->n1->n2->n0
	if _, exists := s.messages[body.Message]; exists {
		broadcastMessageResponse := BroadcastMessageResponse{
			Type: "broadcast_ok",
		}

		return s.node.Reply(msg, broadcastMessageResponse)
	}

	s.messages[body.Message] = struct{}{}

	// To avoid: n0->n0
	for _, peerID := range s.peers {
		go broadcastMessageToPeer(s.node, peerID, body) ⓿
	}

	broadcastMessageResponse := BroadcastMessageResponse{
		Type: "broadcast_ok",
	}

	return s.node.Reply(msg, broadcastMessageResponse)
}

func broadcastMessageToPeer(node *maelstrom.Node, peerID string, body BroadcastMessage) {
	for {
		ctx, cancel := context.WithTimeout(context.Background(), time.Second) ❶
		_, err := node.SyncRPC(ctx, peerID, body) ❷
		cancel() ❸

		if err == nil {
			return
		}

		time.Sleep(time.Duration(rand.Intn(50)) * time.Millisecond) ❹
	}
}
```

The main change is that I spawn ⓿ a separate goroutine when broadcasting a message to each peer. The actual logic is now 
inside the `broadcastMessageToPeer` function, where we essentially repeat ad infinitum the process of spawning a new context ❶
with a one-second timeout. Using that context, we send ❷ a [SyncRPC](https://pkg.go.dev/github.com/jepsen-io/maelstrom/demo/go#Node.SyncRPC) 
call to the given peer. SyncRPC sends a synchronous RPC request. 
RPC errors in the message body are converted to an [RPCError](https://pkg.go.dev/github.com/jepsen-io/maelstrom/demo/go#RPCError)
and returned. Then we `cancel()` our context ❸, and if everything went well, we simply `return`. If an error occurs,
we sleep ❹ for a randomized period of time (up to 50 milliseconds).

**Remark**. Spawning goroutines causes little drawback
for maelstrom to collect *local* statistics of network latencies, since we're basically returning `broadcastHandler` eagerly.
We should not care too much about it, since the most important statistics are percentiles related to propagation of messages
on the whole network.

**Another Remark**. A classic example of sleep randomization ❹ is [Raft](https://raft.github.io/), 
where nodes randomize their election timeouts on startup and after failures. 
This reduces the probability that multiple nodes start elections simultaneously, preventing endless split votes and helping 
the system converge quickly on a single leader.

## Running workload

Let us see if our nodes communicate successfully, regardless of weather conditions:
```sh
❯ make run
go build -o ~/go/bin/maelstrom-broadcast-3c ./broadcast-3c
```


```clojure
...
INFO [2025-12-27 12:22:10,291] jepsen node n0 - maelstrom.net Shutting down Maelstrom network
INFO [2025-12-27 12:22:10,292] jepsen test runner - jepsen.core Analyzing...
INFO [2025-12-27 12:22:10,624] jepsen test runner - jepsen.core Analysis complete
INFO [2025-12-27 12:22:10,629] jepsen results - jepsen.store Wrote /home/deamondev/software_development/tutorials/gossip-glomers-tutorial/store/broadcast/20251227T122137.189+0100/results.edn
INFO [2025-12-27 12:22:10,656] jepsen test runner - jepsen.core {:perf {:latency-graph {:valid? true},
        :rate-graph {:valid? true},
        :valid? true},
 :timeline {:valid? true},
 :exceptions {:valid? true},
 :stats {:valid? true,
         :count 191,
         :ok-count 191,
         :fail-count 0,
         :info-count 0,
         :by-f {:broadcast {:valid? true,
                            :count 97,
                            :ok-count 97,
                            :fail-count 0,
                            :info-count 0},
                :read {:valid? true,
                       :count 94,
                       :ok-count 94,
                       :fail-count 0,
                       :info-count 0}}},
 :availability {:valid? true, :ok-fraction 1.0},
 :net {:all {:send-count 8091,
             :recv-count 4282,
             :msg-count 8091,
             :msgs-per-op 42.361256},
       :clients {:send-count 402, :recv-count 402, :msg-count 402},
       :servers {:send-count 7689,
                 :recv-count 3880,
                 :msg-count 7689,
                 :msgs-per-op 40.256546},
       :valid? true},
 :workload {:worst-stale (),
            :duplicated-count 0,
            :valid? true,
            :lost-count 0,
            :lost (),
            :stable-count 97,
            :stale-count 0,
            :stale (),
            :never-read-count 0,
            :stable-latencies {0 0, 0.5 0, 0.95 0, 0.99 0, 1 0},
            :attempt-count 97,
            :never-read (),
            :duplicated {}},
 :valid? true}


Everything looks good! ヽ(‘ー`)ノ
```

## Summary



