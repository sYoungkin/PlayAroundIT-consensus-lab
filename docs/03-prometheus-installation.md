# Prometheus Installation

## Overview

This document describes the installation and configuration of Prometheus for the Consensus Lab.

Prometheus provides the observability layer for the etcd cluster. Rather than simply confirming that the cluster is operational, Prometheus allows us to observe the internal state of the Raft consensus algorithm over time.

Unlike log files, which record discrete events, Prometheus continuously collects time-series metrics that describe the behavior of the distributed system.

This observability layer will later be visualized using Grafana.

---

# Why Prometheus?

The primary goal of this project is not simply to deploy etcd, but to understand **how distributed consensus behaves**.

Prometheus allows us to answer questions such as:

- Which node is currently the leader?
- Has a leader been elected?
- How many elections have occurred?
- How frequently are proposals committed?
- How does the cluster behave after failures?
- Which metrics change during leader elections?
- Which metrics indicate quorum loss?

Prometheus therefore serves as the "microscope" for the distributed system.

---

# Pull vs Push

One interesting aspect of Prometheus is that it uses a **pull model**.

Rather than applications pushing metrics to a central server, Prometheus periodically contacts each target and retrieves its current metrics over HTTP.

```
Prometheus
     │
HTTP GET /metrics
     │
     ▼
Application
```

This differs from many log aggregation systems (such as Splunk or the Elastic Stack), where data is typically forwarded from the application to the collector.

The pull model makes service discovery, health checking, and metric collection simple and scalable for long-running infrastructure components.

---

# Architecture

```
                              Browser
                                  │
                                  ▼
                      Prometheus (obs:9090)
                                  │
                        HTTP GET /metrics
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
          ▼                       ▼                       ▼
   etcd1 (2379)            etcd2 (2379)            etcd3 (2379)
          │                       │                       │
          └────────────── Raft Consensus Cluster ──────────────┘
```

Prometheus scrapes each etcd member independently.

Each node exposes its own view of the cluster, allowing us to observe leader elections and other distributed-system behavior from multiple perspectives.

---

# Installation

## Install Dependencies

```bash
sudo apt-get update
sudo apt-get install -y curl tar
```

---

## Download Prometheus

```bash
PROM_VERSION="3.6.0"
ARCH="linux-arm64"

cd /tmp

curl -LO https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.${ARCH}.tar.gz

tar xzf prometheus-${PROM_VERSION}.${ARCH}.tar.gz
cd prometheus-${PROM_VERSION}.${ARCH}
```

---

## Create Service Account

```bash
sudo useradd \
    --system \
    --home /var/lib/prometheus \
    --shell /usr/sbin/nologin \
    prometheus
```

The Prometheus service runs as a dedicated system account following the principle of least privilege.

---

## Directory Structure

Create the required directories.

```bash
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus
```

Install the binaries.

```bash
sudo install -m 755 prometheus /usr/local/bin/
sudo install -m 755 promtool /usr/local/bin/
```

Verify the installation.

```bash
prometheus --version
promtool --version
```

Directory layout:

```
/usr/local/bin/
    prometheus
    promtool

/etc/prometheus/
    prometheus.yml

/var/lib/prometheus/
    (TSDB)
```

---

# Prometheus Configuration

Configuration file:

```
/etc/prometheus/prometheus.yml
```

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: "etcd"

    static_configs:
      - targets:
          - "192.168.135.135:2379"
          - "192.168.135.136:2379"
          - "192.168.135.137:2379"
```

## Configuration Explanation

### scrape_interval

Defines how often Prometheus collects metrics from every configured target.

A five-second interval provides sufficient resolution to observe rapid events such as leader elections while keeping storage requirements low.

---

### job_name

Groups related targets together.

All etcd members belong to a single job named `etcd`.

---

### static_configs

Defines a static list of scrape targets.

In this lab, every etcd member is configured explicitly.

---

### targets

Each target is an HTTP endpoint exposing Prometheus metrics.

For etcd, metrics are available on the client endpoint (port **2379**).

---

# systemd Service

Create:

```
/etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple

ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=:9090

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Reload systemd.

```bash
sudo systemctl daemon-reload
```

Enable startup.

```bash
sudo systemctl enable prometheus
```

Start Prometheus.

```bash
sudo systemctl start prometheus
```

Verify status.

```bash
sudo systemctl status prometheus
```

---

# Verification

Open the Prometheus web interface.

```
http://localhost:9090
```

Verify that:

- all three etcd members appear as **UP**
- metrics are successfully scraped
- no scrape errors are reported

---

# Initial Observations

Immediately after installation, Prometheus exposed numerous etcd metrics related to the Raft consensus algorithm.

Examples include:

- `etcd_server_is_leader`
- `etcd_server_has_leader`
- `etcd_server_leader_changes_seen_total`

These metrics directly correspond to concepts introduced in the Raft documentation, providing the first opportunity to observe the consensus algorithm through time-series data.

---

# Lessons Learned

- Prometheus uses a pull-based collection model.
- etcd exposes native Prometheus metrics without requiring an exporter.
- Every etcd member is scraped independently.
- Time-series metrics complement the journal logs by showing how cluster state evolves over time.
- Prometheus forms the observability foundation for the remainder of the Consensus Lab.
- The next phase of the project is to explore, classify, and understand the available etcd metrics before building Grafana dashboards.
