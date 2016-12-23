+++
date = "2016-12-23T00:16:00-08:00"
draft = false
title = "TL;DR: Chubby"

+++

I think it's a generally agreed-upon sentiment that papers are Too Damn Hard to read. Which is really very unfortunate, because I think that reading peer-reviewed papers is basically the only way to *really* learn about a lot of things.

I'm not really at a place to be able to change that. But I *can* try and do the second-best thing: make it easy for others to learn from them. So that's what I'll do.

I'll start with [The Chubby Lock Service for Loosely-Coupled Distributed Systems](https://research.google.com/archive/chubby-osdi06.pdf) by Mike Burrows. It was presented at OSDI 2006. Chubby itself is not open source -- so all we have to go on for how it works is the paper itself -- but every tech company at some sort of nontrivial scale *probably* runs some sort of Chubby-equivalent.

More importantly, I think Chubby is interesting on a large part because the paper has one of the biggest `perceived complexity : actual complexity` ratios.

(You should know that I'm going to try to explain things in a way that I think makes this sound the easiest, instead of the most correct. Of course, the explanations should not be *incorrect*.)

## What Chubby Actually Is
Alright, are you ready for this?

Chubby is a *key-value store* that uses Paxos to keep copies.

If you only wanted to know was what Chubby is, you can stop reading now.

## Paxos
What are the implications of using Paxos?

[Paxos](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf) is basically a technique to make a bunch of separate nodes (i.e. processes, or threads, or actors, or pick your own word) agree on some *state*. In other words, pretend that every participating node has some copy of a state machine. Paxos ensures that *every node applies every transition to their copy of the state machine in the same order*.

It basically works like this:

1. You have a *cluster* of some number of nodes. One node is designated the *leader*. (Chubby uses the term *cell* instead of *cluster*, but I think the term *cluster* is more common, more intuitive and means the same thing.)
2. The leader proposes a transition (numbered `n`) to the state machine to every other node.
3. When enough of the other nodes acknowledges this to make up a majority (or *quorum*), the transition is committed. As a consequence, no transition with the number `n` or less can be committed, since there can exist no other majority to vote for a different transition with the number `n`.

I glossed over how *leader election* works in the steps above, but it uses essentially the same logic. The main difference is that, basically, any node can propose themselves for as a leader, so the first node that gets a quorum vote becomes the leader. Additionally, in Chubby, the master has a *lease* -- basically a leadership term of a few seconds, after which it needs to renew its lease by getting other nodes to vote for it again. Most importantly, this means that if the current leader goes down, the lease runs out and another leader is elected, with an up-to-date copy of the data.

(This actually describes a variant of Paxos called *multi-Paxos*. The *basic* variant of Paxos essentially has a leader election after a every state transition.)

A consequence of how quorums work is that in a Paxos cluster, a quorum needs to be alive for any progress to be made. A Chubby cluster normally consists of five nodes, so any two can be down and the state can still be updated.

In most of the Paxos implementations I've seen, the cluster leader proposes all write requests and serves all read requests -- Chubby is no different. The natural conclusion of this is that a using Paxos-distributed state machine *feels* like you're talking to a single process, because for the most part, it is.

Where in Chubby do you get *state*? You guessed it -- the key-value pairs.

(As a side node, why doesn't everyone use Paxos to replicate state? The problem is *latency* -- Paxos needs a lot of network round trips to even commit a transaction. In contrast, a simpler scheme like *asynchronous streaming*, where replicas just try to catch up to the leader at their own pace, only needs a single round trip to commit a transaction because replication is done separately from writes, sacrificing the strong consistency of Paxos.)

(It should be noted that Chubby's implementation of multi-Paxos eventually ended up being [a bit more complicated](https://static.googleusercontent.com/media/research.google.com/en//archive/paxos_made_live.pdf) than the one described above. Specifically, there are a few modifications to deal with things like changing the cluster size and garbage collecting the transaction log.)

## Chubby's state
So... I kind of lied when I said that Chubby is "just" a key-value store.

Chubby's API makes it look more like a Unix filesystem. The keys look like `/ls/foo/wombat/pouch` (`/ls` is the root, and it stands for *lock service*), and the API has methods like `Open()` and `Close()`.

The differences are that Chubby only has *nodes* that are equivalent to Unix's *files* and *directories* (basically lists containing their children's names), so nothing like a symlink or hard link. There's also no concept of operations like append and seek, so files can only be completely read or completely overwritten -- a design that makes it practical to store only very small files.

An interesting optimization is that nodes can't be moved, only created or deleted. The paper mentions that although it hasn't been needed yet, it also opens them to the possibility of "sharding" data between different Chubby instances, so that, for example, `/ls/foo` and everything in it is in its own Chubby cluster, but `/ls/bar` and everything in it is put into a separate Chubby cluster. (Interestingly, Google's [Megastore](http://cidrdb.org/cidr2011/Papers/CIDR11_Paper32.pdf), published 5 years later, does something like this.) Not allowing moves makes it easier to hide the distributed nature of Chubby, since moving lots of data between machines is hard and requires lots of synchronization.

Things that I've kind of skipped over:

* Ephemeral nodes: It's basically a node that only exists as long as there's at least one session that has it open. The implications of an ephemeral node is that it can be used to indicate that a client is alive.
* Access control: it's done by basically a slightly simpler version of Unix filesystem permissions.
* The Chubby client apparently does even more to make it seem like a file system. When clients open a file or directory, they get an object called a *handle*, which is similar to a Unix file descriptor.
* Events: push notify clients on things that have happened in Chubby, such as a lock being acquired or a file being edited.
* Caching: The Chubby client library apparently aggressively caches data from the Chubby cluster. The cool thing is that data is only ever evicted (presumably upon an event), never updated -- basically, keys are lazily cached. The client's view of the world is either up-to-date or unavailable (i.e if it loses connection), keeping with the theme of consistency over availability.
* Sessions: Clients maintain sessions by sending `KeepAlive` RPCs to Chubby. This constitutes about 93% of the example Chubby cluster's requests.

## Locks
Little does anyone know, the paper has the words "lock service" in the title. Let's look at how that works!

The Chubby handle API has the methods `Acquire()`, `TryAcquire()`, and `Release()` - these methods implement a [reader-writer lock](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock) on each node. It seems like the hierarchical nature of the Chubby data model means that acquiring a lock on a directory named `/ls/foo` means acquiring a lock on things inside it, which includes `/ls/foo/wombat/pouch`.

Chubby locks are *advisory* instead of *mandatory*. It's mentioned that Chubby locks are generally used to protect non-Chubby data or resources, so mandatory locks within Chubby aren't that useful, and mandatory locks that integrate with clients are too expensive.

A client holding a Chubby lock can request a *sequencer*, which is essentially a serialized "snapshot" of the lock. It can then pass the sequencer to operations that use that lock, and those operations will only succeed if the lock still has that state.

What if the client holding a lock loses its connection to Chubby? Instead of automatically releasing the lock, Chubby will hold the lock for some delay (preconfigured by the client). This means that if the client reconnects, it can reacquire the lock.

## Usage
To be sure, a highly consistent key-value store is a very generalized thing. So it's not really surprising that Chubby is widely used within Google for lots of different use cases.

* Leader election: The paper mentions that it became a common pattern to use Chubby as a way for clusters of other services to elect leaders. It's done like this: all the nodes try to grab a Chubby lock. Whoever acquires a lock first becomes the leader.
* As a name service: There's an entire section that talks about how Chubby came to replace DNS as the main way to *discover* servers within Google. Basically, since DNS is a time-based cache, there's no nice way of doing fast updates. In Chubby, you can accurately map keys to servers (i.e. host and port information) using ephemeral nodes.
  * A corollary of this is that in systems that need to shard data, the shard metadata is put onto Chubby. I have heard that this is true with [Vitess](http://vitess.io/).
  * A lot of other Google papers mention doing this. GFS and Bigtable are the main ones that come to mind. The Borg paper mentions a "Borg name service" that's built on Chubby.

The most interesting thing is that Chubby is basically used as a "distributed global variables" system.

The paper mentions that most teams at Google share the same Chubby cluster (and therefore namespace), but initially, developers didn't fully understand their usage, and caused many outages (and presumably tears). It's a very entertaining section on how important it is to either educate or weed out bad users, even internally.

A lot of the misconception around Chubby internally at Google seems to be from not being aware of the [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem). Basically, people assume that Chubby is both strongly consistent and (almost) always available, which is [impossible](https://codahale.com/you-cant-sacrifice-partition-tolerance/).

## Conclusion
I've glossed over a lot of the details of the paper, but hopefully I've given you a decent idea of how Chubby works, and what it's used for within Google!

Chubby has inspired a whole bunch of open-source projects, many of which are being used for the same things as mentioned by the paper. The most well-known one is [Zookeeper](https://zookeeper.apache.org/), which [seems](http://nerds.airbnb.com/smartstack-service-discovery-cloud/) [to](https://engineering.pinterest.com/blog/zookeeper-resilience-pinterest) be [popular](https://groups.google.com/forum/#!topic/mechanical-sympathy/GmyKrZn2Zus), but there's also [etcd](https://github.com/coreos/etcd), [Consul](https://www.consul.io/), and [Doozer](https://github.com/ha/doozerd), among many others.
