---
title: Architecture&#58; Overview
summary: 
toc: false
---

These documents describe the internal processes of the `cockroach` process. Because CockroachDB represents a new type of database––one that blends consistency with scalability––understanding its architecture might enlighten you as to its values.

These documents outline CockroachDB's underlying architecture. While you don't need to read any of these documents to successfully deploy or develop against CockroachDB, they can help grasp how we've created a distributed, transactional database.

And lastly, a caveat: these documents detail how CockroachDB is built, but do not explain how *you* should architect an application using CockroachDB.

## Goals

CockroachDB was designed in service of the following goals:

- Make life easier for humans. This means being low-touch and highly automated for operators and simple to reason about for developers.
- Offer industry-leading consistency, even on massively scaled deployments. This means enabling distributed transactions, as well as removing the pain of eventual consistency issues and stale reads.
- Create an always-on database that accepts reads and writes on all nodes without generating conflicts.
- Allow flexible deployment on any cloud, without tying you to any platform.
- Support familiar tools for working with relational data (i.e., SQL).

## Glossary

### Terms

It's helpful to understand a few terms when reading our architecture documentation.

Term | Definition
-----|-----------
**Cluster** | Your CockroachDB deployment, which acts as a single logical application that contains one or more databases.
**Node** | An individual machine running CockroachDB. Many nodes join together to create your cluster.
**Range** | A set of sorted, contiguous data from your cluster.
**Replicas** | Copies of your ranges, which are stored on at least 3 nodes to ensure survivability. Each node will contain many replicas.

### Concepts

CockroachDB heavily relies on the following concepts, so being familiar with them will help you understand what our architecture achieves.

Term | Definition
-----|-----------
**Consistency** | CockroachDB uses "consistency" in both the sense of [ACID semantics](https://en.wikipedia.org/wiki/Consistency_(database_systems)) and the [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem), albeit less formally than either definition. What we try to express with this term is that your data should be anomaly-free, and each node in a distributed cluster consistently receives the same reads.
**Consensus** | When a range receives a write, a quorum of nodes containing a replica of the range acknowledge the write. This means your data is safely stored and a majority of nodes agree on the database's current state, even if some of the nodes are offline.<br/><br/>When a write *doesn't* achieve consensus, forward progress halts to maintain consistency within the cluster.
**Replication** | Replication involves creating and distributing copies of data, as well as ensuring copies remain consistent. However, there are multiple types of replication: namely, synchronous and asynchronous.<br/><br/>Synchronous replication requires all writes to propagate to each copy of the data before being considered committed. To ensure consistency with your data, this is the kind of replication CockroachDB uses.<br/><br/>Aysnchronous replication only requires a single node to receive the write to be considered committed; it's propogated to each copy of the data after the fact. This is more or less equivalent to "eventual consistency", which was popularized by NoSQL databases. This method of replication is likely to cause anomalies and loss of data.
**Transactions** | A set of operations performed on your database that satisfy the requirements of [ACID semantics](https://en.wikipedia.org/wiki/Database_transaction). This is a crucial component for a consistent system to ensure developers can trust the data in their database.
**Multi-Active Availability** | A consensus-based notion of high availability that lets all nodes in a cluster receive reads and writes. This is similar to active-active replication, but circumvents the possibility of nodes having conflicting information.

## Overview

CockroachDB starts running on machines with two commands:

- `cockroach start` with a `--join` flag for all of the initial nodes in the cluster, so the process knows all of other machines it can communicate with
- `cockroach init` to initialize the entire cluster

Once the `cockroach` process is running, developers interact with CockroachDB through a SQL API, which we've modeled after PostgreSQL. Thanks to the symmetrical behavior of all nodes, you can send SQL requests to any of them; this makes CockroachDB really easy to integrate with load balancers.

After receiving SQL RPCs, nodes converts them into operations that work with our distributed key-value store. As these RPCs start filling your cluster with data, CockroachDB algorithmically starts distributing your data among your nodes, breaking the data up into 64MiB chunks that we call ranges. Each range is replicated to at least 3 nodes to ensure survivability. This way, if nodes go down, you still have copies of the data which can be used for reads and writes, as well as replicating the data to other nodes.

If a node receives a read or write request it can't directly serve, it simply finds the node that can handle the request, and communicates with it. This way you don't need to know where your data lives, CockroachDB tracks it for you, and enables symmetric behavior for each node.

Any changes made to the data in a range rely on a consensus algorithm to ensure a majority of its replicas agree to commit the change, ensuring industry-leading isolation guarantees and providing your application consistent reads, regardless of which node you communicate with.

Ultimately, data is written to and read from disk using an efficient storage engine, which is able to keep track of the data's timestamp. This has the benefit of letting us support the SQL standard `AS OF SYSTEM TIME` clause, letting you find historical data for a period of time.

However, while that high-level overview gives you a notion of what CockroachDB does, looking at how the `cockroach` process operates on each of these needs will give you much greater understanding of our architecture.

### Layers

At the highest level, CockroachDB converts clients' SQL statements into key-value (KV) data, which is distributed among nodes and written to disk in RocksDB. Our architecture is the process by which we accomplish that, which is manifested as a number of layers that interact with those directly above and below it as relatively opaque services.

Level | Layer Name | Purpose
------|------------|--------
1 | SQL | Translate client SQL queries to KV operations.
2 | Transactional | Allow atomic changes to multiple KV entries
3 | Distribution | Present replicated KV ranges as a single entity
4 | Replication | Consistently and synchronously replicate KV ranges across many nodes. This layer also enables consistent reads via leases.
5 | Storage | Write and read KV data on disk.
