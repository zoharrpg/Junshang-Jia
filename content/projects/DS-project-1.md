+++
title="Live Sequence Protocol and Distributed Bitcoin Miner"
date=2023-12-17

[taxonomies]
categories = ["CMU-15640","Distributed System"]
[extra]
toc = true

+++
# Overview
CMU-15640 Distributed Systems Project 1<br>
This project will consist of the following two parts:
• Part 1: Implement the Live Sequence Protocol, a homegrown protocol for providing reliable communication with simple client and server APIs on top of the Internet UDP protocol.
• Part 2: Implement a Distributed Bitcoin Miner.<br>
[Source Code Link](https://github.com/zoharrpg/Distributed-Bitcoin-Miner)<br>
{{ render_link(path="15640-doc/p1_23.pdf",text = "Write-Up")}}

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
{{ render_image(path="/static/15640-doc/p1-pic1.png") }}
{{ render_image(path="/static/15640-doc/p1-pic2.png") }}
{{ render_image(path="/static/15640-doc/p1-pic3.png") }}

This protocol also supports the  Epoch events to make server robust.

Three Epoch events concepts define here:

**EpochLimit** - the maximum amount of epochs that can elapse without a response before we declare the connection dead. We need an epoch limit in order to ensure that we eventually drop dead connections.

**CurrentBackoff** - the amount of epochs we wait before re-transmitting data that did not receive an ACK. For instance, if CurrentBackoff is 8, and we just tried to send some data but did not receive an ACK, then we will only attempt to resend the data again after waiting 8 epochs. We provide some more examples below.

**MaxBackOffInterval** - the maximum amount of epochs we wait without retrying to transmit data. As in the name, CurrentBackoff is always less than or equal to MaxBackOffInterval

## Server API Implementation
In this project, I mainly focus on using Go channel to handel threads and manage the requests coming from client.

```go
type server struct {
	listener         *lspnet.UDPConn            // server listener
	rawMessages      chan MessagePacket         // receive message from client channel
	closeMain        chan int                   // close server signal
	closeClient      chan int                   // close client signal
	readMessages     chan Message               // Read channel
	addrAndSeqNum    map[int]*Client_seq        // ID to client seq map, to get client seq by connId
	readCounters     map[int]int                // read counter for each connId
	outOfOrderList   map[int]*list.List         // outorder list for each connId
	windows          map[int][]Message          // slide window for each client
	windowOutOfOrder map[int][]Message          // slide window outorder list
	isMessageSent    map[int]bool               // in each epoch, True if there is message sent
	sendMessages     chan SendPackage           // send message from server queue channel for send message from server
	readList         *list.List                 // store ordered message
	readRequest      chan int                   // send read signal
	messageDupMap    map[int]map[int]bool       // id to receive request seq
	connectionDupMap map[string]bool            // // The above code is declaring a variable called `connectionDupMap` which is a map with string keys and boolean values.
	idleEpochElapsed map[int]int                //connid to epoch
	messageBackoff   map[MessageId]*BackOffInfo // The above code is declaring a variable named "messageBackoff" which is a map with keys of type "MessageId" and values of type "*BackOffInfo".
	closePending     chan int                   // wait for message sent and acknowledgement.
	closeId          int                        // a variable called "closeId" of type int.
	isClosed         bool                       // a boolean variable named "isClosed" and initializing it to false.
	droppedId        chan int                   // drop connection id channel
	params           *Params                    // params for server
}
```
Go channel and select statement is a powerful tool to handel threads by using message passing, which make it very easy to write the server code. I also use the ticker library to send signal to channel to allow the server to support Epoch events.

```go
func (s *server) mainRoutine() {
	// The above code is creating a new ticker in Go. A ticker is a channel-based timer that will send a
	// value periodically based on the specified duration. In this case, the ticker will send a value
	// every `s.params.EpochMillis` milliseconds.
	ticker := time.NewTicker(time.Duration(s.params.EpochMillis) * time.Millisecond)
	// The above code is declaring a variable called `clientIdCounter` and initializing it with a value of 1
	clientIdCounter := 1
	// The above code is using the `defer` keyword to schedule the `Stop()` method of the `ticker` object
	// to be called when the surrounding function returns. This is typically used to ensure that resources
	// are properly cleaned up or released at the end of a function, regardless of how the function exits
	// (e.g. through a return statement or an error).
	defer ticker.Stop()
	for {
		select {
		// The above code is handling different types of messages received by a server in a Go program.
		case packet := <-s.rawMessages:
			message := packet.message
			sn := message.SeqNum
			messageAddr := packet.packetAddr
			connId := message.ConnID
			switch packet.message.Type {
			// The above code is handling the MsgAck case in a switch statement. It performs the following
			// actions:
			case MsgAck:
				// The above code is checking if a key `connId` exists in the `idleEpochElapsed` map. If the key
				// exists, it sets the value associated with that key to 0. If the key does not exist, it continues
				// to the next iteration of the loop.
				if _, exist := s.idleEpochElapsed[connId]; exist {
					s.idleEpochElapsed[connId] = 0
				} else {
					continue
				}

				// The above code is removing an element from a slice and deleting a key-value pair from a map.
				s.windows[connId] = removeFromSlice(s.windows[connId], sn)
				messageId := MessageId{connId: connId, seqNum: sn}
				if _, exist := s.messageBackoff[messageId]; exist {
					delete(s.messageBackoff, messageId)
				} else {
					continue
				}

				// The above code is part of a function in the Go programming language. It is iterating over a
				// slice of messages in `s.windowOutOfOrder[connId]`. For each message, it checks if the
				// `s.windows[connId]` slice is empty or if the message's sequence number is within the window size
				// and the number of unacknowledged messages is less than the maximum allowed. If the conditions
				// are met, it marshals the message into JSON, writes it to a UDP connection, and updates various
				// data structures and variables.
				var tmp []int
				for _, m := range s.windowOutOfOrder[connId] {
					if len(s.windows[connId]) == 0 || (m.SeqNum < s.windows[connId][0].SeqNum+s.params.WindowSize && len(s.windows[connId]) < s.params.MaxUnackedMessages) {
						marMessage, _ := json.Marshal(m)
						_, err := s.listener.WriteToUDP(marMessage, messageAddr)
						if err != nil {
						}

						tmp = append(tmp, m.SeqNum)
						s.messageBackoff[MessageId{m.ConnID, m.SeqNum}] = &BackOffInfo{0, 0, 0, m}
						s.windows[connId] = append(s.windows[connId], m)
						sort.Slice(s.windows[connId], func(i, j int) bool {
							return s.windows[connId][i].SeqNum < s.windows[connId][j].SeqNum
						})
					}
				}
				for _, seqNum := range tmp {
					s.windowOutOfOrder[connId] = removeFromSlice(s.windowOutOfOrder[connId], seqNum)
				}
				if connId == s.closeId && len(s.windows[connId])+len(s.windowOutOfOrder[connId]) == 0 {
					s.endConnection(s.closeId)
					s.closeId = -1

				}

				if s.isClosed && s.checkClosed() {
					return
				}

			// The above code is handling the MsgCAck case in a switch statement. It performs the following
			// actions:
			case MsgCAck:
				if _, exist := s.idleEpochElapsed[connId]; exist {
					s.idleEpochElapsed[connId] = 0

				}

				for _, m := range s.windows[connId] {
					m_id := MessageId{connId: m.ConnID, seqNum: m.SeqNum}
					if _, exist := s.messageBackoff[m_id]; exist {
						delete(s.messageBackoff, m_id)
					} else {
						continue
					}
				}

				// The above code is removing an element from a slice called `s.windows[connId]`. The element being
				// removed is determined by the value of `sn`.
				s.windows[connId] = removeFromSliceCAck(s.windows[connId], sn)

				// The above code is iterating over messages in the `windowOutOfOrder` slice for a specific
				// `connId`. It checks if the `windows` slice for that `connId` is empty or if the current
				// message's sequence number is within the window range and the number of unacknowledged messages
				// is less than the maximum allowed. If the conditions are met, it marshals the message into JSON,
				// writes it to the UDP listener, updates the `windows` slice and `messageBackoff` map, and sorts
				// the `windows` slice based on sequence number.
				for _, m := range s.windowOutOfOrder[connId] {
					if len(s.windows[connId]) == 0 || (m.SeqNum < s.windows[connId][0].SeqNum+s.params.WindowSize && len(s.windows[connId]) < s.params.MaxUnackedMessages) {
						marMessage, _ := json.Marshal(m)
						_, err := s.listener.WriteToUDP(marMessage, messageAddr)
						if err != nil {
						}
						// update unack count
						s.windows[connId] = append(s.windows[connId], m)
						s.messageBackoff[MessageId{connId: m.ConnID, seqNum: m.SeqNum}] = &BackOffInfo{0, 0, 0, m}
						sort.Slice(s.windows[connId], func(i, j int) bool {
							return s.windows[connId][i].SeqNum < s.windows[connId][j].SeqNum
						})
					}
				}

				if s.closeId != -1 && len(s.windows[s.closeId])+len(s.windowOutOfOrder[s.closeId]) == 0 {
					s.endConnection(s.closeId)
					s.closeId = -1
				}

				if s.isClosed && s.checkClosed() {
					return
				}

			case MsgConnect:
				// The above code is sending an acknowledgment message to a client. It first creates an
				// acknowledgment message using the client ID counter and sequence number. Then, it marshals the
				// acknowledgment message into JSON format. Next, it writes the marshaled acknowledgment message to
				// the UDP listener. If the write operation is successful, it initializes various data structures
				// and maps related to the client's connection. Finally, it increments the client ID counter.
				ackMessage := *NewAck(clientIdCounter, sn)
				ackMessageMar, err := json.Marshal(ackMessage)
				if err != nil {
				}

				_, err = s.listener.WriteToUDP(ackMessageMar, messageAddr)
				if err != nil {
				}
				if _, exist := s.connectionDupMap[packet.packetAddr.String()]; exist {
					continue
				}
				// initialize map
				s.addrAndSeqNum[clientIdCounter] = &Client_seq{packet.packetAddr, sn + 1}
				// message start from sn+1
				s.readCounters[clientIdCounter] = sn + 1
				s.outOfOrderList[clientIdCounter] = list.New()

				s.windows[clientIdCounter] = make([]Message, 0)
				s.windowOutOfOrder[clientIdCounter] = make([]Message, 0)
				s.messageDupMap[clientIdCounter] = make(map[int]bool)
				// mark connect this
				s.connectionDupMap[packet.packetAddr.String()] = true
				s.idleEpochElapsed[clientIdCounter] = 0
				s.isMessageSent[clientIdCounter] = false

				// add id counter
				clientIdCounter++

			case MsgData:
				// The above code is handling the reception of messages in a network communication system. Here is
				// a breakdown of what it does:

				// The above code is checking if a key `connId` exists in the `s.idleEpochElapsed` map. If the key
				// exists, it sets the value associated with that key to 0.
				if _, exist := s.idleEpochElapsed[connId]; exist {
					s.idleEpochElapsed[connId] = 0
				}

				// The above code is sending an acknowledgment message to a specific UDP address.
				ackMessage := *NewAck(connId, sn)
				ackMessageMar, err := json.Marshal(ackMessage)
				if err != nil {
				}
				_, err = s.listener.WriteToUDP(ackMessageMar, messageAddr)

				if _, exist := s.messageDupMap[connId][sn]; exist {
					continue
				}
				// mark receive, for dup reason
				s.messageDupMap[connId][sn] = true
				// make active_client become 0
				s.idleEpochElapsed[connId] = 0

				if err != nil {
				}

				// if received message match the read counter, just add it to read_list
				if sn == s.readCounters[connId] {
					s.readList.PushBack(packet.message)
					s.readCounters[connId]++
					// check if there is an element in out-of-order list that match the seq number after
					element := s.findElement(connId, s.readCounters[connId])
					// add element until there is no element in out-of-order list that match seq number
					for element != nil {
						s.readList.PushBack(element.Value.(Message))
						s.readCounters[connId]++
						element = s.findElement(connId, s.readCounters[connId])
					}
				} else {
					if s.outOfOrderList[connId] == nil {
						s.outOfOrderList[connId] = list.New()
					}
					// out-of-order element
					s.outOfOrderList[connId].PushBack(packet.message)
				}
				s.manageReadList()
			}
		// The above code is using a select statement to wait for a read request on the channel
		// `s.readRequest`. Once a read request is received, it calls the `manageReadList()` function.
		case <-s.readRequest:
			s.manageReadList()
		case sendPackage := <-s.sendMessages:
			connId := sendPackage.connId
			serverMessageInfo, ok := s.addrAndSeqNum[connId]

			if ok {
				seqNum := serverMessageInfo.serverSeq
				checksum := CalculateChecksum(connId, seqNum, len(sendPackage.payload), sendPackage.payload)
				message := *NewData(connId, seqNum, len(sendPackage.payload), sendPackage.payload, checksum)

				// check window

				if len(s.windows[connId]) == 0 || (seqNum < s.windows[connId][0].SeqNum+s.params.WindowSize && len(s.windows[connId]) < s.params.MaxUnackedMessages) {
					marMessage, _ := json.Marshal(message)

					_, err := s.listener.WriteToUDP(marMessage, serverMessageInfo.packetAddr)

					if err != nil {
					}
					s.isMessageSent[connId] = true
					s.messageBackoff[MessageId{connId, seqNum}] = &BackOffInfo{0, 0, 0, message}

					// update unack count
					s.windows[connId] = append(s.windows[connId], message)
					sort.Slice(s.windows[connId], func(i, j int) bool {
						return s.windows[connId][i].SeqNum < s.windows[connId][j].SeqNum
					})
				} else {
					s.windowOutOfOrder[connId] = append(s.windowOutOfOrder[connId], message)
					sort.Slice(s.windowOutOfOrder[connId], func(i, j int) bool {
						return s.windowOutOfOrder[connId][i].SeqNum < s.windowOutOfOrder[connId][j].SeqNum
					})
				}
				// update seq id
				s.addrAndSeqNum[connId].serverSeq++
			}

		case <-ticker.C:
			// add epoch to every client state count
			for k := range s.idleEpochElapsed {
				s.idleEpochElapsed[k]++
				// end the connection
				if s.idleEpochElapsed[k] >= s.params.EpochLimit {
					if !s.containID(s.readList, k) && s.outOfOrderList[k].Len() == 0 {
						if len(s.droppedId) == 0 {
							s.droppedId <- k
							s.endConnection(k)
						}
					}

				}
			}

			// send heartbeat
			for k, v := range s.isMessageSent {
				if !v {
					ackMessage := *NewAck(k, 0)
					ackMessageMar, err := json.Marshal(ackMessage)
					if err != nil {
					}

					_, err = s.listener.WriteToUDP(ackMessageMar, s.addrAndSeqNum[k].packetAddr)
					if err != nil {
					}

				}
				s.isMessageSent[k] = false
			}

			// backoff timer

			for k, v := range s.messageBackoff {
				if v.totalBackOff == s.params.EpochLimit {
					s.endConnection(k.connId)
					continue
				}
				if v.runningBackoff >= v.currentBackoff {
					addr := s.addrAndSeqNum[k.connId].packetAddr
					marMessage, _ := json.Marshal(v.message)
					_, err := s.listener.WriteToUDP(marMessage, addr)

					if err != nil {
					}
					if s.messageBackoff[k].currentBackoff == 0 {
						s.messageBackoff[k].currentBackoff++
					} else {
						s.messageBackoff[k].currentBackoff *= 2
					}
					if s.messageBackoff[k].currentBackoff > s.params.MaxBackOffInterval {
						s.messageBackoff[k].currentBackoff = s.params.MaxBackOffInterval
					}

					s.messageBackoff[k].runningBackoff = 0
					s.messageBackoff[k].totalBackOff++
					continue
				}
				s.messageBackoff[k].runningBackoff++
				s.messageBackoff[k].totalBackOff++
			}

		case <-s.closeMain:
			s.isClosed = true
			if s.checkClosed() {
				return
			}

		// close connection for the close client
		case connId := <-s.closeClient:
			if len(s.windows[connId])+len(s.windowOutOfOrder[connId]) == 0 {
				s.endConnection(connId)
			} else {
				s.closeId = connId
			}
		}
	}
}
```
## Potential Improvment for Server API
Current Implementation makes the server side only has one thread to control all the incoming requests,which has performance issues. To improve the performance, it is better to make each request has it own thread handeled in server side.
## Client API Implementation
Client API is almost like the server API without a hashmap to record the state of threads.
```go
type client struct {
	conn           *lspnet.UDPConn
	connId         int
	connected      chan struct{}
	rawMessages    chan Message // raw message from server
	readPayloads   chan []byte  // payload from server
	readRequest    chan struct{}
	sendPayloads   chan []byte // payload to server
	receivedRecord []Message
	sn             int // seqNum
	params         *Params
	sw             SlidingWindow
	backOffMap     map[int]*BackOff
	unsentMessages []Message
	closeRead      chan struct{}
	closeMain      chan struct{}
	closePending   chan struct{}
	isClosed       bool
}
```
Client Rountine
```go
func (c *client) mainRoutine() {
	sendingSeqNum := c.sn + 1
	receiveSeqNum := c.sn + 1
	isMessageSent := false
	idleEpochTime := 0

	connectRequest := NewConnect(c.sn)
	c.writeToServer(*connectRequest)
	isMessageSent = true
	c.sw.AddMessage(*connectRequest)
	c.backOffMap[connectRequest.SeqNum] = &BackOff{0, 0}

	ticker := time.NewTicker(time.Duration(c.params.EpochMillis) * time.Millisecond)
	defer ticker.Stop()

	heartBeat := Message{}

	for {
		select {
		// Close Connection
		case <-c.closeMain:
			c.isClosed = true
			if c.clientCloseCheck() {
				return
			}
		case <-ticker.C:
			if !isMessageSent {
				c.writeToServer(heartBeat)
			}
			isMessageSent = false
			if (idleEpochTime >= c.params.EpochLimit) && (len(c.receivedRecord) == 0) && (len(c.readPayloads) == 0) {
				c.conn.Close()
				close(c.readPayloads)
				c.closePending <- struct{}{}
				return
			}
			idleEpochTime++
			for _, message := range c.sw.window {
				seqNum := message.SeqNum
				if c.backOffMap[seqNum].epochElapsed >= c.backOffMap[seqNum].currentBackoff {
					if c.backOffMap[seqNum].currentBackoff == 0 {
						c.backOffMap[seqNum].currentBackoff = 1
					} else {
						c.backOffMap[seqNum].currentBackoff = c.backOffMap[seqNum].currentBackoff * 2
					}
					if c.backOffMap[seqNum].currentBackoff > c.params.MaxBackOffInterval {
						c.backOffMap[seqNum].currentBackoff = c.params.MaxBackOffInterval
					}
					if message.Type == MsgConnect {
						c.backOffMap[seqNum].currentBackoff = 0
					}
					c.backOffMap[seqNum].epochElapsed = 0
					c.writeToServer(message)
					isMessageSent = true
				}
				c.backOffMap[seqNum].epochElapsed++
			}
		case <-c.readRequest:
			receiveSeqNum = c.manageReceived(receiveSeqNum)

		case payload := <-c.sendPayloads:
			checksum := CalculateChecksum(c.ConnID(), sendingSeqNum, len(payload), payload)
			message := NewData(c.connId, sendingSeqNum, len(payload), payload, checksum)
			sendingSeqNum++
			c.unsentMessages = append(c.unsentMessages, *message)
			if c.manageToSend() == true {
				isMessageSent = true
			}

		case message := <-c.rawMessages:
			idleEpochTime = 0
			switch message.Type {
			case MsgConnect:
				continue
			case MsgAck:
				if c.connId == -1 {
					c.connId = message.ConnID
					heartBeat = *NewAck(c.connId, 0)
					c.connected <- struct{}{}
				}
				c.sw.RemoveSeqNum(message.SeqNum)
				if c.isClosed {
					if c.clientCloseCheck() {
						return
					}
				}
				delete(c.backOffMap, message.SeqNum)
				if c.manageToSend() == true {
					isMessageSent = true
				}
			case MsgCAck:
				if c.connId == -1 {
					c.connId = message.ConnID
					heartBeat = *NewAck(c.connId, 0)
					c.connected <- struct{}{}
				}
				c.sw.RemoveBeforeSeqNum(message.SeqNum)
				if c.isClosed {
					if c.clientCloseCheck() {
						return
					}
				}
				for key := range c.backOffMap {
					if key <= message.SeqNum {
						delete(c.backOffMap, key)
					}
				}
				if c.manageToSend() == true {
					isMessageSent = true
				}
			case MsgData:
				if message.SeqNum >= receiveSeqNum {
					isDuplicate := false
					for _, m := range c.receivedRecord {
						if message.SeqNum == m.SeqNum {
							isDuplicate = true
							break
						}
					}
					if !isDuplicate {
						c.receivedRecord = append(c.receivedRecord, message)
					}
				}
				receiveSeqNum = c.manageReceived(receiveSeqNum)
				ackMessage := *NewAck(c.connId, message.SeqNum)
				c.writeToServer(ackMessage)
			}
		}
	}
}
```

# Part 2 - Bitcoin Miner based on the LSP
## Scheduler Implementation
The main focus on this part is to write the a scheduler to assign the work for multiple machines. To minimize mean response time for all client requests, I assigned jobs evenly across the machines.
```go
// scheduler
// aims to minimize mean response time for all client requests.
// requests that arrive earlier have some sort of priority.
func (srv *server) scheduler() {
	for {
		select {
		case minerId := <-srv.freeMinerChan:
			// A miner joins the server.
			srv.freeMiners = append(srv.freeMiners, minerId)

		case index := <-srv.closeMinerChan:
			// remove the miner from freeMiners
			srv.freeMiners = append(srv.freeMiners[:index], srv.freeMiners[index+1:]...)

		case reassignJob := <-srv.reassignJobChan:
			// find the request from requestList
			index := ByRemainingJobs(srv.requestList).findByRequestId(reassignJob.requestId)
			if index == -1 {
				// the corresponding client has been disconnected
				continue
			}

			// place the job back to list
			job := srv.busyMiners[reassignJob.minerId].job
			srv.requestList[index].remainingJobs = append(srv.requestList[index].remainingJobs, job)
			sort.Sort(ByRemainingJobs(srv.requestList))

			// remove the job from the previous miner
			delete(srv.busyMiners, reassignJob.minerId)

		case clientId := <-srv.closeClientChan:
			// remove the requests from requestList
			/*
				requestIds := make([]int, initialListLen)
				for reqId, reqMap := range srv.requestInfo {
					if reqMap.clientId == clientId {
						requestIds = append(requestIds, reqId)
					}
				}
				for _, requestId := range requestIds {
					index := ByRemainingJobs(srv.requestList).findByRequestId(requestId)
					srv.requestList = append(srv.requestList[:index], srv.requestList[index+1:]...)
					delete(srv.requestInfo, requestId)
				}
			*/
			requestId := -1
			for reqId, reqMap := range srv.requestInfo {
				if reqMap.clientId == clientId {
					requestId = reqId
					break
				}
			}
			if requestId == -1 {
				continue
			}
			srv.removeRequest(requestId)

		case req := <-srv.requestChan:
			// A client sends a request to the server.
			msg := req.message
			// For each client request, it splits the request into more manageable-sized jobs
			var jobs []job
			current := msg.Lower
			for current+chunkSize < msg.Upper {
				jobs = append(jobs, job{current, current + chunkSize})
				current += chunkSize
			}
			jobs = append(jobs, job{current, msg.Upper})

			srv.requestList = append(srv.requestList, clientRequest{req.requestId, jobs})
			srv.requestInfo[req.requestId] = requestMap{req.clientId, msg.Data, make([]result, initialListLen), len(jobs)}

		case res := <-srv.resultChan:
			// append the res
			requestId := srv.busyMiners[res.minerId].requestId
			if _, ok := srv.requestInfo[requestId]; !ok {
				continue
			}
			// Get a copy of the requestMap
			reqMap := srv.requestInfo[requestId]
			reqMap.results = append(reqMap.results, res.result)
			srv.requestInfo[requestId] = reqMap

			delete(srv.busyMiners, res.minerId)
			// Work should be divided roughly evenly across workers. Try to minimize the idle time of each worker.
			srv.freeMiners = append(srv.freeMiners, res.minerId)

			// The server waits for each worker to respond before generating and sending the final result back to the client.
			// Once the miner exhausts all possible nonces, it determines the least hash value and
			// its corresponding nonce and sends back the final result: [Result minHash nonce]
			if len(srv.requestInfo[requestId].results) == srv.requestInfo[requestId].totalNum {
				// sort the results
				requestMap := srv.requestInfo[requestId]
				sort.Sort(ByHash(requestMap.results))

				result := requestMap.results[0]
				// send the result back to the client
				srv.write2client(requestMap.clientId, bitcoin.NewResult(result.hash, result.nonce))

				// remove the request from requestList
				srv.removeRequest(requestId)
			}
		default:
			if len(srv.requestList) == 0 {
				continue
			}
			if len(srv.requestList[0].remainingJobs) == 0 {
				continue
			}
			// If there are no available miners left, the server should wait for a new miner to join
			// before reassigning the old miner’s job.
			if len(srv.freeMiners) == 0 {
				continue
			}
			// LOAD BALANCING POLICY:
			// FIFO for miner
			miner := srv.freeMiners[0]
			srv.freeMiners = srv.freeMiners[1:]

			// assign the job to the miner
			req := srv.requestList[0]
			job := req.remainingJobs[0]

			// farms them out to its available miners (it is up to you to choose a suitable maximum job size).
			// The server should be "fair" in the way it distributes tasks.
			srv.busyMiners[miner] = jobInfo{req.requestId, job}

			// send the job to the miner
			srv.write2client(miner, bitcoin.NewRequest(srv.requestInfo[req.requestId].message, job.lower, job.upper))

			srv.requestList[0].remainingJobs = srv.requestList[0].remainingJobs[1:]
			sort.Sort(ByRemainingJobs(srv.requestList))
		}
	}
}

```












