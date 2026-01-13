+++
date = '2025-10-28T08:47:45+01:00'
draft = true
enableEmoji = true
title = 'Solving gossip-glomers distributed systems challenges: echo (part 1)'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems']
toc = true
+++

## Echo Challenge

Let's start by solving the [first challenge](https://fly.io/dist-sys/1/). You may get the impression that my solution is
exaggerated, and I would agree with you! I decided to add some extra code at this point to emphasize the project architecture 
that I try to use when solving more complex challenges. I also try to incorporate Golang best practices, but take them with 
a pinch of salt since I've never been paid as a Go developer.

## Setup

Let's start by adding [maelstrom package](https://pkg.go.dev/github.com/jepsen-io/maelstrom/demo/go) to our `echo`
module:

```shell
echo❯ go get github.com/jepsen-io/maelstrom/demo/go
```

### Makefile

To automate process of building binary and running maelstrom workload over it, I created minimal Makefile:

```Makefile
MODULE = echo
BINARY = ~/go/bin/maelstrom-$(MODULE)

WORKLOAD = echo

MAELSTROM_CMD_echo = maelstrom/maelstrom test -w echo --bin $(BINARY) --node-count 1 --time-limit 10

MAELSTROM_RUN_CMD = $(MAELSTROM_CMD_$(WORKLOAD))

run: build
	@$(MAELSTROM_RUN_CMD)

build:
	go build -o $(BINARY) ./$(MODULE)

debug:
	maelstrom/maelstrom serve
```

That requires manual intervention to change module name while solving particular challenge. During this series I'll
update only the `MAELSTROM_CMD_X` entries, `MODULE` and `WORKLOAD` parameters.

I think the flags in the `MAELSTROM_CMD_echo` command are self-explanatory: there is basically exactly one node running
(`--node-count 1`) and we restrict time to 10 seconds (`--time-limit 10`).

## Let's code!

Let's create new file 

```shell
echo❯ touch server.go
```

Inside this file we'll define the struct `Server` which wraps `*maelstrom.Node` and other relevant things. In this  challenge we'll only store pointer to this node and node id. 


### echo/server.go

```go {lineNos=true}
package main 

import (
    "encoding/json"
	"log"

	maelstrom "github.com/jepsen-io/maelstrom/demo/go"
)

type Server struct { 🄌
    node   *maelstrom.Node
	nodeID string
}

type EchoMessage struct { ➊
	Type  string `json:"type"`
	MsgID int64  `json:"msg_id"`
	Echo  string `json:"echo"`
}

type EchoMessageResponse struct {
	Type  string `json:"type"`
	MsgID int64  `json:"msg_id"`
	Echo  string `json:"echo"`
}

func NewServer(n *maelstrom.Node) *Server { ➋
	s := &Server{node: n}

	s.node.Handle("init", s.initHandler)
	s.node.Handle("echo", s.echoHandler)

	return s
}
```
In 🄌, we define our struct that wraps the `*maelstrom.Node` type and store the node id. In ➊, I declare a number of types
related to the messages that our node should understand while running under the Maelstrom controller. In step ➋, I declare 
a public builder method based on `*maelstrom.Node`, and we configure the message handlers for our node. Note that we don't
set `s.nodeID` at this stage. You should understand why after carefully reading the previous part of this series. If you 
missed it, don't worry — I'll explain again shortly. Now, let's define the aforementioned handlers:

```go {lineNos=true,lineNoStart=40}
func (s *Server) initHandler(msg maelstrom.Message) error { 🄌
	var body maelstrom.InitMessageBody
	if err := json.Unmarshal(msg.Body, &body); err != nil {
		return err
	}

	s.nodeID = body.NodeID ➊

	log.Printf("Node id set to: %s", s.nodeID) ❷

	return nil
}

func (s *Server) echoHandler(msg maelstrom.Message) error { ❹
	var body EchoMessage
	if err := json.Unmarshal(msg.Body, &body); err != nil {
		return err
	}

	echoMessageResponse := EchoMessageResponse{
		Type:  "echo_ok",
		MsgID: body.MsgID,
		Echo:  body.Echo,
	}

	return s.node.Reply(msg, echoMessageResponse)
}

func (s *Server) Run() error { ❺
	return s.node.Run()
}
```

The function in  🄌 is an init message handler function. It is required to transform `maelstrom.Message` and to 
return an `error` (possibly `nil`). Note that, unlike for the 'echo' message, I didn't declare the body of the `init` message.
This is because maelstrom provides its own `maelstrom.InitMessageBody` into which we unmarshall bytes sent through
`msg.Body`. 

Having that, we successfully decoded the `InitMessageBody`, we're guaranteed to set ➊ our internal `nodeID` to 
`body.NodeID`. 

Next, we just audit ❷ that fact using [log](https://pkg.go.dev/log) package. I think there is no need at this moment to use more advanced loggers,
like [zap](https://pkg.go.dev/go.uber.org/zap) or [slog](https://go.dev/blog/slog). If during development of this series
I feel the need, then I probably use structured logger [slog](https://go.dev/blog/slog) which I have the optinion is the 
best option for greenfield go projects.

Analogously I declare ❹ `echoHandler`, the only difference is that we unmarshall message body to our own `EchoMessageResponse`.
We need to expplicitly set `Type` of this message to `"echo_ok"` to fullfil workload specification.

Then, I run [Reply](https://pkg.go.dev/github.com/jepsen-io/maelstrom/demo/go@v0.0.0-20250920002117-21168aa9cdd2#Node.Reply)
method which basically do the heavy lifting of wrapping our message `body` to actual message sent
thru' the wire to controller/node. I encourage you to see the
[actual implementation](https://github.com/jepsen-io/maelstrom/blob/21168aa9cdd2/demo/go/node.go#L186) of this method.

Finally, ❺ we wrap node's [Run](https://pkg.go.dev/github.com/jepsen-io/maelstrom/demo/go#Node.Run) method into method on our `*Server` struct. 
We wrap it since we do not expose a node handle as a public member.

If you read previous part or just dabbled into maelstrom protocol specification, then you may be consterned with the
fact we return `nil` in `initHandler` instead of `init_ok` message type. That is very misleading (and poorly documented).
In fact, when we would send such a `init_ok` reply then in some workloads we may get an error of the following form: 

```clojure
2023-02-25 21:15:31,337{GMT}	WARN	[n1 stdout] maelstrom.process: Error!
java.lang.AssertionError: Assert failed: Invalid dest for message #maelstrom.net.message.Message{:id 71, :src "n1", :dest "c10", :body {:in_reply_to 1, :type "init_ok"}}
```

The reason is, maelstrom go library [internally handles](https://github.com/jepsen-io/maelstrom/blob/main/demo/go/node.go#L166) `init` 
message on its own, allowing us only evantually to hook into it:

```go
func (n *Node) handleInitMessage(msg Message) error {
	var body InitMessageBody
	if err := json.Unmarshal(msg.Body, &body); err != nil {
		return fmt.Errorf("unmarshal init message body: %w", err)
	}
	n.Init(body.NodeID, body.NodeIDs)

	// Delegate to application initialization handler, if specified.
	if h := n.handlers["init"]; h != nil {
		if err := h(msg); err != nil {
			return err
		}
	}

	// Send back a response that the node has been initialized.
	log.Printf("Node %s initialized", n.id)
	return n.Reply(msg, MessageBody{Type: "init_ok"})
}
```

If we replied in handler, then `init_ok` would send to the controller node twice, which breaks maelstrom's specification.

As I said, it is very misleading and challenging to investigate what is going on without this knowledge. [Some folks have experienced](https://community.fly.io/t/challenge-3b-inconsistency-with-jepsen/10999) 
this issue and asked for help.

At least it’s good to know that it’s not a bug, just poor documentation. Who among us would want to work with buggy 
software in their free time?

### echo/main.go

We have no choice but to actually run our server. Before doing so, we need to take care of the logger's output.
Otherwise, we risk interfering with maelstrom's controllers. The code below is straightforward:

```go 
package main

import (
	"log"

	maelstrom "github.com/jepsen-io/maelstrom/demo/go"
)

func main() {
    log.SetOutput(os.Stderr)
    
	n := maelstrom.NewNode()

	s := NewServer(n)

	if err := s.Run(); err != nil {
		log.Fatal(err)
	}
}

```

## Double check of maelstrom specification by tracing linux syscalls

In the previous part, I provided a rough description of the Maelstrom specification. In particular, I mentioned that 
nodes communicate via their STDIN and STDOUT streams. But what about a double check?

First, let's run our Echo workload with our echo binary but with a slightly increased timeout.

```shell
❯ maelstrom/maelstrom test -w echo --bin ~/go/bin/maelstrom-echo --node-count 1 --time-limit 180
```

I just want to make sure maelstrom would not finish its work during our diagnostics. 

```shell
ps aux | grep '/go/bin/maelstrom-echo

deamond+   18792  0.0  0.0 1227720 4224 pts/3    Sl+  08:48   0:00 /home/deamondev/go/bin/maelstrom-echo
```

Having PID of our node, we can easily trace syscalls it performs using [strace(1)](https://man7.org/linux/man-pages/man1/strace.1.html)
tool:

```shell
❯ strace -p 18792

strace: Process 18792 attached
read(0, "{\"id\":699,\"src\":\"c2\",\"dest\":\"n0\""..., 2589) = 93
futex(0x5e2310, FUTEX_WAKE_PRIVATE, 1)  = 1
futex(0xc000070958, FUTEX_WAKE_PRIVATE, 1) = 1
--- SIGURG {si_signo=SIGURG, si_code=SI_TKILL, si_pid=18792, si_uid=1000} ---
rt_sigreturn({mask=[]})                 = 1
nanosleep({tv_sec=0, tv_nsec=3000}, NULL) = 0
futex(0xc000100158, FUTEX_WAKE_PRIVATE, 1) = 1
write(2, "2025/11/07 08:49:23 Sent {\"src\":"..., 130) = 130
write(1, "{\"src\":\"n0\",\"dest\":\"c2\",\"body\":{"..., 104) = 104
write(1, "\n", 1)                       = 1
futex(0x5e1c30, FUTEX_WAIT_PRIVATE, 0, NULL) = 0
futex(0x5e1c30, FUTEX_WAIT_PRIVATE, 0, NULL) = 0
futex(0xc000100158, FUTEX_WAKE_PRIVATE, 1) = 1
write(2, "2025/11/07 08:49:23 Sent {\"src\":"..., 131) = 131
write(1, "{\"src\":\"n0\",\"dest\":\"c2\",\"body\":{"..., 105) = 105
write(1, "\n", 1)                       = 1
```

We see a bunch of syscalls caused by interaction of maelstrom's client `c2` and our process. The [futex(2)](https://www.man7.org/linux/man-pages/man2/futex.2.html) syscall
is present since go runtime [uses it internally](https://go.dev/src/runtime/os_linux.go) (under linux) for goroutine 
scheduling and synchronization but it is totally irrelevant for our discussion.

Under `store/latest/node-logs/n0.log` we find logs which corresponds to these [write(2)](https://www.man7.org/linux/man-pages/man2/write.2.html) 
system calls with its first argument `fd=2`:

```json
2025/11/07 08:48:14 Received {c0 n0 {"type":"init","node_id":"n0","node_ids":["n0"],"msg_id":1}}
2025/11/07 08:48:14 Node id set to: n0
2025/11/07 08:48:14 Sent {"src":"n0","dest":"c0","body":{"in_reply_to":1,"type":"init_ok"}}
2025/11/07 08:48:14 Node n0 initialized
2025/11/07 08:48:14 Received {c2 n0 {"echo":"Please echo 123","type":"echo","msg_id":1}}
2025/11/07 08:48:14 Sent {"src":"n0","dest":"c2","body":{"echo":"Please echo 123","in_reply_to":1,"msg_id":1,"type":"echo_ok"}}
2025/11/07 08:48:14 Received {c2 n0 {"echo":"Please echo 108","type":"echo","msg_id":2}}
```

## Running workload

Drums rolling. Let's see if we're good:

```shell
❯ make run
go build -o ~/go/bin/maelstrom-echo ./echo
```

```clojure
...
INFO [2025-11-10 20:58:22,291] jepsen test runner - jepsen.core Run complete, writing
INFO [2025-11-10 20:58:22,313] jepsen node n0 - maelstrom.db Tearing down n0
INFO [2025-11-10 20:58:23,274] jepsen node n0 - maelstrom.net Shutting down Maelstrom network
INFO [2025-11-10 20:58:23,274] jepsen test runner - jepsen.core Analyzing...
INFO [2025-11-10 20:58:23,476] jepsen test runner - jepsen.core Analysis complete
INFO [2025-11-10 20:58:23,483] jepsen results - jepsen.store Wrote /home/deamondev/software_development/tutorials/gossip-glomers-tutorial/store/echo/20251110T205811.203+0100/results.edn
INFO [2025-11-10 20:58:23,502] jepsen test runner - jepsen.core {:perf {:latency-graph {:valid? true},
        :rate-graph {:valid? true},
        :valid? true},
 :timeline {:valid? true},
 :exceptions {:valid? true},
 :stats {:valid? true,
         :count 53,
         :ok-count 53,
         :fail-count 0,
         :info-count 0,
         :by-f {:echo {:valid? true,
                       :count 53,
                       :ok-count 53,
                       :fail-count 0,
                       :info-count 0}}},
 :availability {:valid? true, :ok-fraction 1.0},
 :net {:all {:send-count 109,
             :recv-count 108,
             :msg-count 109,
             :msgs-per-op 2.0566037},
       :clients {:send-count 109, :recv-count 108, :msg-count 109},
       :servers {:send-count 0,
                 :recv-count 0,
                 :msg-count 0,
                 :msgs-per-op 0.0},
       :valid? true},
 :workload {:valid? true, :errors ()},
 :valid? true}


Everything looks good! ヽ(‘ー`)ノ
```

We did it! I think these logs should be understandable after careful read. The most important metric here is `:msgs-per-op`
which measures average number of network messages sent per client operation. For each echo operation, the system typically:

- sends request to the node (1 message)
- node sends response back (1 message)

Thats *roughly* 2 messages per operation. The extra `0.566037` likely comes from protocol overhead - things like heartbeats 
between nodes, timeouts being retried, or acknowledgments in the gossip protocol.

**In summary:** The system is working well — all operations succeeded, the network is stable, and message overhead is 
reasonable.


## Summary

I hope I haven't bored any of you, as there haven't been many distributed systems problems up to this point. Nevertheless,
I have tried to smuggle what I consider to be the best practices for building such a system. The very [next challenge](https://fly.io/dist-sys/2/)
also belongs to the 'warm-up' category and, in fact, is actually quite similar to the one we just solved 
(at least in terms of the number of lines of code).