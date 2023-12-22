+++
title="Live Sequence Protocol and Distributed Bitcoin Miner"
date=2023-12-17

[taxonomies]
categories = ["CMU-15640"]
[extra]
toc = true

+++
# Overview
CMU-15640 Distributed Systems Project 1<br>
This project will consist of the following two parts:
• Part A: Implement the Live Sequence Protocol, a homegrown protocol for providing reliable communication with simple client and server APIs on top of the Internet UDP protocol.
• Part B: Implement a Distributed Bitcoin Miner.<br>
[Source Code Link](https://github.com/zoharrpg/Distributed-Bitcoin-Miner)<br>
[Write-Up]({{config.base_url}}15640-doc/p1_23.pdf)<br>

# Part 1 - Live Sequence Protocol
## LSP Overview
- Unlike UDP or TCP, it supports a client-server communication model.
- The server maintains connections between a number of clients, each of which is identified by a numeric connection identifier.
- Communication between the server and a client consists of a sequence of discrete messages in each direction.
- Message sizes are limited to fit within single UDP packets (around 1,000 bytes).
- Messages are sent reliably: a message sent over the network must be received exactly once, and messages must be received in the same order they were sent.
- Message integrity is ensured: a message sent over the network will be rejected if modified in transit.
- The server and clients monitor the status of their connections and detect when the other side has become disconnected.

## Message type and Acknowledge
In this project, there are common message and acknledge message type, and the implementation is following:<br>
```go
type Message struct {
	Type     MsgType // One of the message types listed above.
	ConnID   int     // Unique client-server connection ID.
	SeqNum   int     // Message sequence number.
	Size     int     // Size of the payload.
	Checksum uint16  // Message checksum.
	Payload  []byte  // Data message payload.
}
```
The Communication is following:<br>
![config.base_url]({{config.base_url}}15640-graph/p1-pic1.png)
![config.base_url]({{config.base_url}}15640-graph/p1-pic2.png)





