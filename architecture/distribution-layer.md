---
title: Architecture&#58; Distribution Layer
summary: 
toc: false
---

Despite your data being replicated among many nodes in your cluster––which might not be in the same datacenter or even the same continent––CockroachDB always presents a unified view of your data through the technology in its Distribution Layer.

In relationship to other layers in CockroachDB, the Distribution Layer:

- Receives requests from the Transaction Layer on the same node.
- Identifies which nodes should receive the request, and then sends the request to the proper node's Replication Layer.

<div id="toc"></div>

## Overview

CockroachDB is a distributed, fault-tolerant database, which means that it spreads data among many nodes in your cluster. However, all of the data still needs to be easily accessible. To achieve this Cockroach uses a monolithic sorted map, which is a sorted key-value range that describes all of the data in your cluster.

CockroachDB implements an ordered map to enable:
  - **Simple lookups**: Because we identify which nodes are responsible for certain portions of the data, queries are able to quickly locate where to find the data they want.
  - **Efficient scans**: By defining the order of data, it's easy to find data within a particular range during a scan.

### Monolithic Sorted Map Structure

The monolithic sorted map is comprised of two fundamental elements:

- System keys, which include **meta ranges** that describe the locations of data in your cluster (among many other cluster-wide and local data elements)
- **Table data** keys, which store your cluster's data

#### Meta Ranges

To address data, the location of all ranges in your cluster are stored in a two-level index at the beginning of your key-space, known as meta ranges. Importantly, these are treated mostly like normal ranges and are accessed and replicated just like other elements of your cluster's KV data. 

The two levels of the index are:

1. `meta1` contains the address for the nodes containing the `meta2` ranges. The first address listed is for the leaseholder (which is the node responsible for reads and writes for ranges of data and is discussed in greater detail in the Replication Layer), followed by the other nodes to which the data is replicated.

    For example, imagine we have an alphabetically

    ~~~
    # Points to meta2 range for keys [A-M)
    meta1/M -> node1:26257, node2:26257, node3:26257

    # Points to meta2 range for keys [M-Z]
    meta1/maxKey -> node4:26257, node5:26257, node6:26257
    ~~~

    Importantly, every node has information on where to locate the `meta1` range (known as its Range Descriptor, detailed below), and the range is never split.

2. `meta2` contains addresses for the nodes "responsible" for each range in the cluster. 

    For example:

    ~~~
    # Contains [A-G)
    meta2/G -> node1:26257, node2:26257, node3:26257

    # Contains [G-M)
    meta2/M -> node1:26257, node2:26257, node3:26257

    #Contains [M-Z)
    meta2/Z -> node4:26257, node5:26257, node6:26257

    #Contains [Z-maxKey)
    meta2/maxKey->  node4:26257, node5:26257, node6:26257
    ~~~

Each node stores a cache of the `meta2` range. Whenever it accesses a key on another node, it caches the value in its `meta2` range. If the node's the `meta2` cache misses, the value is evicted and the new set of addresses for the key it's accessing is updated by performing a regular read from the `meta2` range.

#### Table Data

After the node's meta-ranges come the key-value pairs it stored, which are structured like this:

~~~
/<table_id>/<index_id>/<indexed column values> -> STORING/COVERING column values
~~~

The table itself is stored with an `index_id` of 1 for its `PRIMARY KEY` columns, with the rest of the columns in the table considered as stored/covered columns.

This key-value data is broken up into 64MiB sections of contiguous key-space known as ranges. This size represents a sweet spot for us between a size that's small enough to move quickly between nodes, but large enough to store meaningfully contiguous set of data whose keys are more likely to be accessed together. However, not all ranges on a node are contiguous with one another. 

### Using the Monolithic Sorted Map

When a node receives a request, its looks at the Meta Ranges to find out which node it needs to route the request to by comparing the keys in the request to the keys in its `meta2` range.

These meta ranges are heavily cached, so this is normally handled without having to send an RPC to the node actually containing the `meta2` ranges.

The node then sends those KV operations to the leaseholder identified in the `meta2` range. However, it's possible that the data moved, in which case the node that no longer has the information replies to the requesting node where it's now located. In this case we go back to the `meta2` range to get more up-to-date information and try again.

## Technical Details & Components

### gRPC

gRPC is the software nodes use to communicate with one another. Because the Distribution Layer is the first layer to communicate with other nodes, CockroachDB implements gRPC here.

gRPC requires inputs and outputs to be formatted as protocol buffers (protobufs). To leverage gRPC, CockroachDB implements a protocol-buffer-based API defined in [`api.proto`](https://github.com/cockroachdb/cockroach/blob/295ae3bfba88e96a1fd641fd515998e2804bc7ef/pkg/roachpb/api.proto).

You can also find much more information about gRPC at http://www.grpc.io/docs/guides/

### BatchRequest

All KV operation requests are bundled into a [protobuf](https://en.wikipedia.org/wiki/Protocol_Buffers)––known as a `BatchRequest`––the destination of this batch is identified in the `BatchRequest` header, as well as a pointer to the request's transaction record. (On the other side, when a node is replying to a BatchRequest, it uses a protobuf––`BatchResponse`.)

This BatchRequest is also what's used to send requests between nodes using gRPC, which accepts and sends protocol buffers.

### DistSender

The gateway/coordinating node's `DistSender` receives `BatchRequest`s from its own `TxnCoordSender`. `DistSender` is then responsible for breaking up `BatchRequests` and routing a new set of `BatchRequests` to the nodes it identifies contain the data using its `meta2` ranges.  It will use the cache to send the request to the lease holder, but it's also prepared to try the other replicas, in order of "proximity". The replica that the cache says is the leaseholder is simply moved to the front of the list of replicas to be tried and then an RPC is sent to all of them, in order.

Requests received by a non-lease holder fail with an error pointing at the replica's last known lease holder. These requests are retried transparently with the updated lease by the gateway node and never reach the client.

As nodes begin replying to these commands, `DistSender` also aggregates the results in preparation for returning them to the client.

### Range Descriptors

All ranges in CockroachDB contain metadata, which is known as a "range descriptor":

- A sequential RangeID
- The keyspace (i.e., the set of keys) the range contains
- The addresses of nodes containing replicas of the range, with its leaseholder (which is responsible for its reads and writes) in the first position

Whenever there are membership changes to a range's Raft group (discussed in more detail in the Replication Layer) or its leaseholder, the range updates its `meta2` values so it can be located by any node in the system.

### Range Splits

By default, CockroachDB attempts to keep ranges/replicas at 64MiB; this number attempts to find a balance between being large enough to contain a meaningful amount of data, while being small enough to allow rapid replication between nodes.

Once a range reaches that limit we split it into two 32 MiB ranges (composed of contiguous key spaces). Once this happens, the node creates a new Raft group containing all of the same members as the range that was split. The fact that there are now two ranges also means that there is a transaction that updates `meta2` with the new keyspace boundaries, as well as the addresses of the nodes from the Range Descriptor.

## Technical Interactions with Other Layers

The Distribution Layer's `DistSender` receives `BatchRequests` from its own node's `TxnCoordSender`, housed in the Transaction Layer.

The Distribution Layer routes `BatchRequests` to nodes containing ranges of data, which is ultimately routed to the Raft group leader or Lease Holder, which are handled in the Replication Layer.
