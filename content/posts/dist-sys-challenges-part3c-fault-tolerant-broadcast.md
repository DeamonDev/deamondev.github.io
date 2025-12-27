+++
date = '2025-12-27T11:54:46+01:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: fault tolerant broadcast (part 3c)'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems', 'broadcast', 'fault tolerance']
toc = true
+++

## Fault Tolerant Broadcast Challenge

...

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

## Code

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

...

## Running workload

Let's see if...

```sh
❯ make run
go build -o ~/go/bin/maelstrom-broadcast-3c ./broadcast-3c
```


```clojure

```

## Summary



