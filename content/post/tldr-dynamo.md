+++
Description = "The things you sacrifice for scalability"
date = "2017-02-26T16:03:52-05:00"
title = "TL;DR: Dynamo: Amazon’s Highly Available Key-value Store"
draft = false
+++

At [SOSP 2007](http://www.sosp2007.org/), Amazon presented *[Dynamo: Amazon’s Highly Available Key-value Store](https://s3.amazonaws.com/AllThingsDistributed/sosp/amazon-dynamo-sosp2007.pdf)*. It's the thing that "launched a thousand NoSQL databases" -- along with Google's Bigtable, it kicked off the NoSQL movement of the late 2000s and early 2010s.

[Last time](../tldr-chubby), we looked at Google's Chubby, which provides a key-value store replicated using Paxos for strong consistency. In many ways, Dynamo is the opposite of Chubby.

Dynamo -- not to be confused with Dynamo*DB*, which was inspired by Dynamo's design -- is a key-value store designed to be scalable and highly-available. In [CAP theorem](https://jvns.ca/blog/2016/10/21/consistency-vs-availability/) terms, Chubby is a CP system, whereas Dynamo is on the other end, falling squarely within the category of AP systems.

The core rationale that drives Dynamo's design principles is the observation that the *availability* of a system directly correlates to the number of customers served. On the other hand, imperfect *consistency* can usually be masked, and resolved in the backend without the customer knowing about it. Informed by this main idea, Dynamo aggressively optimizes for availability, and makes a few very interesting tradeoffs.

The Dynamo design is highly influential. It inspired a large number of NoSQL databases (sometimes lumped together as the category of *Dynamo systems*), like [Cassandra](https://cassandra.apache.org/), [Riak](http://basho.com/products/), and [Voldemort](http://www.project-voldemort.com/voldemort/) -- not to mention Amazon's own [DynamoDB](https://aws.amazon.com/dynamodb/). The core architecture has even influenced projects like [Ringpop](http://uber.github.io/ringpop/) (a load balancer) and [LeoFS](https://github.com/leo-project/leofs) (a distributed filesystem).

In this post I'll spend some time talking about the core design of Dynamo (occasionally cross-referencing some open-source Dynamo systems), and how it influenced the software industry.

(Usual disclaimer that I'll be focusing on making things digestible, and not strictly correct.)

## Design goals
Like I mentioned above, the goal of Dynamo is, above all else, to be highly available. In summary, they came up with these fundamental properties:

* The system should be **incrementally scalable**. You should be able to just throw a machine into the system, and see proportional improvement.
* There should be no **leader** process. A leader is a single point of failure, which bottlenecks the system at some point, and makes it harder to scale the system. If every node is the same, that complexity goes away.
* Data should be **optimistically replicated**, which means that instead of incurring write-time costs to ensure correctness throughout the system, inconsistencies should be resolved at some other time.

Ultimately, Dynamo was built around six core techniques:

* **Consistent hashing** to shard data between nodes, and make it easy to add new nodes.
* A **gossip protocol** for to keep track of the cluster state, and make the system "always-writeable" by using **hinted handoff**.
* Replicate writes to a **sloppy quorum** of other nodes in the system, instead of a strict majority quorum like Paxos.
* Since there are no write-time guarantees that nodes agree on values, resolve potential conflicts using other mechanisms:
  * Use **vector clocks** to keep track of value history, and reconcile divergent histories at read time.
  * In the background, use **Merkle trees** to resolve conflicts between different nodes.

## Data model
In the interest of scalability, Dynamo exposes only an extremely barebones interface: you can `get()` a key, and you can `put()` to a key. Values are treated as opaque blobs. In other words, Dynamo provides massive scalability and not much more.

A simple key-value data model is great because because it's [embarrassingly parallel](https://en.wikipedia.org/wiki/Embarrassingly_parallel): since a single-key operation doesn't depend anything other than its value, you can split up the workload across arbitrarily many processes easily. But this is a far cry from the traditional relational database that it replaces, that most people are used to.

In a standard RDBMS, you have a bunch of tuples, organized into a bunch of relations. On top of that, you get features like foreign keys, joins, a rich query language, and transactions with correctness guarantees. But these features are very expensive to implement in a system that's potentially spread across multiple machines in multiple continents, especially when Dynamo was first being developed, more than a decade ago.

A consequence of this is that although Dynamo is supposed to replace relational databases, what it really does is push up those responsibilities up to the application. Using a Dynamo system, if application developers need to make sure that their data is written correctly into the system, or correctly references things, they have to write their own logic to do it.

Subsequent Dynamo systems are a bit better in this regard; Cassandra, Riak and DynamoDB all have a real type system. (They still lack things like foreign keys, but it's more understandable because referential integrity across a distributed system is *really* slow.)

## Consistent hashing
You probably already know how a hash table works: you have an array of buckets; to figure out which bucket a key goes into, you pass it through a hash function, and the result modulo the size of the table is the index of the bucket that you want. Hashing is a great way to map arbitrary keys to a finite range.

But you incur the cost of occasionally having to rebuild the entire data structure whenever you need to resize it. This is fine for small in-memory hash tables, but with a multi-terabyte store like Dynamo, it's disastrous, and could mean hours of downtime.

**Consistent hashing** is one way to tackle this issue. The idea is that instead of a one-to-one mapping between index and bucket, a contiguous range of indices maps to a bucket. Whenever you want to add a bucket, you just split an existing index range.

The entire range is treated as a *ring*, wrapping around after the maximum index. Every bucket is assigned a random point on the ring. To figure out which bucket a key should go into, you hash your key to figure out the index, and then you *walk clockwise* along the ring until you reach a bucket -- that bucket is where the key should go.

(Astute readers will note that the worst case performance with a naive implementation is when you have a single bucket assigned to the biggest possible index, and you have to traverse the entire ring to get to it. In real life, hash rings are usually implemented as a balanced binary tree, to avoid this problem.)

Awesome, this means that consistent hashing can be done *incrementally*: a new bucket would get randomly placed onto the ring as normal, which means that it takes over a portion of some existing bucket. If there are `k` keys and `n` buckets, this means that on average you only need to shift around `k/n` keys.

### Virtual nodes

There are two main problems with the naive consistent hashing algorithm that we just looked at, and they both have to do with how the key-space is split up. First, even though we expect a uniform distribution, we probably won't see a very even distribution [unless we have a lot of nodes](https://en.wikipedia.org/wiki/Law_of_large_numbers). Second, we can't control how much data a machine stores: a beefy machine might get the same "share" of keys as a weaker machine. A psuedorandom number generator is a capricious and fickle thing.

Dynamo deals with both of these issues very nicely. A physical machine actually isn't just a single node; it's treated as multiple *virtual* nodes, configurable on a per-machine basis. This makes it more likely for the keys to be distributed evenly, and a machine with more resources can easily be configured to store more data, just by increasing the number of virtual nodes it gets allocated. Pretty clever.

The paper doesn't actually talk about this, but one problem with consistent hashing is that it doesn't explicitly deal with *hotspots*. If a specific range is being accessed more often than others, there isn't a way to split up just that range. You add more machines and hope for the best. A Cassandra [blog post](http://www.datastax.com/dev/blog/we-shall-have-order) suggests to choose your sharding key carefully.

## Replication
To replicate updates, Dynamo uses something called a **sloppy quorum**. This is in contrast to something like [Paxos](../tldr-chubby), which relies on a strict quorum's consensus to make progress. Instead of a strict majority, Dynamo lets you configure the number of nodes that need to acknowledge a read or a write before the client receives a response. It means that you don't necessarily get the strict guarantees that Paxos provide, but you can get better latency and availability.

The main problem is that since a sloppy quorum isn't a strict majority, your data can and will *diverge*: it's possible for two concurrent writes to the same key to be accepted by non-overlapping sets of nodes. Dynamo allows this, and resolves these conflicts at some other time. More on this later.

An interesting trick to increase availability is **hinted handoff**: when a node is unreachable, another node can accept writes on its behalf. The write is then kept in a local buffer, and sent out once the destination node is reachable again. This is what makes Dynamo *always-writeable*: even in the extreme case where only a single node is alive, write requests will still get accepted, and eventually processed.

## Gossip protocol
Let's say you have a cluster of a bunch of nodes that need to work together. One node goes down. How does everyone else find out? (Remember, we don't have a leader node that can serve as the source of truth.)

The simplest way to do this is to have every node maintain [heartbeats][1] with every other node. When a node goes down, it'll stop sending out heartbeats, and everyone else will find out immediately. But then `O(n^2)` messages get sent every tick -- a ridiculously high amount, and obviously not feasible in any sizable cluster.

Instead, Dynamo uses a **gossip protocol**. Every node keeps track of what it thinks the cluster looks like, i.e. which nodes are reachable, what key ranges they're responsible for, and so on. (This is basically a copy of the hash ring.) Every tick, a node tries to contact one other node at random. If the other node is alive, the two nodes then exchange information, and both now see the same state.

This means that any new events will eventually propagate through the system. If there are no more changes to the cluster, then it is guaranteed that the system will eventually converge on the same state.

## Conflicts
This is where things get exciting. Sloppy quorum means that multiple conflicting values for the same key can exist in the system, and must be resolved somehow. There are few tricks that Dynamo uses.

### Vector clocks
Time is a tricky thing.

On a single machine, all you need to know about is the absolute or **wall clock** time: suppose you perform a write to key `k` with timestamp `t1`, and then perform another write to `k` with timestamp `t2`. Since `t2 > t1`, the second write must have been newer than the first write, and therefore the database can safely overwrite the original value. (*Note:* This is a bit of an oversimplification that assumes the wall clock timestamp is monotonically increasing. In Linux, [`gettimeofday`](http://man7.org/linux/man-pages/man2/gettimeofday.2.html) doesn't actually provide this guarantee, but [`clock_gettime` with the `CLOCK_MONOTONIC` clock](http://man7.org/linux/man-pages/man2/clock_gettime.2.html) is supposed to.)

In a distributed system, this assumption doesn't hold true. The problem is **clock skew** -- different clocks tend to run at different rates, so you can't assume that time `t` on node `a` happened before time `t + 1` on node `b`. The most practical techniques that help with synchronizing clocks, like [NTP](https://en.wikipedia.org/wiki/Network_Time_Protocol), still don't let you make the guarantee that every clock in a distributed system is synchronized at all times. So, without special hardware like GPS units and atomic clocks, just using wall clock timestamps is not enough.

So -- without a means of tight synchronization -- Dynamo uses something called a **vector clock**. Basically, objects given a *version* based on knowledge of causality (the **happens-before relation**). Divergences can occur, and are resolved at read-time, either automatically by the server, or manually by the client. A very simplified example:

1. Node `A` serves a write to key `k`, with value `foo`. It assigns it a version of (`A`, 1). This write gets replicated to node `B`.
1. A network partition occurs. `A` and `B` can't talk to each other.
1. Node `A` serves a write to key `k`, with value `bar`. It assigns it a version of (`A`, 2). It can't replicate it to node `B`, but it gets stored in a hinted handoff buffer somewhere.
1. Node `B` sees a write to key `k`, with value `baz`. It assigns it a version of (`B`, 2). It can't replicate it to node `A`, but it gets stored in a hinted handoff buffer somewhere.
1. Node `A` serves a read to key `k`. It sees (`A`, 1) and (`A`, 2), but it can *automatically* resolve this since it knows (`A`, 2) is newer, so it returns `bar`.
1. The network heals. Node `A` and `B` can talk to each other again.
1. Either node sees a read to key `k`. It sees the same key with different versions (`A`, 2) and (`B`, 2), but it *doesn't* know which one is newer. It returns both, and tells the client to figure it out themselves and write the newer version back into the system.

### Merkle trees
If a copy of a range falls significantly behind others, it might take a very long time to resolve conflicts using just vector clocks. It would be nice to be able to automatically resolve some conflicts in the background. To do this, we need to be able to quickly compare two copies of a range, and figure out exactly which parts are different.

A range can contain a lot of data. Naively splitting up the entire range for checksums not very infeasible; there's simply too much data to be transferred.

Instead, Dynamo uses **Merkle trees** to compare replicas of a range. A Merkle tree is a binary tree of hashes, where each internal node is the hash of its two children, and each leaf node is a hash of a portion of the original data. Comparing Merkle trees is conceptually simple:

1. Compare the root hashes of both trees.
1. If they're equal, stop.
1. Recurse on the left and right children.

Ultimately, this means that replicas know exactly which parts of the range are different, but the amount of data exchanged is minimized.

### CRDTs
All this complicated conflict resolution logic, and it turns out there's an even simpler way to resolve conflicts: just don't ever have them. Use a **conflict-free replicated datatype**.

The main idea is this: if you can model the data as a series of *commutative* changes, i.e. they can be applied in *any* order and have the same result, then you don't need any ordering guarantees in the system.

A shopping cart is a very good example: adding one item `A` and then adding one item `B` can be done from any nodes and in any order. (Removing from the shopping cart is modeled as a negative add.) The idea that any two nodes that have received the same set of updates will see the same state is called **strong eventual consistency**.

Riak has a few [built-in CRDTs](http://docs.basho.com/riak/kv/2.2.0/developing/data-types/).

### Last write wins
Unfortunately, CRDTs aren't as easy as "just add commutative operations". In practice, excepting very simple data structures, it's generally pretty hard to model data as a CRDT. In many cases, it's too much effort to do that, and client-side resolution is considered good enough.

Actually, in many cases, it's even worse. Because it's still pretty hard to reason about vector clocks, Dynamo systems generally offer ways to resolve these conflicts automatically on the server side. Riak and Cassandra deployments often use a simple policy: last write wins, based on the wall-clock timestamp.

Remember everything I mentioned above about clock skew? 😱

Wall-clock LWW is a really good way to lose data. If conflicting writes happen at around the same time, you're *basically* flipping a coin on which write to throw away. [Much](https://aphyr.com/posts/299-the-trouble-with-timestamps) [has](https://blog.discordapp.com/how-discord-stores-billions-of-messages-7fa6ec7ee4c7#.5uonqtiyc) [been](https://issues.apache.org/jira/browse/CASSANDRA-580) [said](https://aphyr.com/posts/285-jepsen-riak) [about](http://queue.acm.org/detail.cfm?id=2610533) [this](http://basho.com/posts/technical/clocks-are-bad-or-welcome-to-distributed-systems/).

## Conclusion
I've mostly skipped a things that I thought were unnecessary, but I hope I've given a decent overview on how Dynamo (and systems inspired by it) works.

Dynamo is genuinely a brilliant piece of engineering. At Amazon's scale, Dynamo's optimizations for high availability probably made it a critical piece of infrastructure. But at the same time, in order to achieve its scalability and availability goals, it required significant sacrifices in terms of data model, features, and more importantly, safety.

Although Dynamo-like NoSQL systems were once poised to take over the world (of databases), the pendulum is swinging back. More and more engineering teams are going the "safer" route of sharding a relational database like PostgreSQL or MySQL. Furthermore, the success of "NewSQL" systems like [F1](https://research.google.com/pubs/pub41344.html) and [H-Store](http://hstore.cs.brown.edu/)/[VoltDB](https://www.voltdb.com/overviews) says that scalability and strong consistency are a false dichotomy -- it is in fact possible to have both the scalability of a NoSQL and the features and ACID correctness of a relational database.  I hope to talk more about them in future posts.

[1]: https://en.wikipedia.org/wiki/Heartbeat_(computing)
