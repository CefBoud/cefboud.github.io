---
title: "Gossip and Consensus: Using Serf and Raft to Build a Kafka-esque System"
author: cef
date: 2025-02-23
categories: [Technical Writing, Distributed Systems]
tags: [Open Source, Raft, Serf, Apache Kafka]
render_with_liquid: false
description: We play with Serf and Raft to implement cluster mode for Monkafka.
---

## Intro
I've embarked on the foolish journey of writing my own implementation of Apache Kafka: [MonKafka](https://cefboud.com/posts/monkafka/). Things started off smoothly, and I was able to implement a subset of the protocol, getting the official Java clients to communicate with my Monkafka broker—i.e., produce and consume messages.
However, as I've come to realize, this was just the easiest part of the endeavor.

Distributing the workload across multiple nodes and building a scalable, resilient, and consistent distributed system is the name of the game. After all, the primary goal of running Kafka is handling massive amounts of data through a distributed, highly scalable, elastic, and fault-tolerant log.

Kafka originally relied on Zookeeper for distributed state management and cluster coordination. This changed with the advent of Kraft, which consolidates cluster management into Kafka itself, using Raft instead of relying on Zookeeper. Why the shift? I invite you to check out the [Zookeeper Connection section](https://cefboud.com/posts/exploring-kafka-internals/) of a previous post where I took a look at some of the Apache Kafka codebase.

I chose Golang for my implementation, for reasons (Hehe!). My work is largely inspired by the tremendous [Jocko](https://github.com/travisjeffery/jocko), which in turn builds on the incredible work done by HashiCorp with Consul, a paragon distributed sophistication written in Go. The two core components of Consul are Serf and Raft, and these are the giants upon whose shoulders my modest implementation timidly stands.

### The Problem

The problem we're trying to solve is ensuring a coherent, consistent state across all brokers. If a user wants to create a topic, all nodes need to agree on that before the user's request is acknowledged. Similarly, if we update or remove a topic, all nodes must update their local state accordingly.

Additionally, we need to detect node failures and respond appropriately. We must also allow our cluster to scale by reacting to new nodes joining.

## Serf: Even Computers Like to Gossip

In modern cloud infrastructure, it's common to have a fleet of machines working together to perform tasks. In this post, we're building up to a Kafka cluster for data streaming, which could comprise over a hundred nodes. We can also imagine a load balancer routing traffic between dozens of servers. But how do we determine if a server has joined or left the cluster? How do we propagate information or trigger events (like deployments or updates) to all servers?

The broader problem I'm pointing to is that, with a significant number of nodes in our infrastructure, it becomes difficult to maintain a clear view of cluster membership (healthy vs. unresponsive nodes) while accounting for temporary (and inevitable?) network failures. 

For example, imagine Subgroup A can reach Subgroup B but not C. Subgroup C can reach B, but not A. If we rely on some notion of leadership, losing connectivity with the leader could result in disconnection, even if the other nodes are still reachable.

Enter gossip protocols. The principle behind them is peer-to-peer communication, propagating information much like how humans gossip. In our previous example, even if A can't reach C, C would still learn about A through B.

Serf is lightweight, using only 5MB to 10MB of memory, and communicates over UDP. [Quoting from the documentation](https://github.com/hashicorp/serf/blob/a445e0b8e9997239ac58adc0c29215d9473eb908/docs/intro/index.html.markdown):


> Serf solves three major problems:
* Membership: Serf maintains cluster membership lists and is able to execute custom handler scripts when that membership changes. For example, Serf can maintain the list of web servers for a load balancer and notify that load balancer whenever a node comes online or goes offline.
* Failure detection and recovery: Serf automatically detects failed nodes within seconds, notifies the rest of the cluster, and executes handler scripts allowing you to handle these events. Serf will attempt to recover failed nodes by reconnecting to them periodically. 
* Custom event propagation: Serf can broadcast custom events and queries to the cluster. These can be used to trigger deploys, propagate configuration, etc. Events are simply fire-and-forget broadcast, and Serf makes a best effort to deliver messages in the face of offline nodes or network partitions. Queries provide a simple realtime request/response mechanism.

The Serf API is beautifully designed. When initializing a Serf node, a channel for all events is made available. This means that whenever a node joins, fails, or leaves, we get notified. Custom events can also be triggered, allowing for various tasks to be performed. These custom events carry a payload that the event handler can use.

Each node has tags that carry its metadata. In our case, the tags store details about the broker, such as its Kafka and Raft ports, as well as its node ID.

Here is what a Serf event handler can look like:

```go
func (b *Broker) handleSerfEvent() {
	for {
		select {
		case e := <-b.SerfEventCh:
			switch e.EventType() {
			case serf.EventMemberJoin:
				b.handleSerfMemberJoin(e.(serf.MemberEvent))
			case serf.EventMemberFailed:
				b.handleSerfMemberFailed(e.(serf.MemberEvent))
			case serf.EventMemberReap, serf.EventMemberLeave:
				b.handleSerfMemberLeft(e.(serf.MemberEvent))
			}
		case <-b.ShutDownSignal:
			b.ShutdownSerf()
		}
	}
}
```

We notice that there are multiple event types. `EventMemberFailed` is triggered when a member doesn't respond to direct and indirect probes and fails to exit the suspicious state. The member is not yet considered out of the cluster (imagine a DC outage). Only after a `reconnect_timeout` (default of 24 hours) does the member get ousted, and the `EventMemberReap` event is triggered.

`EventMemberLeave` is triggered when a member willingly leaves the cluster, typically during a graceful shutdown.

If we want to perform an action across some or all of the nodes, we can create a custom event with a payload and then write a handler for it.

## Raft: Gossip isn't enough, We need Consensus

From the previous section, we might think that Serf provides all we need to manage our distributed state, but by design, Serf isn't consistent and offers no consistency guarantees. If we want to create a topic across multiple nodes, we can't simply issue an event and expect it to be eventually handled. We need confirmation that the topic was reflected into a state shared between all nodes, and this is where Raft comes into the picture.

The problem we want to solve is the following: we need a distributed, consistent state across all our brokers. For example, if we add a topic with 3 partitions, all brokers must agree on that synchronously. Furthermore, these partitions should be distributed across different brokers to balance the load (that's the whole point). Raft was built specifically to solve this. The way I like to think about it is as an append-only file (log) with the same entries across all nodes. The interesting thing about this is that we can construct any state just from that log. Databases use a similar approach: in MySQL, it's called a binlog, and in PostgreSQL, it's a WAL (Write-Ahead Log). Create a table, insert a row, update another, and delete one more, and we end up with a fully-functioning database.

Raft works similarly: we append entries and construct our shared state from them. We just need a function that can update a state using an log entry. An entry is referred to as a command, and we "apply" it to update a Finite State Machine (FSM).

In our simple implementation, the FSM holds the topics and partitions information. When a topic creation request is made, a `CreateTopic` command is appended to the log:

```go
// CommandType is a raft log command type
type CommandType int

// Commands types that can be applied to the raft log to change the state machine
const (
	AddTopic CommandType = iota
	RemoveTopic
	AddPartition
	RemovePartition
)

// Command represents a command type with its payload
type Command struct {
	Kind    CommandType
	Payload json.RawMessage
}

// ApplyCommand to append a command to the raft log
func (kf *FSM) ApplyCommand(cmd Command) error {
	log.Debug("ApplyCommand %v", cmd.Kind)
	switch cmd.Kind {
	case AddTopic:
		var topic types.Topic
		err := json.Unmarshal(cmd.Payload, &topic)
		if err != nil {
			return fmt.Errorf("could not parse topic: %s", err)
		}
		log.Debug("Raft ApplyCommand AddTopic: %+v", topic)
		kf.StoreTopic(topic)
	case AddPartition:
		var partition types.PartitionState
		err := json.Unmarshal(cmd.Payload, &partition)
		if err != nil {
			return fmt.Errorf("could not parse partition command: %s", err)
		}
		log.Debug("Raft ApplyCommand AddPartition: %+v", partition)
		err = kf.StorePartition(partition)
		if err != nil {
			return fmt.Errorf("error applying partition %+v command: %s", partition, err)
		}
	default:
		return fmt.Errorf("unknown command type: %##v", cmd.Kind)
	}
	return nil
}


// CreateTopicPartitions creates a new topic with its partition by appending them to the raft log.
func (b *Broker) CreateTopicPartitions(name string, numPartitions uint32, configs map[string]string) error {
	resp, err := b.AppendRaftEntry(raft.AddTopic, types.Topic{Name: name, Configs: configs})
	if err != nil {
		log.Error("Error append topic to raft log %v", err)
	}
	log.Debug("raft AddTopic entry for %v with configs [%+v]. Result: %v ", name, configs, resp)

	nodes, err := b.GetClusterNodes()
	if err != nil || len(nodes) < 1 {
		return fmt.Errorf("CreateTopicPartitions GetClusterNodes error: %v", err)
	}

	for i := uint32(0); i < numPartitions; i++ {
		// pick a leader randomly
		leaderID := nodes[rand.Intn(len(nodes))].NodeID
		partition := types.PartitionState{
			Topic:          name,
			PartitionIndex: i,
			LeaderID:       leaderID,
		}
		resp, err = b.AppendRaftEntry(raft.AddPartition, partition)
		if err != nil {
			return fmt.Errorf("Error appending partition to raft log %v", err)
		}
		log.Debug("raft AddPartition entry for %v. Result: %v ", name, resp)
	}
	return nil
}
```

HashiCorp's Raft library almost makes things too easy. When we want to update the distributed state (FSM), we apply a command—i.e., append an entry to the Raft log. On each node in the Raft cluster, the `ApplyCommand` function will run and update its local state, ensuring consistency with the local states of all other nodes.

In our case, the load is shared across brokers by assigning a leader randomly. A more sophisticated approach would assign the leader based on the existing load to achieve better balance across the cluster.


## The Raft and Serf Dance
Serf and Raft need to play nice with each other. Serf, as we said, is used for membership and fault tolerance. Whenever a node joins through the Serf Gossip, we need to add it to the Raft cluster, when a node fails or leaves, we remove it from the cluster. This latter part, of course, comes with multiple caveats. If we want to ensure data replication and durability, we can't allow the cluster to operate with fewer than a specified number of replicas, but the current implementation isn't there yet.

Another point to keep in mind, is that the first node needs to bootstrap both the Serf and Raft cluster and this needs to be explicitly indicated using a `bootstrap` flag. Subsequent nodes join the existing cluster by specifying the Serf address to connect to. 

```go
func (b *Broker) Startup() {
    // ...
	err = b.SetupRaft()
	if err != nil {
		log.Panic("Raft Setup failed: %v", err)
	}
	err = b.SetupSerf()
	if err != nil {
		log.Panic("Serf Setup failed: %v", err)
	}

	go b.handleSerfEvent()
    // ...
}
```
The Broker's `Startup` function sets up the Raft and Serf cluster and then we can see `handleSerfEvent` that we encountered above follow.

Inside `SetupRaft`, we can find:

```go
	if b.Config.Bootstrap {
		logging.Info("bootstrapping raft with nodeID %v ....", nodeID)
		hasState, err := hraft.HasExistingState(store, store, snapshots)
		if err != nil {
			return err
		}
		logging.Info("bootstrapping hasState %v", hasState)
		if !hasState {
			future := b.Raft.BootstrapCluster(hraft.Configuration{
				Servers: []hraft.Server{
					{
						ID:      hraft.ServerID(nodeID),
						Address: transport.LocalAddr(),
					},
				},
			})
			if err := future.Error(); err != nil {
				logging.Error(" bootstrap cluster error: %s", err)
			}
		}
	}
```

If the node is configured to bootstrap and it doesn't have previous state -an existing Raft log file -,it will `Bootstrap` its own cluster.


```go
func (b *Broker) handleSerfMemberJoin(event serf.MemberEvent) error {
    // ...
	for raftAddr, m := range newMembers {
		log.Info("handleSerfMemberJoin: adding voter to the raft cluster with addr %s", raftAddr)
		b.Raft.AddVoter(raft.ServerID(m.Tags["raft_server_id"]), raft.ServerAddress(m.Tags["raft_addr"]), 0, 0).Error()
	}
    // ...
}

```

Inside `handleSerfMemberJoin`, the Raft leader will add Serf member that joined the Serf cluster to Raft cluster. Notice how the Raft information (server id and address) are given as Serf tags (metadata).


## Conclusion 
Inspired by Consul and Jocko, we explored how Serf and Raft can be used to manage distributed state across multiple nodes. Serf, using its Gossip protocol, handles membership management and fault tolerance, while Raft ensures consistency across the system through its append-only log.

We're just scratching the surface here. There are numerous edge cases I didn't account for in my implementation, and I've only laid the first building blocks. The next step is a more thorough implementation, complete with proper distributed system testing and some more.

This is my first time implementing something like this, and it's been a lot of fun. More to come.