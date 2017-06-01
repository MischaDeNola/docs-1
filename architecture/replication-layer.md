---
title: Architecture&#58; Replication Layer
summary: 
toc: false
---

CockroachDB's replication layer not only copies data between nodes, but also implements our consensus algorithm, which is the integral component that enables us to provide consistency across a distributed cluster.

In relationship to other layers in CockroachDB, the Distribution Layer:

- Receives requests from and sends responses to the Distribution Layer.
- Writes accepted requests to the Storage Layer.

<div id="toc"></div>

## Overview

The idea behind high availability (including CockroachDB's Multi-Active Availability) is that your database can tolerate a certain number of nodes going offline without interrupting service to your application. With CockroachDB, if your cluster contains 3 or more nodes, availability is ensured by replicating each range at least 3 times. This allows for one replica to become inaccessible without compromising the consistency of the system.

{{site.data.alerts.callout_info}}Because CockroachDB relies on consensus replication, it does not work with 2-node deployments, which would not let you reach quorum.{{site.data.alerts.end}}

The reason CockroachDB can tolerate 1 failure with 3 replicas is that to enforce consistency, it uses "consensus replication," which means that a quorum of replicas must agree on any changes to the range before it's committed. With 2 replicas missing, you couldn't achieve consensus––and to preserve consistency CockroachDB would halt forward progress.

The number of failures that can be tolerated is equal to *(Replication factor - 1)/2*. For example, with 3x replication, one failure can be tolerated; with 5x replication, two failures, and so on. You can control the replication factor per-table in your cluster's [Replication Zones](stable/configure-replication-zones.html).

## Components

### Raft

Raft is a consensus protocol––an algorithm which makes sure that your data is safely stored on multiple machines, and that those machines agree on the current state even if some of them are temporarily disconnected.

Raft organizes all nodes that contain a replica of a range into a group––unsurprisingly called a Raft Group.

Raft elects a relatively long-lived leader which must be involved to propose commands. It heartbeats followers periodically and keeps their logs replicated. In the absence of heartbeats, followers become candidates after randomized election timeouts and proceed to hold new leader elections. Cockroach weights random timeouts such that the replicas with shorter round trip times to peers are more likely to hold elections first (not implemented yet). 

Once a node receives `BatchRequests` for a range it contains, it converts those KV operations into Raft commands. Those commands are proposed to the Raft group leader––which is what makes it ideal for the [Lease Holder](#leases) and the Raft leader to be one in the same; it reduces network roundtrips if the Lease Holder receiving the requests can simply propose the Raft commands to itself, rather than communicating them to another node.

Proposed Raft commands are proposed to have them serialized through quorum voting into a coherent distributed log.

#### Logs

The log voted on by the Raft group contains an ordered set of commands that a quorum of replicas acknowledged and can be replayed to bring the node from a past state to its current state.

This log also lets nodes that temporarily went offline to be "caught up" to the current state without needing to receive a copy of the existing data in the form of a snapshot.

### Snapshots

Each replica can be "snapshotted", which copies all of its data as of a specific timestamp (which is available because of [MVCC](storage-layer.html#mvcc)). This snapshot can be sent to other nodes during a rebalance event to expedite replication.

After receiving a snapshot, the node gets it up to date by playing all of the actions from the Raft group's committed log that have occurred since the snapshot was taken.

### Leases

A single node in the Raft group acts as the lease holder, and is the only node that can serve reads or propose writes to the Raft group leader (both actions are received as `BatchRequests` from [`DistSender`](distribution-layer.html#distsender)) related to keys in the range.

CockroachDB attempts to elect a leaseholder who is also the Raft group leader, which has the benefit of improving the speed of reads drastically by bypassing Raft; these reads don't need consensus, since a quorum of nodes already agreed with the lease holder as to the status of the node after the write.

If there is no lease holder, any node receiving a request will attempt to become to Lease holder for the range.

#### Epoch-Based Leases (Table Data)

To manage leases for table data, CockroachDB implements a notion of "epochs," which are defined as the period between a node joining a cluster and a node disconnecting from a cluster. When the node disconnects, the epoch is considered changed, and the node immediately loses all of it leases.

This mechanism lets us avoid tracking leases for every range, which eliminates a substantial amount of traffic we would otherwise incur. Instead, we assume leases don't expire until a node loses connection.

#### Expiration-Based Leases (Meta & System Ranges)

Your table's meta and system ranges (detailed in the Distribution Layer) are treated as normal key-value data, and therefore have Leases, as well. However, instead of using epochs, they have an expiration-based lease. These leases simply expire at a particular timestamp (typically a few seconds)––however, as long as the node continues proposing Raft commands, it continues to extend the expiration of the lease. If it doesn't, the next node containing a replica of the range that tries to read from or write to the range will become the lease holder.

Since reads to these ranges can potentially bypass Raft, a new lease holder will ascertain that its timestamp cache does not report timestamps smaller than the previous Lease holder's, ensuring it's compatible with reads which may have occurred on the former lease holder. This is accomplished by letting leases enter a stasis period (which is just the expiration minus the maximum clock offset) before the actual expiration of the lease, so that all the next lease holder has to do is set the low water mark of the timestamp cache to its new lease's start time.

As a lease enters its stasis period, no more reads or writes are served, which is undesirable. However, this would only happen in practice if a node became unavailable. In almost all practical situations, no unavailability results since leases are usually long-lived (and/or eagerly extended, which can avoid the stasis period) or proactively transferred away from the lease holder, which can also avoid the stasis period by promising not to serve any further reads until the next lease goes into effect.

#### Co-location with Raft leadership

The range lease is completely separate from Raft leadership, and so without further efforts, Raft leadership and the Range lease might not be held by the same Replica. Since it's expensive to not have these two roles colocated (the lease holder has to forward each proposal to the leader, adding costly RPC round-trips), each lease renewal or transfer also attempts to collocate them. In practice, that means that the mismatch is rare and self-corrects quickly.

### Membership Changes: Rebalance/Repair

Whenever there are changes to a cluster's number of nodes, the members of Raft groups change––to ensure optimal performance, replicas need to be rebalanced. What that looks like varies depending on whether the membership change is nodes being added or going offline.

**Nodes added**: The new node communicates information about itself to other nodes, indicating that it has space available. The cluster then rebalances some replicas onto the new node based on its zone configs, that control replication with table-level granularity.

**Nodes going offline**: If a member of a Raft group ceases to respond, after 5 minutes, the cluster begins to rebalance by replicating the data the downed node held onto nodes with available space.

#### Rebalancing Replicas

When CockroachDB detects a membership change, ultimately, replicas are moved between nodes.

This is achieved by taking using a snapshot of a replica from the lease-holding node, and then sending the data to another node over [gRPC](distribution-layer.html#grpc). After the transfer has been completed, the node with the new replica joins the Raft group for the range; it then detects that its latest timestamp is behind the most recent entries in the Raft log and it replays all of the actions in the Raft log on itself.

## Interactions with Other Layers

The Replication Layer receives requests from its and other nodes' `DistSender`, which it accepts if it is the leaseholder or errors if it's not. These KV requests are then turned into Raft commands.

The Replication layer sends `BatchResponses` back to the Distribution Layer's `DistSender`.

Accepted Raft commands are written to the Raft log and ultimately stored on disk through the Storage Layer.
