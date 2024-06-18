+++
title="The Raft Consensus Algorithm"
date=2023-12-18

[taxonomies]
categories = ["CMU-15640","Distributed System"]
[extra]
toc = true

+++

# Overview
CMU-15640 Distributed Systems Project 2<br>
A replicated service (e.g., key/value database) achieves fault tolerance by storing copies of its data on multiple replicas. Replication allows the service to continue operating even if some of its replicas experience failures (crashes or a broken/flaky network). The challenge is that failures may cause the replicas to hold differing copies of the data.

One protocol to ensure all of these copies of the data are consistent across all non-faulty replicas is Raft. Raft implements a replicated state machine by sequencing client requests into a log, and ensuring that the replicas agree on the contents and ordering of the log entries. Each replica asynchronously applies the client requests in the order they appear in the replica’s log of the service’s state. If a replica fails and later recovers, Raft takes care of bringing the log of the recovered replica up to date. Raft will continue to operate as long as at least a quorum of replicas is alive and able to communicate. If a quorum is not available, Raft will stop making progress but will resume as soon as a quorum becomes available.

[Source Code Link](https://github.com/zoharrpg/Raft)<br>
{{ render_link(path="15640-doc/p2_23.pdf",text = "Write-Up") }}
# Raft Implementation
## Introduction
In this project, the Raft algorithm uses Remote Procedure Call(RPC) to send and receive requests from other servers. This algorithm does involve a lot code, but it is really hard to debug and has a many deadlocks and performance issue when I implemented. State transitation and vote process are the headest parts in this project.
{{render_image(path="/static/15640-doc/p2-pic1.png") }}

## Main Data Structs
Each server needs to send heartbeats to the leader and send appendEntries message to append logs. Also, there is a term concept to avoid duplicate Leader in the distributed systems
```go
type Raft struct {
	mux   sync.Mutex       // Lock to protect shared access to this peer's state
	peers []*rpc.ClientEnd // RPC end points of all peers
	me    int              // this peer's index into peers[]
	// You are expected to create reasonably clear log files before asking a
	// debugging question on Edstem or OH. Use of this logger is optional, and
	// you are free to remove it completely.
	logger *log.Logger // We provide you with a separate logger per peer.

	ElectionTimeout int // heartbeat timeout

	Interval int

	currentTerm int // Term for each serve

	state State // State of the server

	votedFor int // vote information

	applyCh chan ApplyCommand

	hearthbeat_signal chan int

	step_down_signal chan int // down level the state of the server

	logs []LogStruct

	commitIndex int

	lastApplied int

	// leader part
	nextIndex []int

	matchIndex []int
}
```
Message structs are used for RPC
```go
// RequestVoteArgs
// ===============
//
// Example RequestVote RPC arguments structure.
//
// # Please note: Field names must start with capital letters!
type RequestVoteArgs struct {
	Term         int
	CandidateId  int
	LastLogIndex int
	LastLogTerm  int
	// TODO - Your data here (2A, 2B)
}

// RequestVoteReply
// ================
//
// Example RequestVote RPC reply structure.
//
// # Please note: Field names must start with capital letters!
// The RequestVoteReply type is used to store the response to a request for vote in a distributed
// consensus algorithm.
// @property {int} Term - The Term property represents the current term of the candidate requesting the
// vote. It is an integer value that is used to keep track of the current election term.
// @property {bool} VoteGranted - A boolean value indicating whether the vote has been granted or not.
type RequestVoteReply struct {
	Term        int
	VoteGranted bool

	// TODO - Your data here (2A)
}

// The `AppendEntriesArgs` type represents the arguments for an append entries RPC call in a
// distributed system.
// @property {int} Term - The current term of the leader sending the AppendEntries RPC.
// @property {int} LeaderId - The LeaderId property represents the unique identifier of the leader in a
// distributed system.
// @property {int} PrevLogIndex - PrevLogIndex is the index of the log entry immediately preceding the
// new entries being sent in the AppendEntries RPC.
// @property {int} PrevLogTerm - PrevLogTerm is the term of the log entry immediately preceding the new
// log entries being sent in the AppendEntries RPC.
// @property {[]LogStruct} Entries - The `Entries` property is a slice of `LogStruct` objects. It
// represents the log entries that the leader wants to append to the follower's log.
// @property {int} LeaderCommit - LeaderCommit is the index of the highest log entry known to be
// committed by the leader.
type AppendEntriesArgs struct {
	Term         int
	LeaderId     int
	PrevLogIndex int
	PrevLogTerm  int
	Entries      []LogStruct
	LeaderCommit int
}

// The AppendEntriesReply type represents a response to an append entries request in the Go programming
// language.
// @property {int} Term - The "Term" property represents the term of the leader that sent the
// AppendEntries RPC (Remote Procedure Call). It is an integer value that is used to ensure that
// followers are up to date and do not fall behind in terms of the leader's log.
// @property {bool} Success - A boolean value indicating whether the AppendEntries RPC request was
// successful or not. If Success is true, it means that the log entries were successfully appended to
// the receiver's log. If Success is false, it means that the receiver rejected the log entries due to
// inconsistencies or other reasons.
type AppendEntriesReply struct {
	Term    int
	Success bool
}
```

## Vote Process
Candidates can request votes from other followers. The hardest part is handling the signals from other followers. My approach is using the RPC to get the vote information. If the votes is greater than half of number of the servers, the candidate become the leader. Handling signals is very difficule here, because it is hard to know whether the follower server vote for the candidate. The channel may wait for a very long time for the vote process. Another problem is that there is only channel, and there are multiple senders with one receiver. Due to the multiple senders, it is hard to manage and close channel. My approach is create a new channel for each vote process and make it manage by go runtime, which is not a good way. 
```go
func (rf *Raft) ProcessCandidateState() {
	rf.initState(CANDIDATE)
	//rf.Clear()

	rf.setVotedFor(rf.me)
	t := rf.getTerm() + 1
	rf.setTerm(t)
	lastIndex, lastTerm := rf.getLastLogInfo()
	vote_count := 1
	vote_signal := make(chan bool)
	for peer := range rf.peers {
		if peer != rf.me {
			reply := RequestVoteReply{}
			args := RequestVoteArgs{CandidateId: rf.me, Term: t, LastLogIndex: lastIndex, LastLogTerm: lastTerm}

			go rf.sendRequestVote(peer, &args, &reply, vote_signal)

		}
	}

	for {
		//rf.logger.Printf("Election pid is %d Vote Pending\n", pid)
		select {
		case term := <-rf.step_down_signal:
			rf.setTerm(term)
			rf.setVotedFor(-1)

			rf.logger.Printf("Candidate %d to follower", rf.me)
			go rf.ProcessFollowerState()
			return

		case isVoted := <-vote_signal:
			if isVoted {

				vote_count++

				if vote_count > len(rf.peers)/2 {
					go rf.ProcessLeader()
					return

					//rf.logger.Println("Leader selected")

				}

			}
		case <-time.After(time.Duration(rf.ElectionTimeout) * time.Millisecond):
			rf.logger.Printf("Candidate %d reelection", rf.me)

			go rf.ProcessCandidateState()

			return

		}

	}

}

// The above code is a method in the Raft struct that is used to send a RequestVote RPC to a specific
// server. It takes in the server index, the arguments for the RequestVote RPC, a reply struct to store
// the response, and a vote_signal channel to signal whether the vote was granted or not.

func (rf *Raft) sendRequestVote(server int, args *RequestVoteArgs, reply *RequestVoteReply, vote_signal chan bool) {
	ok := rf.peers[server].Call("Raft.RequestVote", args, reply)
	//rf.logger.Println(*reply == RequestVoteReply{})
	//rf.logger.Println("ok state is ", ok)

	if ok {
		if reply.Term > rf.getTerm() {
			rf.setVotedFor(-1)
			rf.setState(FOLLOWER)
			rf.step_down_signal <- reply.Term
			return
		}

		vote_signal <- reply.VoteGranted
	}

}
```
## Potential Improvement
My design for vote process is not a good design, and the problem is huge. A better practice for handling signal is **Close the Channel from Sender side**. Multiple senders and on receiver is not a good design. A better way is to use dynamic Select handle signal from each follower.

Another approach is waiting for the all the vote result from all other servers.

# Detial Raft Explanation Video
{{ youtube(id="IujMVjKvWP4") }}








