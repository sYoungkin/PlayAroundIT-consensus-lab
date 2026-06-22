# etcd Cluster

## Overview

This document describes the installation, configuration, and verification of the three-node etcd cluster used throughout the Consensus Lab.

The cluster is intentionally built manually in order to understand every component involved in bootstrapping a distributed system based on the Raft consensus algorithm.

---

# Cluster Topology

| Node | Hostname | IP Address      | Role                 |
| ---- | -------- | --------------- | -------------------- |
| 1    | etcd1    | 192.168.135.135 | etcd Member          |
| 2    | etcd2    | 192.168.135.136 | etcd Member          |
| 3    | etcd3    | 192.168.135.137 | etcd Member          |
| 4    | obs      | 192.168.135.138 | Prometheus / Grafana |

---

# Virtual Machine Specifications

| VM    | CPU |  Memory |
| ----- | --: | ------: |
| etcd1 |   1 | 1024 MB |
| etcd2 |   1 | 1024 MB |
| etcd3 |   1 | 1024 MB |
| obs   |   2 | 2048 MB |

---

# Installation

## Install Required Packages

Run on every etcd node.

```bash
sudo apt-get update
sudo apt-get install -y curl tar
```

---

## Download etcd

```bash
ETCD_VER="v3.6.12"
ARCH="arm64"
DOWNLOAD_URL="https://github.com/etcd-io/etcd/releases/download"

cd /tmp

curl -L "${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-${ARCH}.tar.gz" \
  -o "etcd-${ETCD_VER}-linux-${ARCH}.tar.gz"

tar xzf "etcd-${ETCD_VER}-linux-${ARCH}.tar.gz"

sudo install -m 0755 "etcd-${ETCD_VER}-linux-${ARCH}/etcd" /usr/local/bin/etcd
sudo install -m 0755 "etcd-${ETCD_VER}-linux-${ARCH}/etcdctl" /usr/local/bin/etcdctl
sudo install -m 0755 "etcd-${ETCD_VER}-linux-${ARCH}/etcdutl" /usr/local/bin/etcdutl
```

Verify the installation.

```bash
etcd --version
etcdctl version
etcdutl version
```

---

# Linux Service Account

Create a dedicated system user.

```bash
sudo useradd \
  --system \
  --home /var/lib/etcd \
  --shell /usr/sbin/nologin \
  etcd
```

---

# Directory Layout

Create the required directories.

```bash
sudo mkdir -p /etc/etcd
sudo mkdir -p /var/lib/etcd

sudo chown -R etcd:etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd
```

Directory structure

```text
/usr/local/bin/
    etcd
    etcdctl
    etcdutl

/etc/etcd/
    etcd.env

/var/lib/etcd/
    (persistent database)

/etc/systemd/system/
    etcd.service
```

---

# Environment Configuration

Each node has an environment file located at

```text
/etc/etcd/etcd.env
```

Example (`etcd1`)

```bash
ETCD_NAME=etcd1
ETCD_DATA_DIR=/var/lib/etcd

ETCD_LISTEN_PEER_URLS=http://192.168.135.135:2380
ETCD_LISTEN_CLIENT_URLS=http://192.168.135.135:2379,http://127.0.0.1:2379

ETCD_INITIAL_ADVERTISE_PEER_URLS=http://192.168.135.135:2380
ETCD_ADVERTISE_CLIENT_URLS=http://192.168.135.135:2379

ETCD_INITIAL_CLUSTER=etcd1=http://192.168.135.135:2380,etcd2=http://192.168.135.136:2380,etcd3=http://192.168.135.137:2380
ETCD_INITIAL_CLUSTER_TOKEN=consensus-lab-etcd
ETCD_INITIAL_CLUSTER_STATE=new
```

The only values that differ between nodes are:

- `ETCD_NAME`
- `ETCD_LISTEN_PEER_URLS`
- `ETCD_LISTEN_CLIENT_URLS`
- `ETCD_INITIAL_ADVERTISE_PEER_URLS`
- `ETCD_ADVERTISE_CLIENT_URLS`

All other configuration is identical.

---

## ETCD_NAME

Identifies the member within the cluster.

Example

```text
ETCD_NAME=etcd1
```

Every member name must be unique.

---

## ETCD_DATA_DIR

Specifies where the Raft log and database are stored.

```text
/var/lib/etcd
```

This directory represents the persistent state of the cluster.

Deleting this directory effectively removes the node's local history.

---

## ETCD_LISTEN_CLIENT_URLS

Defines where clients may connect.

Port

```text
2379
```

This is used by:

- etcdctl
- Kubernetes
- Prometheus
- Applications

---

## ETCD_LISTEN_PEER_URLS

Defines where Raft peers communicate.

Port

```text
2380
```

This traffic never comes from clients.

It is exclusively used for cluster communication.

---

## ETCD_ADVERTISE_CLIENT_URLS

The address advertised to clients.

Other systems use this address to connect to the member.

---

## ETCD_INITIAL_ADVERTISE_PEER_URLS

The address advertised to the remaining cluster members.

This is the address used during Raft communication.

---

## ETCD_INITIAL_CLUSTER

Defines every member participating in the initial cluster.

Example

```text
etcd1=http://192.168.135.135:2380
etcd2=http://192.168.135.136:2380
etcd3=http://192.168.135.137:2380
```

Every node starts with the exact same cluster definition.

Only `ETCD_NAME` differs between members.

---

## ETCD_INITIAL_CLUSTER_TOKEN

Unique identifier for the cluster.

This prevents accidental interaction between unrelated clusters.

---

## ETCD_INITIAL_CLUSTER_STATE

During the initial bootstrap:

```text
new
```

After a cluster already exists, members recover from their persisted state rather than creating a new cluster.

---

# systemd Service

`/etc/systemd/system/etcd.service`

```ini
[Unit]
Description=etcd key-value store
Documentation=https://etcd.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
Group=etcd
EnvironmentFile=/etc/etcd/etcd.env
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=5s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```

Reload systemd.

```bash
sudo systemctl daemon-reload
```

Enable automatic startup.

```bash
sudo systemctl enable etcd
```

---

# Starting the Cluster

Start the service on all three nodes.

```bash
sudo systemctl start etcd
```

Check status.

```bash
sudo systemctl status etcd --no-pager
```

---

# Cluster Verification

Check endpoint status.

```bash
export ETCDCTL_API=3

etcdctl \
  --endpoints=http://192.168.135.135:2379,http://192.168.135.136:2379,http://192.168.135.137:2379 \
  endpoint status --write-out=table
```

Check endpoint health.

```bash
etcdctl \
  --endpoints=http://192.168.135.135:2379,http://192.168.135.136:2379,http://192.168.135.137:2379 \
  endpoint health --write-out=table
```

List cluster members.

```bash
etcdctl \
  --endpoints=http://192.168.135.135:2379,http://192.168.135.136:2379,http://192.168.135.137:2379 \
  member list --write-out=table
```

---

# Functional Verification

Write a key.

```bash
etcdctl \
  --endpoints=http://192.168.135.135:2379 \
  put lab/status "cluster-online"
```

Read the key from another member.

```bash
etcdctl \
  --endpoints=http://192.168.135.136:2379 \
  get lab/status
```

Successful output:

```text
lab/status
cluster-online
```

This confirms:

- client connectivity
- leader coordination
- log replication
- replicated state across the cluster

---

# Viewing Raft Activity

View recent log messages.

```bash
sudo journalctl -u etcd -n 100 --no-pager
```

Follow the logs in real time.

```bash
sudo journalctl -u etcd -f
```

Show election-related events.

```bash
sudo journalctl -u etcd --no-pager | grep -Ei "leader|candidate|follower|election|term"
```

---

# Initial Cluster State

Observed after the initial bootstrap:

- Three healthy members
- Automatic leader election
- Leader: **etcd3**
- Followers: **etcd1**, **etcd2**
- Successful write/read test
- Cluster healthy

Observed Raft state transitions:

Leader node:

```text
Follower
    ↓
Candidate
    ↓
Leader
```

Follower nodes:

```text
Follower
```

These transitions were visible directly in the systemd journal, providing the first real observation of the Raft election process.

---

# Lessons Learned

- Every node starts with the same cluster definition.
- Each member has a unique identity (`ETCD_NAME`).
- Raft communication uses port **2380**.
- Client communication uses port **2379**.
- A leader is elected automatically during cluster startup.
- `systemd` provides a production-style way to manage the service.
- The journal is an excellent source for observing Raft state transitions.
- `etcdctl` provides all the tools required to inspect cluster health and membership.
- The cluster successfully replicated data across all three members, confirming a healthy consensus group.
