# Raft Concepts

## Purpose

Raft is a distributed consensus algorithm designed to allow a collection of machines to behave as a single, reliable system.

Its purpose is simple:

> Ensure that every node agrees on the same ordered sequence of state changes, even when machines fail.

Raft was designed to be easier to understand than previous consensus algorithms such as Paxos while providing the same fundamental guarantees.

---

# The Consensus Problem

Imagine three servers storing the same data.

```
Node A
Node B
Node C
```

A client wants to write:

```
x = 42
```

Questions immediately arise:

- Which machine accepts the write?
- What happens if one machine is offline?
- What happens if two machines disagree?
- What if the leader crashes halfway through the write?
- How do all machines eventually end up with identical data?

Consensus algorithms answer these questions.

---

# The Goal of Consensus

Consensus is **not** about making every machine identical all the time.

Consensus is about ensuring that every machine agrees on the same **history** of operations.

Instead of agreeing on the current state,

```
x = 42
```

Raft agrees on the ordered log:

```
1  create x
2  set x = 5
3  set x = 17
4  set x = 42
```

If every machine has the same ordered log, they will eventually compute the same state.

The log—not the data itself—is what consensus protects.

---

# The Three Node Roles

At any moment, every node is exactly one of three roles.

## Leader

The leader coordinates the cluster.

Responsibilities:

- accepts client writes
- replicates log entries
- commits entries
- sends heartbeat messages
- maintains cluster leadership

Only one leader may exist within a term.

---

## Follower

Followers are passive.

They:

- receive replicated log entries
- acknowledge writes
- respond to vote requests
- expect periodic heartbeats from the leader

Followers never initiate writes.

---

## Candidate

A follower becomes a candidate when it believes the leader has failed.

It then:

- increments the election term
- votes for itself
- requests votes from the other members

If it receives a majority of votes:

```
Candidate
        ↓
Leader
```

Otherwise it returns to follower.

---

# Terms

Time is divided into **terms**.

Each election starts a new term.

```
Term 1
Leader: Node A

↓

Leader crashes

↓

Term 2
Leader: Node C

↓

Leader crashes

↓

Term 3
Leader: Node B
```

Terms provide a global ordering of leadership.

Every node always knows the highest term it has observed.

---

# Heartbeats

Leaders periodically send heartbeat messages.

```
Leader

↓

Follower
Follower
```

The heartbeat tells followers:

> "I am still alive."

If followers stop receiving heartbeats, they assume the leader has failed and begin a new election.

---

# Leader Election

Suppose we begin with

```
Leader
Follower
Follower
```

The leader crashes.

Followers stop receiving heartbeats.

After an election timeout:

```
Follower

↓

Candidate
```

The candidate asks:

```
May I become leader?
```

Each node votes once per term.

If the candidate receives a majority:

```
Leader
Follower
Follower
```

The cluster continues operating.

---

# Majority (Quorum)

Raft never requires every node.

It requires a majority.

For:

```
3 nodes
```

Majority =

```
2
```

For:

```
5 nodes
```

Majority =

```
3
```

This majority is called the **quorum**.

Without a quorum:

- no leader can be elected
- no writes may be committed

This is one of the central ideas of distributed systems.

---

# The Log

Every write becomes a log entry.

Example:

```
1  Create user

2  Update email

3  Delete session

4  Grant permission
```

Every node stores the same ordered log.

The leader's primary responsibility is ensuring that every follower eventually has this same log.

---

# Log Replication

Suppose a client writes:

```
SET balance = 100
```

The leader:

1. appends the entry to its own log
2. sends it to followers
3. waits for acknowledgements from a majority
4. marks the entry committed
5. informs followers that the entry is committed

Only then is the write considered successful.

---

# Commit vs Apply

These are different concepts.

## Committed

The cluster has agreed the entry is permanent.

It will never disappear.

---

## Applied

The application has actually executed the command.

Example:

```
Committed

↓

Applied

↓

Database updated
```

This distinction is surprisingly important when studying Raft.

---

# Safety

Raft guarantees:

- at most one leader per term
- committed entries are never lost
- all nodes agree on log ordering
- stale leaders cannot overwrite newer data

These guarantees are maintained even when machines fail.

---

# Availability

Raft is highly available, but not infinitely available.

For a three-node cluster:

```
Node 1
Node 2
Node 3
```

The cluster survives:

- one node failure

The cluster cannot survive:

- two simultaneous failures

Availability depends on maintaining quorum.

---

# Why Odd Numbers?

Odd numbers maximize fault tolerance for the fewest machines.

| Cluster Size | Quorum | Failures Tolerated |
| ------------ | ------ | ------------------ |
| 1            | 1      | 0                  |
| 2            | 2      | 0                  |
| 3            | 2      | 1                  |
| 4            | 3      | 1                  |
| 5            | 3      | 2                  |
| 6            | 4      | 2                  |
| 7            | 4      | 3                  |

Adding an even-numbered node increases cost without increasing fault tolerance.

---

# Raft in etcd

etcd is an implementation of the Raft consensus algorithm.

In this lab we will directly observe:

- leader elections
- heartbeat traffic
- leader failures
- follower recovery
- quorum loss
- log replication
- committed and applied indexes
- cluster membership
- Raft metrics
- Raft log messages

Rather than treating these as abstract concepts, we will watch them occur in real time.

---

# Mental Model

A useful way to think about Raft is:

> A group of computers continuously negotiating a shared history.

The leader proposes new history.

Followers verify and store that history.

A majority agrees before history becomes permanent.

As long as a majority of machines remain alive and connected, the cluster continues operating as a single logical system.

Everything else in Raft—terms, elections, heartbeats, logs, commits, and quorum—exists to protect that shared history.
