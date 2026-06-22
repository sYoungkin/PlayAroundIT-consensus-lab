# Consensus Lab Goals

## Overview

The purpose of this project is to develop a practical understanding of distributed systems by building and experimenting with a small **etcd** cluster implementing the **Raft consensus algorithm**. Rather than treating etcd as a black box, the goal is to understand _why_ the cluster behaves the way it does during normal operation and under failure conditions.

This project emphasizes experimentation, observation, and documentation over automation.

---

# Objectives

## Primary Objectives

- Build a three-node etcd cluster from scratch.
- Understand the Raft consensus algorithm through real experiments.
- Observe leader election, quorum formation, and log replication.
- Visualize cluster health and consensus metrics using Prometheus and Grafana.
- Develop an intuition for how distributed systems maintain consistency despite failures.

---

## Technical Learning Objectives

By the end of this project I should understand:

- Why distributed consensus is necessary.
- The problem consensus algorithms solve.
- The responsibilities of leaders and followers.
- Election terms and leader elections.
- Quorum and majority voting.
- Log replication.
- Commit vs. apply semantics.
- Cluster membership.
- Failure recovery.
- Split-brain prevention.
- Linearizable reads and writes.
- Why writes require a majority of nodes.
- Why reads may or may not require contacting the leader.
- Network partitions and their effects.
- Recovery after partitions heal.

---

## Observability Objectives

The cluster should be observable rather than opaque.

The lab should make it possible to observe:

- Current cluster leader
- Leader changes over time
- Raft terms
- Proposal rates
- Commit index
- Applied index
- Cluster health
- Member availability
- Request latency
- System resource utilization

using Prometheus metrics, Grafana dashboards, and etcd logs.

---

# Experiments

The lab will include controlled experiments that demonstrate the behavior of the Raft algorithm.

Examples include:

- Cluster startup
- Normal write operations
- Stopping a follower
- Restarting a follower
- Stopping the leader
- Automatic leader election
- Quorum loss
- Node recovery
- Network latency experiments
- Network partition experiments
- Simultaneous node failures
- Performance under load

Each experiment should answer:

1. What happened?
2. Why did it happen?
3. What changed inside the cluster?
4. Which metrics reflected the change?
5. Which log messages explain the behavior?

---

# Project Philosophy

This lab intentionally avoids infrastructure automation.

Configuration will be performed manually to better understand:

- etcd configuration
- cluster formation
- service management
- troubleshooting
- operational procedures

Automation can always be added later; understanding comes first.

---

# Repository Organization

The repository contains four major components:

- **config/** – configuration files for etcd, Prometheus, and Grafana
- **docs/** – theory, architecture, and implementation notes
- **experiments/** – documented experiments and observations
- **scripts/** – small utility scripts used during the lab

---

# Success Criteria

The project is considered successful when I can confidently explain:

- how Raft elects a leader,
- why quorum is required,
- how log replication works,
- how the cluster recovers from failures,
- how consensus guarantees consistency,
- how etcd exposes its internal state through metrics and logs,
- and how to diagnose cluster behavior using observability tools.

Beyond operating an etcd cluster, the ultimate goal is to develop an intuition for distributed consensus that transfers to larger distributed systems such as Kubernetes, databases, service discovery systems, and distributed data platforms.
