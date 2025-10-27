+++
date = '2025-10-19T17:43:15+02:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: setup (part 0)'
categories = ['software-development', 'distributed-systems']
tags = ['distributed systems']  
+++

# Introduction

In this series, I will demonstrate my approach to solving the distributed systems challenges posted
by [fly.io](https://fly.io/), known as [Gossip Glomers Challenges](https://fly.io/blog/gossip-glomers/). These are some
of my favourite distributed systems challenges and are built on top of
the [maelstrom](https://github.com/jepsen-io/maelstrom) library (the younger brother
of [jepsen](https://github.com/jepsen-io/jepsen) fault injection framework). In this post, I'll provide a brief
introduction to maelstrom and I'll setup the golang repository.

At the end of each article, you will find the diff output of `git show HEAD`, which you can use to double-check or to
get the TL;DR. I also manage git branches corresponding to these articles, for example the branch corresponding to this
one is `part0-setup`.

Here is link to the repository: https://github.com/DeamonDev/gossip-glomers-tutorial/tree/master

**Disclaimer:** Regarding the challenges: I don't want to rewrite what is already written in the challenge
specifications. So, before digging into the challenge, please read the corresponding specification. I assume you
understand what needs to be done before you start looking at solutions.

# Why golang?

Mainly because there is an [official maelstrom package](https://pkg.go.dev/github.com/jepsen-io/maelstrom/demo/go) and I
quite like golang for writing such kind of a software. I'd most likely opt for [gleam](https://gleam.run)'s package over
go's if gleam offered it.

# What is maelstrom?

Maelstrom is a distributed systems testing framework. It allows us to test distributed algorithms under load. network
partitions and so on. You may be familiar with jepsen, which is another framework for testing actual software (such as
mysql or postgres). Maelstrom focuses on testing only the logic (algorithm) of the underlying distributed system.

Technically, maelstrom is just a binary and CLI tool that takes your binary and runs it (possibly replicated) as a
normal process on your host machine. These running processes are called maelstrom nodes and are supposed to satisfy
maelstrom protocol (more on that later). Then maelstrom controller (the parent process which manages everything)
runs tests based on *workload*.

The workload is simply a description of what should happen to your nodes. This may involve simulating network
partitions, send messages to nodes, and so on. Based on their replies, it then collects statistics, among other things,
in the generated store directory. To name a few:

- **Jepsen log**
    - file: `store/jepsen.log`
    - description: The full logs from the test run, as printed to the console.
- **Results log**
    - file: `store/results.edn`
    - description: The results of the test's checker, including statistics and safety analysis. This structure is also
      printed at the end of a test.
- **Latency graphs**
    - file: `store/latencies.png`
    - description: Shows distribution of request latencies over time or as a histogram.
- **Node's logs**
    - file: `store/node-logs/{{ node_id }}.log`
    - description: logs of the node. Technically, the result of writing of corresponding process to its STDERR.
    - [example](https://gist.github.com/DeamonDev/f4072f0fb2c933aa37915798139f4be0)

See [here](https://github.com/jepsen-io/maelstrom/blob/main/doc/results.md#common-files) for complete description of
what can be found under the `store` directory.

Maelstrom provides [precompiled workloads](https://github.com/jepsen-io/maelstrom/blob/main/doc/workloads.md#workloads),
which I'll use in these challenges. If you really want to, you can even write your own workload which is just clojure
file and run maelstrom binary with the workload flag set to this local file. For example,
this [echo workload](https://github.com/jepsen-io/maelstrom/blob/main/src/maelstrom/workload/echo.clj) describes nodes
that respond with the same message body as they receive, and it is precompiled into maelstrom binary directly.

## Maelstrom protocol

Quoting official protocol documentation:

> Maelstrom nodes receive messages on STDIN, send messages on STDOUT, and log debugging output on STDERR. Maelstrom
> nodes must not print anything that is not a message to STDOUT.

Generic maelstrom message has the following form:

```
{
  "src":  A string identifying the node this message came from
  "dest": A string identifying the node this message is to
  "body": An object: the payload of the message
}
```

these messages are sent/received via STDOUT/STDIN of nodes (OS processes). I highly encourage you to read this protocol
specification before tackling the challenges. It is a short and simple specification.

![Maelstrom](/images/maelstrom.drawio.svg)

As shown above, nodes can communicate directly with each other. There are also maelstrom's internal processes, called
*clients* or *controllers* which also communicate with nodes. An example of such a message exchange is the `init`
message sent by (some) client to node:

```json
{
  "type": "init",
  "msg_id": 1,
  "node_id": "n3",
  "node_ids": [
    "n1",
    "n2",
    "n3"
  ]
}
```

The node is obligated to reply with `"init_ok"` message type:

```json
{
  "type": "init_ok",
  "in_reply_to": 1
}
```

It is beneficial to read about this protocol before attempting to solve the challenges. For example, as I already
pointed out, I'll use golang maelstrom package which
exposes [ID method](https://pkg.go.dev/github.com/jepsen-io/maelstrom/demo/go#Node.ID)
on [Node](https://pkg.go.dev/github.com/jepsen-io/maelstrom/demo/go#Node) struct:

```golang
func (n *Node) ID() string
```

This identifier is valid only after the `init` message has been received.

**Remark:** It should be noted that there is no network stack involved (counting loopback interface). Communication is
based solely on sending JSON messages using raw syscalls using relevant file descriptions of internal OS processes.

# Project setup

Enough talk. Let's start here. The plan is to create new golang project using go's workspace setup. After creating a new
directory and initializing git repo, let's create our workspace at the root of this directory:

```shell
❯ go work init
```

Having that, let's create a new subdirectory with the go module for our echo challenge. Then, we'll add to our
workspace:

```shell
❯ mkdir echo
❯ cd echo
echo❯ go mod init github.com/deamondev/gossip-glomers-tutorial/echo
echo❯ echo 'package main; import "fmt"; func main() { fmt.Println("echo") }' > main.go
echo❯ cd ..
❯ go work use ./echo
```

Please don't judge me on this formatting, it's temporary. We should see that the generated `go.work` file is being
updated:

```shell
❯ cat go.work
go 1.25.3

use ./echo
```

## .gitignore

Since I rather want to avoid commiting large files to git, I post here a minimal .gitignore file:

```.gitignore
maelstrom/
store/
.idea/
```

# Summary

We’re ready to start solving the first challenge — the [echo challenge](https://fly.io/dist-sys/1/) (who would’ve
expected that?). This part was just groundwork for our solutions, but I believe having a well-structured codebase with
an automated workflow will pay off later.

{{< details summary="**Click here to see the directory structure**">}}

```shell
.
├── echo
│   ├── go.mod
│   └── main.go
├── .gitignore
└── go.work

2 directories, 4 files
```

{{< /details >}}

{{< details summary="**Click here to see the diff**">}}

```diff
commit 013abf47fd8d49fc73633e19aa97d072c6e95866
Author: deamondev <piotr.rudnicki94@protonmail.com>
Date:   Mon Oct 27 08:50:33 2025 +0100

    part0 setup

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..87bb0cb
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,3 @@
+maelstrom/
+store/
+.idea/
\ No newline at end of file
diff --git a/echo/go.mod b/echo/go.mod
new file mode 100644
index 0000000..9e617ee
--- /dev/null
+++ b/echo/go.mod
@@ -0,0 +1,3 @@
+module github.com/deamondev/gossip-glomers-tutorial/echo
+
+go 1.25.3
diff --git a/echo/main.go b/echo/main.go
new file mode 100644
index 0000000..e222e9f
--- /dev/null
+++ b/echo/main.go
@@ -0,0 +1,5 @@
+package main
+
+import "fmt"
+
+func main() { fmt.Println("echo") }
diff --git a/go.work b/go.work
new file mode 100644
index 0000000..989015b
--- /dev/null
+++ b/go.work
@@ -0,0 +1,3 @@
+go 1.25.3

```

{{< /details >}}
