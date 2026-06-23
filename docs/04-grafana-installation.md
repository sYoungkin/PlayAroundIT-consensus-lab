# Grafana Installation

## Overview

This document describes the installation and configuration of Grafana for the Consensus Lab.

Grafana provides the visualization layer of the observability stack.

Where Prometheus is responsible for collecting and storing time-series metrics, Grafana transforms those metrics into dashboards that allow us to observe the internal behavior of the distributed system.

The objective is **not** simply to create attractive graphs, but to build an observatory that makes the Raft consensus algorithm visible.

---

# Architecture

```text
                             Browser
                                 │
                                 ▼
                       Grafana (obs:3000)
                                 │
                            PromQL Queries
                                 │
                                 ▼
                     Prometheus (obs:9090)
                                 │
                        HTTP GET /metrics
                                 │
          ┌──────────────────────┼──────────────────────┐
          ▼                      ▼                      ▼
      etcd1                  etcd2                  etcd3
      :2379                  :2379                  :2379
```

Grafana never communicates directly with the etcd cluster.

Instead:

1. Prometheus periodically scrapes each etcd node.
2. Prometheus stores the collected metrics.
3. Grafana queries Prometheus.
4. The browser renders the dashboard.

This separation of responsibilities keeps metric collection and visualization independent.

---

# Why Grafana?

During this project we want to answer questions such as:

- Which node is currently the leader?
- How often do leader elections occur?
- Does every node currently know the leader?
- What happens immediately after the leader crashes?
- Which metrics change during quorum loss?
- Which subsystem is currently active?

Grafana allows us to answer these questions visually.

---

# Installation

## Install Prerequisites

```bash
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    software-properties-common \
    wget
```

---

## Add the Grafana GPG Key

```bash
wget -q -O - https://apt.grafana.com/gpg.key | \
sudo gpg --dearmor \
-o /etc/apt/keyrings/grafana.gpg
```

---

## Add the Repository

```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
sudo tee /etc/apt/sources.list.d/grafana.list
```

---

## Install Grafana

```bash
sudo apt-get update
sudo apt-get install -y grafana
```

---

# systemd

Grafana is installed as a native Linux service.

Enable startup.

```bash
sudo systemctl enable grafana-server
```

Start Grafana.

```bash
sudo systemctl start grafana-server
```

Verify status.

```bash
sudo systemctl status grafana-server
```

---

# Verification

Open Grafana.

```text
http://localhost:3000
```

Default credentials.

```text
Username: admin
Password: admin
```

Grafana requests a password change during the first login.

---

# Configure the Prometheus Data Source

Navigate to

```text
Connections
    ↓
Data Sources
```

Choose

```text
Prometheus
```

Configure the URL.

```text
http://localhost:9090
```

Because Grafana and Prometheus run on the same virtual machine, `localhost` refers to the Prometheus instance running on the **obs** node.

Click

```text
Save & Test
```

Expected result:

```text
Data source is working
```

---

# Initial Verification

Before creating dashboards, verify that Grafana can successfully execute PromQL queries.

Example:

```promql
etcd_server_is_leader
```

Expected result:

- three time series
- one member reports a value of **1**
- two members report a value of **0**

This confirms that:

- Grafana can communicate with Prometheus
- Prometheus is successfully scraping every etcd member
- Raft metrics are available for visualization

---

# Lessons Learned

- Grafana is responsible only for visualization.
- Prometheus remains the system of record for all collected metrics.
- Dashboards execute PromQL queries against Prometheus.
- Grafana never communicates directly with etcd.
- The observability stack is now complete:

```text
Browser
    │
Grafana
    │
Prometheus
    │
etcd Cluster
```

The next stage of the project is to design a dashboard that exposes the internal behavior of the Raft consensus algorithm and the major etcd subsystems.
