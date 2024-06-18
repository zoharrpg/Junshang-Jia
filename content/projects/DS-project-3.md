+++
title="Multi-User Dungeon-CMUD"
date=2023-12-19

[taxonomies]
categories = ["CMU-15640","Distributed System"]
[extra]
toc = true

+++

# Overview
CMU-15640 Distributed Systems Project 3<br>
In CMUD, players progress through the ranks of CMU computer science by mining coins,
improving their hardware, and solving hard problems. The game is a Multi-User Dungeon
(MUD), meaning that all players inhabit the same virtual world and interact with it through
a text terminal.


Unlike a traditional MUD with a single application server, CMUD uses the globally dis-
tributed architecture shown in Figure 1. Players run the CMUD client app on their own
machine, which connects to a nearby (low ping) backend storage server. The CMUD client
is stateless, converting the player’s inputs into queries against the backend storage server
and printing out the results. Different backend storage servers “sync” with each other (i.e.,
exchange data in the background) to maintain a single consistent global state, so that all
users see the same virtual world, including each others’ actions.

The backend storage system itself implements a simple key-value store for string keys and
values, similar to Project 0. Specifically, it supports operations to Get and Put key value
pairs, plus an operation to List all key-value pairs starting with a given prefix. Since this
API is very simple, all application logic happens on the client, unlike the classic three-tier
architecture with separate application and backend servers; see Figure 2.
The advantage of this architecture is that it simplifies app development and deployment:
a cloud provider can run the same generic backend storage system for many apps, while
individual app developers only need to code and deploy their app’s client, not their own
servers.

[Source Code Link](https://github.com/zoharrpg/CMUD)<br>
{{ render_link(path="15640-doc/p3_23.pdf",text = "Write-Up") }}

# Actor Model
This game will use Actor model to make users connect game simultaneously and share information. The actor model means each user have its own state about itself, and directly send message to other users.
{{render_image(path="/static/15640-doc/p3-pic1.png")}}

# System Architecture
{{render_image(path="/static/15640-doc/p3-pic2.png")}}
{{render_image(path="/static/15640-doc/p3-pic3.png")}}
{{render_image(path="/static/15640-doc/p3-pic4.png")}}

# System Implementation

## Concurrent Mailbox
Use mutex and conditional veriables to implement unbounded FIFO queue
```go
package actor

import (
	"sync"
)

// Mailbox
// a thread-safe unbounded FIFO queue.
//
// You can think of Mailbox like a Go channel with an infinite buffer.
//
// Mailbox is only exported outside the actor package for use in tests;
// we do not expect you to use it, just implement it.
type Mailbox struct {
	mu      sync.Mutex
	message []any
	closed  bool
	cond    *sync.Cond
}

// NewMailbox Returns a new mailbox that is ready for use.
func NewMailbox() *Mailbox {
	mailbox := &Mailbox{}
	mailbox.cond = sync.NewCond(&mailbox.mu)
	return mailbox
}

// Push
// Pushes message onto the end of the mailbox's FIFO queue.
//
// This function should NOT block.
//
// If mailbox.Close() has already been called, this may ignore the message. It still should NOT block.
//
// Note: message is not a literal actor message; it is an ActorSystem wrapper around a marshalled actor message.
func (mailbox *Mailbox) Push(message any) {
	mailbox.mu.Lock()
	defer mailbox.mu.Unlock()
	if !mailbox.closed {
		mailbox.message = append(mailbox.message, message)
		mailbox.cond.Signal()
	}
}

// Pop
// Pops a message from the front of the mailbox's FIFO queue,
//
// blocking until a message is available.
//
// If mailbox.Close() is called (either before or during a Pop() call), this should unblock and return (nil, false).
// Otherwise, it should return (message, true).
func (mailbox *Mailbox) Pop() (message any, ok bool) {
	mailbox.mu.Lock()
	defer mailbox.mu.Unlock()

	for len(mailbox.message) == 0 && !mailbox.closed {
		mailbox.cond.Wait()
	}
	if len(mailbox.message) > 0 && !mailbox.closed {
		message = mailbox.message[0]
		mailbox.message = mailbox.message[1:]
		ok = true
		return message, ok
	}

	return nil, false
}

// Close
// Closes the mailbox, causing future Pop() calls to return (nil, false)
// and terminating any goroutines running in the background.
//
// If Close() has already been called, this may exhibit undefined behavior, including blocking indefinitely.
func (mailbox *Mailbox) Close() {
	mailbox.mu.Lock()
	defer mailbox.mu.Unlock()

	if !mailbox.closed {
		mailbox.closed = true
		mailbox.cond.Broadcast()
	}
}

```
## Actor Model Define Type

Define Actor Message type and server side implementation
```go
package kvserver

import (
	"encoding/gob"
	"fmt"
	"github.com/cmu440/actor"
	"strings"
	"time"
)

// Implement your queryActor in this file.
func init() {
	gob.Register(GetResult{})
	gob.Register(Init{})
	gob.Register(ListResult{})
	gob.Register(SynMsg{})
	gob.Register(SynSignal{})
	gob.Register(MGet{})
	gob.Register(MPut{})
	gob.Register(MList{})
	gob.Register(PutResult{})
	gob.Register(NotifyNewServer{})
}

// queryActor represents an actor that handles GET, PUT, and LIST requests.
type queryActor struct {
	ActorsInfo  []*actor.ActorRef
	ActorSystem *actor.ActorSystem
	Context     *actor.ActorContext
	Logs        map[string]MPut
	Me          int
	RemoteInfo  [][]*actor.ActorRef
	Store       map[string]StoreValue
}

// StoreValue is the value stored in the store
type StoreValue struct {
	Sender    *actor.ActorRef
	Timestamp int64 //millisecond resolution
	Value     string
}

// MGet is the message type for GET requests.
type MGet struct {
	Key    string
	Sender *actor.ActorRef
}

// MPut is the message type for PUT requests.
type MPut struct {
	Key       string
	Sender    *actor.ActorRef
	Timestamp int64
	Value     string
}

// MList is the message type for LIST requests.
type MList struct {
	Prefix string
	Sender *actor.ActorRef
}

// ListResult is the message type for LIST responses.
type ListResult struct {
	Pair map[string]string
}

// GetResult is the message type for GET responses.
type GetResult struct {
	Ok    bool
	Value string
}

// PutResult is the message type for PUT responses.
type PutResult struct {
}

// Init is the message type for initializing the queryActor
type Init struct {
	ActorsInfo []*actor.ActorRef
	RemoteInfo [][]*actor.ActorRef
	Me         int
}

// SynSignal is the message type for triggering synchronization
type SynSignal struct {
}

// SynMsg is the message type for synchronization
type SynMsg struct {
	Data map[string]MPut
}

// NotifyNewServer is the message type for notifying that a new server came online
type NotifyNewServer struct {
	Refs []*actor.ActorRef
}

// "Constructor" for queryActors, used in ActorSystem.StartActor.
func newQueryActor(context *actor.ActorContext) actor.Actor {
	return &queryActor{
		ActorsInfo: make([]*actor.ActorRef, 0),
		Context:    context,
		Logs:       make(map[string]MPut),
		Me:         -1,
		RemoteInfo: make([][]*actor.ActorRef, 0),
		Store:      make(map[string]StoreValue),
	}
}

// OnMessage implements actor.Actor.OnMessage.
// Sync Strategy:
//  1. When a new server joins, it will send a NotifyNewServer message to all servers.
//  2. When a server receives a NotifyNewServer message, it will send a SynMsg message to the new server.
//  3. When a server receives a SynMsg message, it will update its own store and logs.
//  4. Every 100ms, a server will send a SynSignal message to itself
//  5. When a server receives a SynSignal message, it will send a SynMsg message to all servers.
func (actor *queryActor) OnMessage(message any) error {
	switch m := message.(type) {
	case NotifyNewServer:
		actor.RemoteInfo = append(actor.RemoteInfo, m.Refs)
		logs := make(map[string]MPut)

		for k, v := range actor.Store {
			logs[k] = MPut{Key: k, Value: v.Value, Sender: v.Sender, Timestamp: v.Timestamp}
		}

		for _, ref := range m.Refs {
			actor.Context.Tell(ref, SynMsg{Data: logs})
		}

	case SynSignal:
		for index, a := range actor.ActorsInfo {
			if index != actor.Me {
				actor.Context.Tell(a, SynMsg{Data: actor.Logs})
			}
		}
		for _, remote := range actor.RemoteInfo {
			actor.Context.Tell(remote[0], SynMsg{Data: actor.Logs})
		}
		actor.Logs = make(map[string]MPut)
		actor.Context.TellAfter(actor.ActorsInfo[actor.Me], SynSignal{}, 100*time.Millisecond)

	case SynMsg:
		for _, data := range m.Data {
			doPut := false

			if v, ok := actor.Store[data.Key]; ok {
				if data.Timestamp > v.Timestamp {
					doPut = true
				} else if data.Timestamp == v.Timestamp && data.Sender.Uid() < v.Sender.Uid() {
					doPut = true
				}
			} else {
				doPut = true
			}

			if doPut {
				actor.Store[data.Key] = StoreValue{data.Sender, data.Timestamp, data.Value}
				actor.Logs[data.Key] = data
			}
		}

	case Init:
		actor.ActorsInfo = append(actor.ActorsInfo, m.ActorsInfo...)
		actor.RemoteInfo = append(actor.RemoteInfo, m.RemoteInfo...)
		actor.Me = m.Me
		actor.Context.Tell(actor.ActorsInfo[actor.Me], SynSignal{})

	case MGet:
		v, exist := actor.Store[m.Key]
		result := GetResult{Value: v.Value, Ok: exist}
		actor.Context.Tell(m.Sender, result)

	case MPut:
		m.Timestamp = time.Now().UnixMilli()
		doPut := false
		if v, ok := actor.Store[m.Key]; ok {
			if m.Timestamp > v.Timestamp {
				doPut = true
			} else if m.Timestamp == v.Timestamp && m.Sender.Uid() < v.Sender.Uid() {
				doPut = true
			}
		} else {
			doPut = true
		}
		if doPut {
			actor.Store[m.Key] = StoreValue{m.Sender, m.Timestamp, m.Value}
			actor.Logs[m.Key] = m
		}
		result := PutResult{}
		actor.Context.Tell(m.Sender, result)

	case MList:
		result := ListResult{Pair: make(map[string]string)}
		for k, v := range actor.Store {
			if strings.HasPrefix(k, m.Prefix) {
				result.Pair[k] = v.Value
			}
		}
		actor.Context.Tell(m.Sender, result)

	default:
		return fmt.Errorf("Unexpected counterActor message type: %T", m)
	}
	return nil
}
```

## RPC handler
Used RPC functions for each player
```go
package actor

import (
	"net/rpc"
)

type RemoteTellArgs struct {
	Ref  *ActorRef
	Mars []byte
}

// remoteTellReply represents the reply for the remoteTell RPC.
type RemoteTellReply struct {
	// You can define fields here if needed.
}

//var mailBox *Mailbox
//var mux sync.Mutex

// Calls system.tellFromRemote(ref, mars) on the remote ActorSystem listening
// on ref.Address.
//
// This function should NOT wait for a reply from the remote system before
// returning, to allow sending multiple messages in a row more quickly.
// It should ensure that messages are delivered in-order to the remote system.
// (You may assume that remoteTell is not called multiple times
// concurrently with the same ref.Address).
func remoteTell(client *rpc.Client, ref *ActorRef, mars []byte) {
	// TODO (3B): implement this!

	args := &RemoteTellArgs{Ref: ref, Mars: mars}
	//mailBox.Push(args)

	//go func() {
	//	mux.Lock()
	//	defer mux.Unlock()
	//	next, ok := mailBox.Pop()
	//	if !ok {
	//
	//		return
	//	}
	client.Go("RemoteTellHandler.RemoteTell", args, nil, nil)
	//}()
	//if err != nil {
	//	fmt.Printf("Error calling RemoteTell RPC: %v\n", err)
	//	// Handle the error as needed.
	//}
}

// Registers an RPC handler on server for remoteTell calls to system.
//
// You do not need to start the server's listening on the network;
// just register a handler struct that handles remoteTell RPCs by calling
// system.tellFromRemote(ref, mars).
func registerRemoteTells(system *ActorSystem, server *rpc.Server) error {
	// TODO (3B): implement this!
	//mailBox = NewMailbox()
	handler := &RemoteTellHandler{
		ActorSys: system,
	}

	err := server.RegisterName("RemoteTellHandler", handler)
	if err != nil {
		return err
	}

	return nil
}

type RemoteTellHandler struct {
	ActorSys *ActorSystem
}

// TODO (3B): implement your remoteTell RPC handler below!

// RemoteTell handles the remoteTell RPC.
func (h *RemoteTellHandler) RemoteTell(args *RemoteTellArgs, reply *RemoteTellReply) error {
	// Call system.tellFromRemote(ref, mars) using the provided arguments.
	h.ActorSys.tellFromRemote(args.Ref, args.Mars)

	return nil
}
```
