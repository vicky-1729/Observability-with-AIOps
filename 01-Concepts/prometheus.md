# Prometheus — Concepts and Setup Notes

> Companion to the class running notes (`session-01.txt`, `session-02.txt`).
> Use this for revision.

**Covered so far**

1. [Why metrics?](#1-why-metrics)
2. [Prometheus as a Time Series Database (TSDB)](#2-prometheus-as-a-time-series-database-tsdb)
3. [Push vs Pull](#3-push-vs-pull)
4. [How Prometheus uses the pull model (scraping)](#4-how-prometheus-uses-the-pull-model-scraping)
5. [Exporters and their types](#5-exporters)
6. [Installing Prometheus and Node Exporter](#6-installing-prometheus-and-node-exporter)
7. [Static configs in `prometheus.yml`](#7-static-configs-in-prometheusyml)
8. [Service Discovery — EC2 SD](#8-service-discovery--ec2-sd)
9. [Relabeling — `relabel_configs` and `metric_relabel_configs`](#9-relabeling--relabel_configs-and-metric_relabel_configs)

---

## 1. Why metrics?

### The doctor analogy

When we visit a doctor, they don't guess. They measure **BP, sugar, temperature, height, weight** and
run tests. From these numbers they find **what is wrong**, and further tests tell them **why it is wrong**.

Our applications and servers are no different. To run them reliably we need to answer:

- **Where** is the application running? (servers, containers, cloud)
- **What** is the application doing? (requests, errors, resource usage)
- **What is wrong?**
- **Why is it wrong?**

### The three pillars of observability

| Pillar      | What it is                                         | Answers                  | Example tool        |
|-------------|----------------------------------------------------|--------------------------|---------------------|
| **Metrics** | Numbers measured over time (CPU %, requests/sec)   | **What** is wrong?       | Prometheus          |
| **Logs**    | Timestamped text records of events                 | **Why** is it wrong?     | Loki, ELK           |
| **Traces**  | The path of one request across many services       | **Where / why** it is slow | Tempo, Jaeger     |

> **Metrics tell us *what* is wrong. Logs and traces tell us *why* it is wrong.**

### Why a single value is useless — we need the trend

A single reading ("CPU is 70% right now") tells us very little. Is it rising? Is it normal at this hour?
Metrics are valuable because we collect them **continuously** and store them, so we can:

- see **trends** (disk filling up 2% per day → full in 10 days),
- compare **now vs. earlier** (latency doubled after the last deployment),
- **alert** when a value crosses a threshold for some time,
- do **capacity planning**.

So both the **infrastructure** (servers, disks, network) and the **application** (requests, errors)
must emit data continuously, and we need a system that collects and stores it against time.

### What should we measure? — The 4 Golden Signals (Google SRE) — "LETS"

| Signal         | Question it answers                                         |
|----------------|-------------------------------------------------------------|
| **L**atency    | How long does a request take?                               |
| **E**rrors     | Are requests failing? (HTTP 5xx, exceptions)                |
| **T**raffic    | How many requests are coming in? (requests/sec)             |
| **S**aturation | How full are our resources? (CPU, RAM, disk, bandwidth)     |

Related methods you may hear in interviews:
- **USE** (for resources/infra): Utilisation, Saturation, Errors.
- **RED** (for services): Rate, Errors, Duration.

---

## 2. Prometheus as a Time Series Database (TSDB)

### What is Prometheus?

Prometheus is an open-source **monitoring and alerting system** (a CNCF graduated project, originally built at
SoundCloud). At its core it is a **Time Series Database** — it stores every metric value **against a timestamp**.

> "There is only one real thing in the universe — time." Every metric value is bound to a timestamp.

### What is a time series?

A **time series** is a sequence of values for the same thing, recorded over time:

```
bp{person="ramesh", city="hyderabad"}

value      timestamp
140/100    09:19:00 PM   (30 sec back)
130/90     09:19:15 PM   (15 sec back)
120/80     09:19:30 PM   (now)
```

Just like tracking daily expenses: *date → food / travel / movie → how much*. The date is the timestamp,
the category is the label, the amount is the value.

### The Prometheus data model

Every time series is identified by a **metric name** plus a set of **labels** (key-value pairs):

```
<metric_name>{<label_1>="<value_1>", <label_2>="<value_2>"}  <sample_value>  [timestamp]
```

Example:

```
node_cpu_seconds_total{cpu="0", mode="idle", instance="10.0.1.187:9100", job="node"}  18342.53
```

| Part              | Meaning                                                              |
|-------------------|----------------------------------------------------------------------|
| Metric name       | *What* is measured — `node_cpu_seconds_total`                        |
| Labels            | *Which one* — dimensions to filter/group by (cpu, mode, instance)    |
| Sample value      | A float64 number                                                     |
| Timestamp         | Millisecond precision, added by Prometheus at scrape time            |

**Key point:** a different combination of labels = a **different time series**.
`{cpu="0"}` and `{cpu="1"}` are two separate series of the same metric.

> ⚠️ **High cardinality:** never put unbounded values (user IDs, request IDs, emails) in labels — every unique
> value creates a new series and can blow up Prometheus memory.

### The `up` metric

For every target it scrapes, Prometheus automatically records:

```
up{instance="localhost:9090", job="prometheus"}  1
```

- `1` → the scrape succeeded (target is reachable)
- `0` → the scrape failed (target is down / unreachable)

`up == 0` is the most basic "server is down" alert.

### Metric types (common interview question)

| Type          | Behaviour                                   | Example                                 |
|---------------|---------------------------------------------|-----------------------------------------|
| **Counter**   | Only goes up (resets to 0 on restart)       | `http_requests_total`, `node_cpu_seconds_total` |
| **Gauge**     | Goes up and down                            | `node_memory_MemAvailable_bytes`, temperature |
| **Histogram** | Counts observations into buckets            | request latency buckets (`_bucket`, `_sum`, `_count`) |
| **Summary**   | Like histogram, but quantiles computed on the client | `go_gc_duration_seconds{quantile="0.5"}` |

Counters are almost always used with `rate()` in queries, e.g. `rate(http_requests_total[5m])`.

### PromQL

Data in Prometheus is queried with **PromQL (Prometheus Query Language)**. Examples:

```promql
up                                              # which targets are up/down
node_memory_MemAvailable_bytes / 1024 / 1024    # available memory in MB
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)   # CPU usage %
```

### What Prometheus does — in four steps

1. **Identify** the targets (static config or service discovery)
2. **Scrape** metrics from them
3. **Store** them in the TSDB
4. **Evaluate rules and alert** (via Alertmanager)

---

## 3. Push vs Pull

| | **Push model** | **Pull model** |
|---|---|---|
| How it works | The target **sends** data to the monitoring server when something happens | The monitoring server **fetches** data from targets periodically (e.g. every 15s) |
| Direction | target → server | server → target |
| Examples | StatsD, Graphite, CloudWatch agent, Pushgateway | **Prometheus** |
| Health detection | Silence is ambiguous — is the target down or just idle? | Failed scrape = target down (`up == 0`) |
| Control | Targets decide how often to send; can flood the server | Server controls the rate and load |
| Network | Target needs outbound access to server | Server needs to reach the target's port |

### Why observability prefers pull

- We need **complete, regular data** to analyse trends — pull guarantees a sample every interval.
- Prometheus knows **exactly which targets should exist**, so a missing target is detected immediately.
- The monitoring server controls the load; a misbehaving app cannot flood it.
- Easy to debug: open `http://<target>:9100/metrics` in a browser and see exactly what Prometheus sees.

### When is push still the right choice?

1. **Short-lived / batch jobs** — a job that runs for 5 seconds may finish before Prometheus scrapes it.
   The job **pushes** its metrics to the **Pushgateway** when it completes, and Prometheus scrapes the Pushgateway.
2. **Strict networking rules** — when the monitoring server is not allowed to open connections to the target
   (e.g. target is behind NAT/firewall), but the target can make outbound connections.

**Networking reminder:**
- Pull: `prometheus → app` — the target's security group must **allow inbound** on the exporter port
  (e.g. 9100) from the Prometheus server.
- Push: `app → server` — the target needs **outbound** access to the server.

---

## 4. How Prometheus uses the pull model (scraping)

**Scraping** = Prometheus sending an HTTP `GET` request to a target's metrics endpoint and storing the response.

```
every scrape_interval (e.g. 15s):

Prometheus ──HTTP GET /metrics──► exporter (on target) ──reads──► OS / app / DB
Prometheus ◄── text in Prometheus format ── exporter
Prometheus adds timestamp + labels (job, instance) → stores in TSDB
```

### Anatomy of a target endpoint

```
http://10.0.1.187:9100/metrics
└─┬─┘  └──────┬──────┘└──┬───┘
scheme     address      path
(http/https) (host:port) (default /metrics)
```

In config these map to `scheme`, `targets`, and `metrics_path`.

### What the `/metrics` response looks like

```
# HELP node_memory_MemAvailable_bytes Memory information field MemAvailable_bytes.
# TYPE node_memory_MemAvailable_bytes gauge
node_memory_MemAvailable_bytes 1.523456e+09
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 18342.53
node_cpu_seconds_total{cpu="0",mode="user"} 512.11
```

This plain-text format is called the **Prometheus exposition format** (OpenMetrics is its standardised successor).

### Labels Prometheus adds automatically

- `job` — the `job_name` from the scrape config
- `instance` — the `host:port` of the target

### Important timing settings

| Setting           | Default | Meaning                                            |
|-------------------|---------|----------------------------------------------------|
| `scrape_interval` | 1m (we use 15s) | How often to scrape                        |
| `scrape_timeout`  | 10s     | Give up on a scrape after this long (must be ≤ interval) |
| `evaluation_interval` | 1m (we use 15s) | How often alert/recording rules run    |

---

## 5. Exporters

### Why do we need exporters?

Targets are of many kinds — Linux servers, Windows, MySQL, Nginx, Java apps, network devices. Most of them
**don't speak Prometheus format**. Prometheus cannot understand each platform's internals.

An **exporter** is a small agent program that runs **alongside the target** and acts as a translator:

1. Prometheus sends a scrape request to the exporter.
2. The exporter **reads** the metrics from the target (e.g. `/proc` on Linux, `SHOW STATUS` on MySQL).
3. It **converts** them into Prometheus format.
4. It **responds** to Prometheus with that data.

> Exporters don't push and don't store anything. They answer Prometheus's pull request on demand.

### Ways to expose metrics

| Approach | Description | Example |
|---|---|---|
| **Direct instrumentation** | The app itself exposes `/metrics` using a Prometheus client library | Go/Java/Python apps using `prometheus_client`; Prometheus itself on `:9090/metrics`; Kubernetes components |
| **Exporter (sidecar/agent)** | Separate process translates a third-party system's stats | node_exporter, mysqld_exporter |
| **Pushgateway** | Short-lived jobs push metrics, Prometheus scrapes the gateway | cron jobs, batch jobs |

### Common exporters and default ports

| Exporter | Monitors | Default port |
|---|---|---|
| **node_exporter** | Linux/Unix host: CPU, memory, disk, filesystem, network | **9100** |
| **windows_exporter** | Windows host metrics | 9182 |
| **mysqld_exporter** | MySQL / MariaDB | 9104 |
| **postgres_exporter** | PostgreSQL | 9187 |
| **redis_exporter** | Redis | 9121 |
| **nginx-prometheus-exporter** | Nginx (stub_status) | 9113 |
| **jmx_exporter** | Java/JVM apps (heap, GC, threads) | configurable |
| **cAdvisor** | Container (Docker) resource usage | 8080 |
| **kube-state-metrics** | Kubernetes object state (deployments, pods) | 8080 |
| **blackbox_exporter** | Probes endpoints from outside: HTTP, HTTPS, DNS, TCP, ICMP | 9115 |
| **Prometheus itself** | Its own internal metrics | 9090 |

### Exporter categories (how to explain in an interview)

- **Host / OS exporters** — node_exporter, windows_exporter
- **Database exporters** — mysqld, postgres, mongodb, redis
- **Web server / proxy exporters** — nginx, apache, haproxy
- **Runtime exporters** — jmx_exporter for JVM
- **Container / orchestration exporters** — cAdvisor, kube-state-metrics
- **Probing (black-box) exporters** — blackbox_exporter: checks "is the website up, is DNS resolving,
  when does the TLS certificate expire, how long does it take to respond" — without knowing system internals.

**White-box vs black-box monitoring**
- **White-box** — we know the internals (CPU, memory, app counters). node_exporter, mysqld_exporter.
- **Black-box** — we only see the outside behaviour, like a user would. blackbox_exporter.

---

## 6. Installing Prometheus and Node Exporter

### Lab setup used in class

| Item | Value |
|---|---|
| Instance | `t3.small`, RHEL-9, 50 GB disk, `us-east-1` |
| VPC | `expense-vpc` |
| Security group | must allow **9090** (Prometheus UI) to us, and targets must allow **9100** from Prometheus |

### Standard directory layout

| Path | Purpose |
|---|---|
| `/opt/prometheus` | Prometheus binaries (artifact) |
| `/etc/prometheus/prometheus.yml` | Configuration file |
| `/var/lib/prometheus` | TSDB data storage |

Prometheus runs as a dedicated **system user** `prometheus` (no login shell) — never as root.

### Step 1 — Install Prometheus

```bash
# Pick the latest version from https://prometheus.io/download/
PROM_VERSION=3.5.0

sudo useradd --system --no-create-home --shell /sbin/nologin prometheus

cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz
tar -xzf prometheus-${PROM_VERSION}.linux-amd64.tar.gz

sudo mkdir -p /opt/prometheus /etc/prometheus /var/lib/prometheus
sudo cp prometheus-${PROM_VERSION}.linux-amd64/{prometheus,promtool} /opt/prometheus/
sudo cp prometheus-${PROM_VERSION}.linux-amd64/prometheus.yml /etc/prometheus/prometheus.yml

sudo chown -R prometheus:prometheus /opt/prometheus /etc/prometheus /var/lib/prometheus
```

### Step 2 — systemd service for Prometheus

`/etc/systemd/system/prometheus.service`

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/opt/prometheus/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=15d \
  --web.enable-lifecycle
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

Open `http://<prometheus-public-ip>:9090` → **Status → Targets** should show the `prometheus` job as **UP**.

### Step 3 — Install node_exporter on the target (e.g. MySQL server)

```bash
NODE_VERSION=1.9.1

sudo useradd --system --no-create-home --shell /sbin/nologin node_exporter

cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v${NODE_VERSION}/node_exporter-${NODE_VERSION}.linux-amd64.tar.gz
tar -xzf node_exporter-${NODE_VERSION}.linux-amd64.tar.gz
sudo mkdir -p /opt/node_exporter
sudo cp node_exporter-${NODE_VERSION}.linux-amd64/node_exporter /opt/node_exporter/
sudo chown -R node_exporter:node_exporter /opt/node_exporter
```

`/etc/systemd/system/node_exporter.service`

```ini
[Unit]
Description=Node Exporter
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
ExecStart=/opt/node_exporter/node_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
curl http://localhost:9100/metrics | head      # verify locally
```

### Step 4 — Network access

The **MySQL server's security group must allow inbound TCP 9100 from the Prometheus server**
(best: allow from the Prometheus server's security group or private IP, not from `0.0.0.0/0`).

Verify from the Prometheus server:

```bash
curl http://10.0.1.187:9100/metrics | head
```

---

## 7. Static configs in `prometheus.yml`

> The file is conventionally named `prometheus.yml` (`.yaml` also works — just make sure `--config.file` points to it).

**Static config** = we hard-code the list of targets in the config file. Simple and good for a few fixed servers.

### Full example

```yaml
global:
  scrape_interval: 15s        # how often to scrape targets
  evaluation_interval: 15s    # how often to evaluate rules
  scrape_timeout: 10s

scrape_configs:
  # Prometheus scraping itself
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          app: "prometheus"

  # node_exporter on our servers
  - job_name: "node"
    scheme: http              # default
    metrics_path: /metrics    # default
    static_configs:
      - targets: ["10.0.1.187:9100"]      # mysql
        labels:
          component: "mysql"
          env: "dev"
      - targets: ["10.0.1.25:9100", "10.0.1.40:9100"]   # backend, frontend
        labels:
          component: "app"
          env: "dev"
```

### How to read it

| Field | Meaning |
|---|---|
| `global` | Defaults applied to every job unless overridden |
| `scrape_configs` | List of **jobs**; each job is a group of similar targets |
| `job_name` | Becomes the `job` label on every series from these targets |
| `static_configs` | Hard-coded target groups |
| `targets` | List of `host:port` — becomes the `instance` label |
| `labels` | Extra labels attached to **all** series from those targets |
| `scheme`, `metrics_path` | Override `http` / `/metrics` if needed |

Resulting series:

```
up{job="node", instance="10.0.1.187:9100", component="mysql", env="dev"}  1
```

### Validate and reload safely

```bash
/opt/prometheus/promtool check config /etc/prometheus/prometheus.yml   # always validate first

sudo systemctl reload prometheus                    # sends SIGHUP, no restart needed
# or, when started with --web.enable-lifecycle:
curl -X POST http://localhost:9090/-/reload
```

Then check **Status → Targets** in the UI and run `up` in the query page.

### Limitation of static configs

In the cloud, servers come and go (autoscaling, re-creation, changing IPs). Editing the file by hand every time
doesn't scale — this is solved by **Service Discovery** (e.g. `ec2_sd_configs`) together with **relabel_configs**.

---

## 8. Service Discovery — EC2 SD

### What is Service Discovery?

**Service Discovery (SD)** is the mechanism by which Prometheus **finds its targets dynamically** instead of us
hard-coding IPs. Prometheus asks a source of truth (AWS, Kubernetes, Consul, a file…) "which targets exist right
now?" and keeps that list refreshed automatically.

| | Static config | Service discovery |
|---|---|---|
| Target list | Hard-coded IPs in `prometheus.yml` | Fetched from an API, refreshed periodically |
| New server added | Edit file + reload | Picked up automatically |
| Server replaced / IP changed | Stale target, shows DOWN | Old one disappears, new one appears |
| Good for | Few fixed servers, labs | Cloud, autoscaling, Kubernetes |

Common SD mechanisms: `ec2_sd_configs`, `kubernetes_sd_configs`, `consul_sd_configs`, `azure_sd_configs`,
`gce_sd_configs`, `file_sd_configs` (targets in a JSON/YAML file maintained by another tool).

### EC2 Service Discovery — how it works

```
Prometheus ──(AWS API: DescribeInstances)──► EC2
           ◄── list of instances + metadata (private IP, tags, AZ, instance id, state…)
Prometheus builds one target per instance: <private_ip>:<port>
           ──► relabel_configs ──► scrape http://<private_ip>:9100/metrics
```

Prerequisites:

1. **Exporters installed on the instances** — EC2 SD only *finds* the instances; something must still be listening
   on the port (e.g. node_exporter on 9100).
2. **Prometheus must be allowed to call the EC2 API.** Attach an **IAM Role** (instance profile) to the Prometheus
   server — never put access keys in the config.
3. **Security groups** on the targets must allow the exporter port from the Prometheus server.
4. **Tag the instances** so we can choose which ones to monitor (e.g. `Monitoring = true`).

Minimum IAM policy for the Prometheus server's role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeAvailabilityZones"
      ],
      "Resource": "*"
    }
  ]
}
```

### Our setup (v1.2)

- `mysql`, `backend`, `frontend` instances are created along with the application **and** the exporters.
- They are tagged: `Monitoring=true`, `Name`, `Tier`, `Component`, `Project`, `Environment`.
- The Prometheus server has an IAM role that can describe EC2 instances and availability zones.
- Prometheus uses EC2 SD to discover the instances with `Monitoring=true`.

### EC2 SD config

```yaml
scrape_configs:
  - job_name: "node"
    ec2_sd_configs:
      - region: us-east-1
        port: 9100                    # port to scrape on each discovered instance
        refresh_interval: 60s         # how often to re-query the EC2 API (default 60s)
        filters:                      # applied by the AWS API, before Prometheus sees the list
          - name: tag:Monitoring
            values: ["true"]
          - name: instance-state-name
            values: ["running"]
```

- `filters` use the same filter names as the AWS `DescribeInstances` API (`tag:<key>`, `instance-state-name`,
  `vpc-id`, …). Filtering here is cheaper than dropping targets later with relabeling.
- By default the target address is `<private_ip>:<port>` — Prometheus must be in the same VPC (or have a route to it).

### Metadata labels from EC2 SD

Along with each target, SD attaches **metadata labels** starting with `__meta_ec2_`. Useful ones:

| Label | Value |
|---|---|
| `__meta_ec2_tag_<TagKey>` | Value of each instance tag, e.g. `__meta_ec2_tag_Name` |
| `__meta_ec2_instance_id` | `i-0abc123…` |
| `__meta_ec2_private_ip` | Private IP |
| `__meta_ec2_public_ip` | Public IP (if any) |
| `__meta_ec2_availability_zone` | `us-east-1a` |
| `__meta_ec2_instance_type` | `t3.micro` |
| `__meta_ec2_instance_state` | `running` |
| `__meta_ec2_vpc_id`, `__meta_ec2_subnet_id` | Network info |

These labels are **not stored** with the metrics — they are thrown away after relabeling unless we copy them into
normal labels. That is exactly what `relabel_configs` is for.

In the UI, **Status → Service Discovery** shows every discovered target with its "Discovered labels" (before
relabeling) and "Target labels" (after relabeling) — very useful for debugging.

---

## 9. Relabeling — `relabel_configs` and `metric_relabel_configs`

### The idea

Prometheus treats **every target as a set of labels**. Before scraping, it can apply a list of edits (rules) to those
labels — rename, copy, keep or drop the whole target. This is **relabeling**.

Medical analogy:
- **Service discovery** → the list of patients who came to the hospital today.
- **relabel_configs** → the front desk, *before* the check-up: decide which patients get checked, write their
  name/ward on the file, send away those who don't need a check-up.
- **Scrape** → the actual tests, which produce a huge amount of information.
- **metric_relabel_configs** → *after* the tests: drop the readings nobody needs before filing the report.

### Labels starting with `__` (double underscore)

| Label type | Example | What happens |
|---|---|---|
| Starts with `__` | `__address__`, `__scheme__`, `__metrics_path__`, `__meta_ec2_*` | **Scrape settings / metadata.** Prometheus uses them to build the scrape request, then **throws them away** |
| Does not start with `__` | `job`, `instance`, `tier`, `env` | **Stored** on every series scraped from this target |

Special `__` labels you can set via relabeling:
- `__address__` — the `host:port` to scrape (becomes `instance` if `instance` isn't set)
- `__scheme__` — `http` / `https`
- `__metrics_path__` — `/metrics`
- `__param_<name>` — URL query parameters (used heavily with blackbox_exporter)

So, to keep a piece of metadata, **copy it from a `__meta_*` label into a normal label**.

### Anatomy of a relabel rule

```yaml
- source_labels: [label_a, label_b]   # values are joined with `separator` (default ";")
  separator: ";"
  regex: "(.*)"                       # default "(.*)" — matched against the joined value (fully anchored)
  target_label: new_label             # where to write the result
  replacement: "$1"                   # default "$1" — can use regex capture groups
  action: replace                     # default action
```

Rules run **one by one, top to bottom**; each rule sees the output of the previous one.

### Common actions

| Action | What it does |
|---|---|
| `replace` (default) | If `regex` matches, write `replacement` into `target_label` |
| `keep` | Keep the target **only if** `regex` matches the source value; drop it otherwise |
| `drop` | Drop the target if `regex` matches |
| `labelmap` | Copy every label whose **name** matches `regex` to a new name (e.g. all EC2 tags at once) |
| `labeldrop` | Remove labels whose name matches `regex` |
| `labelkeep` | Remove all labels whose name does **not** match `regex` |
| `hashmod` | Hash the source into N buckets — used to shard targets across Prometheus servers |

### Our config — EC2 SD + relabeling

```yaml
scrape_configs:
  - job_name: "node"
    ec2_sd_configs:
      - region: us-east-1
        port: 9100
        filters:
          - name: tag:Monitoring
            values: ["true"]
    relabel_configs:
      # no keep rule: every Monitoring=true instance is a node target
      # --- labels on every series (instance stays <private-ip>:9100) ---
      - source_labels: [__meta_ec2_tag_Tier]
        target_label: tier
      - source_labels: [__meta_ec2_tag_Component]
        target_label: component
      - source_labels: [__meta_ec2_tag_Project]
        target_label: project
      - source_labels: [__meta_ec2_tag_Environment]
        target_label: env
      - source_labels: [__meta_ec2_tag_Name]
        target_label: name
      - source_labels: [__meta_ec2_instance_id]
        target_label: instance_id
      - source_labels: [__meta_ec2_availability_zone]
        target_label: az
```

Result — every series from the MySQL server now carries meaningful labels (values depend on your tags):

```
up{job="node", instance="10.0.1.187:9100", tier="database", component="mysql",
   project="expense", env="dev", name="mysql", instance_id="i-0abc...", az="us-east-1a"}  1
```

Now we can query by meaning instead of by IP:

```promql
up{tier="backend"}                       # are all backend servers up?
count by (tier) (up{env="dev"} == 1)     # how many healthy servers per tier
```

### Selecting targets with `keep` / `drop`

Example — a job that scrapes **only backend** instances:

```yaml
  - job_name: "backend"
    ec2_sd_configs:
      - region: us-east-1
        port: 9100
        filters:
          - name: tag:Monitoring
            values: ["true"]
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Tier]
        regex: backend
        action: keep                      # every target whose Tier is not "backend" is dropped
```

Flow: SD returns all instances with `Monitoring=true` → the `keep` rule keeps only `Tier=backend` → only those
are scraped. `up{job="backend"}` then shows just the backend servers.

### Other useful patterns

```yaml
# Show the Name tag as `instance` instead of IP:port
- source_labels: [__meta_ec2_tag_Name]
  target_label: instance

# Copy ALL EC2 tags as labels in one rule (__meta_ec2_tag_Tier -> Tier, ...)
- regex: __meta_ec2_tag_(.+)
  action: labelmap

# Scrape the public IP instead of the private IP
- source_labels: [__meta_ec2_public_ip]
  regex: (.+)
  target_label: __address__
  replacement: "$1:9100"
```

> Be careful with `labelmap` on all tags — every tag becomes a label on every series (more cardinality and noise).

### `relabel_configs` vs `metric_relabel_configs`

| | `relabel_configs` | `metric_relabel_configs` |
|---|---|---|
| When | **Before** the scrape | **After** the scrape, before storing |
| Works on | **Targets** (their labels) | **Individual scraped series** |
| Can see `__meta_*` labels? | Yes | No (already gone) |
| Typical use | Choose targets, add labels from SD metadata, change address/path | Drop unneeded/expensive metrics, drop noisy labels |

Example — drop metrics we never use, to save storage:

```yaml
    metric_relabel_configs:
      - source_labels: [__name__]          # __name__ holds the metric name
        regex: "go_.*|node_scrape_collector_.*"
        action: drop
```

### Full flow

```
1. Service discovery      → list of targets + __meta_* labels
2. relabel_configs        → keep/drop targets, build labels, set __address__ / __metrics_path__
3. Scrape                 → GET <__scheme__>://<__address__><__metrics_path__>
4. metric_relabel_configs → drop/modify individual series
5. Store in TSDB          → only labels without "__" remain (the metric name is kept as __name__)
```

