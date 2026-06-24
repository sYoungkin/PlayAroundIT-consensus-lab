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
        labels:
          node: "etcd1"

      - targets:
          - "192.168.135.136:2379"
        labels:
          node: "etcd2"

      - targets:
          - "192.168.135.137:2379"
        labels:
          node: "etcd3"
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

---

# Troubleshooting

This section documents issues encountered during the implementation of the Prometheus observability stack and the corresponding resolutions.

---

## Issue: Stale Scrape Times

### Symptoms

The Prometheus **Targets** page reported:

- All targets **UP**
- Last scrape approximately **45 minutes ago**
- Imported Grafana dashboards showed no recent data.

Example:

```text
Target: 192.168.135.135:2379

State:
UP

Last Scrape:
43 minutes ago
```

Although the configured scrape interval was:

```yaml
scrape_interval: 5s
```

Prometheus was clearly no longer ingesting new samples.

---

## Issue: Sample Ingestion Errors

The Prometheus journal reported warnings similar to:

```text
Error on ingesting samples that are too old or are too far into the future
```

and later

```text
Error on ingesting out-of-order samples
```

for every etcd scrape target.

Example:

```text
Error on ingesting out-of-order samples

component="scrape manager"

scrape_pool=etcd

target=http://192.168.135.135:2379/metrics
```

---

## Initial Investigation

The following checks were performed.

### Verify Prometheus Service

```bash
sudo systemctl status prometheus
```

Result:

- Service running normally.

---

### Verify Prometheus Logs

```bash
sudo journalctl -u prometheus -f
```

Result:

- Continuous warnings regarding out-of-order samples.

---

### Verify NTP Synchronization

Each virtual machine was checked.

```bash
timedatectl
```

Result:

- System clock synchronized: **yes**
- NTP active on all nodes.

---

### Verify Target Health

Prometheus successfully reached every etcd node.

All scrape targets remained:

```text
UP
```

Therefore the problem was **not** network connectivity.

---

## Root Cause

The issue was determined to be related to the local Prometheus time-series database (TSDB).

Although the scrape targets were healthy, Prometheus rejected incoming samples because their timestamps conflicted with previously stored samples.

Possible contributing factors include:

- virtual machine clock adjustments
- host sleep/resume
- NTP clock corrections
- existing TSDB data containing future or out-of-order timestamps

As a result, Prometheus refused to ingest new samples.

---

## Resolution

Stop Prometheus.

```bash
sudo systemctl stop prometheus
```

Remove the existing TSDB.

```bash
sudo rm -rf /var/lib/prometheus/*
```

Restart Prometheus.

```bash
sudo systemctl start prometheus
```

---

## Verification

Confirm that targets are scraped every few seconds.

Expected:

```text
Last Scrape

2s ago
5s ago
```

rather than

```text
43m ago
```

Verify that:

- no new ingestion warnings appear in the journal
- Grafana dashboards receive current data
- Prometheus queries return recent timestamps

---

## Time Zone Configuration

Although Linux internally keeps time in UTC, configuring every virtual machine to use the local time zone simplifies troubleshooting.

Recommended configuration:

```bash
sudo timedatectl set-timezone Europe/Berlin
```

Verify:

```bash
timedatectl
```

Expected:

```text
Local time:
Europe/Berlin

Universal time:
UTC

System clock synchronized:
yes
```

Changing the time zone affects only the displayed local time and does **not** alter the underlying system clock.

---

## Lessons Learned

- Healthy scrape targets do not necessarily imply successful metric ingestion.
- Prometheus may reject samples that are older than previously ingested samples or appear to be from the future.
- The Prometheus journal is the primary source for diagnosing ingestion problems.
- In a virtualized lab environment, host suspend/resume or clock corrections can result in timestamp inconsistencies.
- During development, clearing the TSDB is an acceptable recovery procedure when timestamp ordering becomes invalid.
- Maintaining synchronized clocks across all systems is essential for reliable observability.
- Using a consistent local time zone across the host and all virtual machines simplifies log correlation and troubleshooting.

---

## Persistent Startup Time Synchronization

### Problem

After shutting down and restarting the virtual machines, Prometheus repeatedly stopped ingesting new metrics.

Typical symptoms included:

- Prometheus targets remained **UP**
- Last scrape remained approximately **40–45 minutes** in the past
- Grafana dashboards stopped updating
- Prometheus reported repeated ingestion warnings such as:

```text
Error on ingesting samples that are too old or are too far into the future

Error on ingesting out-of-order samples
```

Clearing the TSDB restored normal operation, but the problem reappeared after the next reboot.

---

### Working Hypothesis

The current hypothesis is that Prometheus starts before the virtual machine clock has fully synchronized via NTP/Chrony.

The suspected sequence is:

```text
VM boots
      │
      ▼
Prometheus starts
      │
      ▼
Initial samples written using an incorrect system clock
      │
      ▼
Clock synchronizes via NTP/Chrony
      │
      ▼
New samples appear older or out-of-order
      │
      ▼
Prometheus rejects incoming samples
```

Although the virtual machines eventually report synchronized clocks, the initial startup sequence may already have introduced invalid timestamps into the TSDB.

This hypothesis has not yet been fully verified but is consistent with the observed behavior.

---

### Proposed Solution

Configure Prometheus to wait until the system clock has synchronized before starting.

---

### Install Chrony

```bash
sudo apt-get update
sudo apt-get install -y chrony

sudo systemctl enable --now chrony
```

---

### Create a Startup Synchronization Script

Create:

```text
/usr/local/bin/wait-for-time-sync.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

for i in {1..60}; do
    if chronyc tracking >/dev/null 2>&1 && \
       chronyc sources | grep -q '^\^\*'; then
        exit 0
    fi

    sleep 2
done

echo "Time synchronization not confirmed."
exit 1
```

Make the script executable.

```bash
sudo chmod +x /usr/local/bin/wait-for-time-sync.sh
```

---

### Add a systemd Override

Create a systemd override for Prometheus.

```bash
sudo systemctl edit prometheus
```

Add:

```ini
[Unit]
After=network-online.target chrony.service time-sync.target
Wants=network-online.target chrony.service time-sync.target

[Service]
ExecStartPre=/usr/local/bin/wait-for-time-sync.sh
```

Reload systemd.

```bash
sudo systemctl daemon-reload
```

Restart Prometheus.

```bash
sudo systemctl restart prometheus
```

---

### One-Time Recovery

If timestamp inconsistencies already exist within the local TSDB, clear the database before restarting Prometheus.

```bash
sudo systemctl stop prometheus

sudo rm -rf /var/lib/prometheus/*

sudo systemctl start prometheus
```

Because this lab environment is used for experimentation rather than production monitoring, removing the local TSDB is acceptable.

---

### Verification

After restarting:

- Prometheus targets should be scraped every few seconds.
- No "out-of-order samples" warnings should appear.
- No "samples too old" warnings should appear.
- Grafana dashboards should update continuously.

Verify:

```bash
sudo journalctl -u prometheus -f
```

Expected:

No further timestamp-related ingestion warnings.

---

### Future Investigation

This solution represents the current best hypothesis based on observed behavior.

Future experiments should determine:

- Whether the issue is reproducible after every VM reboot.
- Whether waiting for time synchronization completely eliminates the problem.
- Whether VMware Fusion guest time synchronization contributes to the issue.
- Whether the problem is specific to Prometheus 3.x or also occurs with earlier releases.

Documenting both the hypothesis and the validation results will improve the operational understanding of the observability stack over time.
