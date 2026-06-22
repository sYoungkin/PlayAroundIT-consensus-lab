# Consensus Lab

Local distributed-systems lab for studying Raft consensus using etcd.

## Goals

- Build a 3-node etcd cluster.
- Observe leader election, quorum, log replication, and failure recovery.
- Collect etcd metrics with Prometheus.
- Visualize cluster behavior in Grafana.
- Document controlled failure experiments.

## Nodes

| Host  |            IP | Role                 |
| ----- | ------------: | -------------------- |
| etcd1 | 192.168.56.11 | etcd member          |
| etcd2 | 192.168.56.12 | etcd member          |
| etcd3 | 192.168.56.13 | etcd member          |
| obs1  | 192.168.56.20 | Prometheus + Grafana |

consensus-lab/
├── README.md
├── Vagrantfile
├── .gitignore
│
├── config/
│ ├── etcd/
│ ├── prometheus/
│ └── grafana/
│
├── docs/
│ ├── 00-lab-goals.md
│ ├── 01-raft-concepts.md
│ ├── 02-etcd-cluster.md
│ └── 03-observability.md
│
├── experiments/
│ ├── 01-baseline-health.md
│ ├── 02-stop-follower.md
│ ├── 03-stop-leader.md
│ ├── 04-quorum-loss.md
│ └── 05-rejoin-node.md
│
└── scripts/
└── lab_status.sh
