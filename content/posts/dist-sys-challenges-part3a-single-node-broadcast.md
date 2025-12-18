+++
date = '2025-12-18T09:22:19+01:00'
draft = true
title = 'Solving gossip-glomers distributed systems challenges: single node broadcast (part 3a)'
categories = ['software-development', 'distributed-systems', 'broadcast']
tags = ['distributed systems']
toc = true
+++

## Single Node Broadcast Challenge

## Setup

Run these commands to bootstrap this part:

```shell
❯ mkdir broadcast-3a
unique-ids❯ go mod init github.com/deamondev/gossip-glomers-tutorial/broadcast-3a
❯ go work use ./broadcast-3a
```

### Makefile
Let's calibrate `MODULE` and `WORKLOAD` parameters to be `broadcast-3a`. Our new maelstrom command should be:

```shell
MAELSTROM_CMD_broadcast-3a = maelstrom/maelstrom test -w broadcast --bin $(BINARY) --node-count 1 --time-limit 20 --rate 10
```

## Code