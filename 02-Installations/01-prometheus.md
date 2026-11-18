# 01. Prometheus server — manual install, step by step

Lab: expense app on VMs (frontend, backend, MySQL behind the ALB) plus a separate monitoring VM running Prometheus.
Approach: install by hand first (v1), understand every file, automate in the next version.

**Series order:** 01 Prometheus (default config, no targets) → 02 node_exporter → 03 mysqld_exporter → 04 nginx exporter → back to 01, section 4: add the targets.

| Install on | Port | Version | Download source | Runs as user |
| --- | --- | --- | --- | --- |
| monitoring VM | 9090 | 3.16.0 | https://prometheus.io/download/#prometheus | `prometheus` |

**File layout:** binary in `/opt/<tool>-<version>/`, stable symlink `/opt/<tool>` (systemd uses it), config in `/etc/<tool>/`, unit in `/etc/systemd/system/`. To upgrade, extract the new version, repoint the symlink, restart.

Check the download page before class and change only the `VERSION=` line if a newer version is out.

---

## 1. Before you start

### 1.1 Check the CPU architecture

```bash
uname -m
```

| Output | Download the file with |
| --- | --- |
| `x86_64` (t3.micro) | `linux-amd64` |
| `aarch64` (t4g / Graviton) | `linux-arm64` |

Never download `darwin` (that's macOS). If you do, systemd fails with `Exec format error`.

### 1.2 Become root and work directly in /opt

```bash
sudo -i
cd /opt
```

Everything is downloaded and extracted straight into `/opt`. The tarball is deleted after extracting, so `/opt` only keeps the versioned folders and symlinks.

---

## 2. Prometheus server (monitoring VM only)

We install Prometheus first with its **default config**: it scrapes only itself. The exporter targets are added in section 4, after files 02–04 are done and every exporter is running.

### 2.1 Download and extract into /opt

Source: https://prometheus.io/download/#prometheus (file `prometheus-<version>.linux-amd64.tar.gz`).

```bash
cd /opt
VERSION=3.16.0
curl -fLO https://github.com/prometheus/prometheus/releases/download/v${VERSION}/prometheus-${VERSION}.linux-amd64.tar.gz
tar -xzf prometheus-${VERSION}.linux-amd64.tar.gz
```

Make sure the file name says `linux-amd64`. A `darwin` file is the macOS build and fails with `Exec format error`.

### 2.2 Rename to a versioned folder, symlink, clean up

```bash
mv /opt/prometheus-${VERSION}.linux-amd64 /opt/prometheus-${VERSION}
rm -f /opt/prometheus-${VERSION}.linux-amd64.tar.gz
ln -sfn /opt/prometheus-${VERSION} /opt/prometheus
ls /opt/prometheus/
/opt/prometheus/prometheus --version
```

The folder holds `prometheus` (the server), `promtool` (config checker) and a sample `prometheus.yml`.

### 2.3 System user, config and data folders

```bash
id prometheus || useradd --system --no-create-home --shell /sbin/nologin prometheus
mkdir -p /etc/prometheus /var/lib/prometheus
cp /opt/prometheus/prometheus.yml /etc/prometheus/prometheus.yml
chown -R prometheus:prometheus /opt/prometheus-${VERSION} /etc/prometheus /var/lib/prometheus
```

| Path | Holds |
| --- | --- |
| `/opt/prometheus` | Binaries only (replaced on upgrade) |
| `/etc/prometheus/prometheus.yml` | **The live config** — the only file we edit |
| `/var/lib/prometheus` | TSDB data (survives upgrades) |

Why copy the config to `/etc`: an upgrade replaces `/opt/prometheus-<version>`, and your edits would be left behind in the old folder. Editing `/opt/prometheus/prometheus.yml` has no effect, because the service reads `/etc/prometheus/prometheus.yml`.

### 2.4 Look at the default config (no targets yet)

```bash
cat /etc/prometheus/prometheus.yml
```

Only one job, `prometheus`, scraping `localhost:9090`. Leave it as it is for now.

### 2.5 systemd unit

```bash
cat > /etc/systemd/system/prometheus.service <<'UNIT'
[Unit]
Description=Prometheus Server
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
  --web.listen-address=:9090 \
  --web.enable-lifecycle
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
UNIT
```

| Flag | Meaning |
| --- | --- |
| `--config.file` | Which config file is live |
| `--storage.tsdb.path` | Where metrics are stored on disk |
| `--storage.tsdb.retention.time=15d` | Delete data older than 15 days |
| `--web.listen-address=:9090` | UI and API port |
| `--web.enable-lifecycle` | Allows `curl -X POST .../-/reload` to reload the config without a restart |

### 2.6 Validate, start, enable

```bash
/opt/prometheus/promtool check config /etc/prometheus/prometheus.yml
systemctl daemon-reload
systemctl enable --now prometheus
systemctl status prometheus --no-pager
```

If it isn't `active (running)`, read the error first:

```bash
journalctl -u prometheus -n 50 --no-pager
```

### 2.7 Verify

```bash
ss -tlnp | grep 9090
curl -s http://localhost:9090/-/healthy
curl -s 'http://localhost:9090/api/v1/query?query=up'
```

Open `http://<monitoring-public-ip>:9090` in a browser (needs port 9090 open in the monitoring SG for your IP; see section 3). **Status → Target health** shows exactly one target: `prometheus`, `UP`.

---

## 3. Open the port (AWS security group)

| Security group | Inbound port | Source |
| --- | --- | --- |
| monitoring SG | 9090 (Prometheus UI) | your IP only |

Open `http://<monitoring-public-ip>:9090`. **Status → Target health** shows one target: `prometheus`, `UP`.

**Now stop here and install the exporters (files 02, 03, 04). Come back to section 4 when all of them are running.**

---

## 4. Add the targets to Prometheus (monitoring VM)

Do this only after files 02–04 are done and each exporter answers `curl` from the monitoring VM. Until now Prometheus has scraped only itself.

### 4.1 Back up the default config

```bash
cp /etc/prometheus/prometheus.yml /etc/prometheus/prometheus.yml.default
```

### 4.2 Write the new config

Written with a heredoc, so vim paste can't break the YAML indentation:

```bash
cat > /etc/prometheus/prometheus.yml <<'YAML'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["frontend.daws90s.shop:9100"]
        labels:
          tier: "frontend"
      - targets: ["backend.daws90s.shop:9100"]
        labels:
          tier: "backend"
      - targets: ["mysql.daws90s.shop:9100"]
        labels:
          tier: "mysql"

  - job_name: "nginx"
    static_configs:
      - targets: ["frontend.daws90s.shop:9113"]
        labels:
          tier: "frontend"

  - job_name: "mysql"
    static_configs:
      - targets: ["mysql.daws90s.shop:9104"]
        labels:
          tier: "mysql"

  - job_name: "backend"
    metrics_path: /metrics
    static_configs:
      - targets: ["backend.daws90s.shop:9101"]
        labels:
          tier: "backend"
YAML
chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

| Job | Targets | What it scrapes |
| --- | --- | --- |
| prometheus | localhost:9090 | Prometheus itself (the default job) |
| node | 3 VMs on 9100 | node_exporter, with a `tier` label per VM |
| nginx | frontend:9113 | nginx-prometheus-exporter |
| mysql | mysql:9104 | mysqld_exporter |

### 4.3 Validate and reload

```bash
/opt/prometheus/promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

The reload endpoint works because Prometheus runs with `--web.enable-lifecycle`. Without it, use `systemctl reload prometheus` (sends SIGHUP) or `systemctl restart prometheus`.

If `promtool` reports an error, fix it before reloading. A reload with a broken config is rejected, and Prometheus keeps running the old one; check `journalctl -u prometheus -n 20`.

---

## 5. Verify end to end

In the Prometheus UI (`http://<monitoring-ip>:9090`):

1. **Status → Target health**: every target shows `UP`.
2. Run these queries:

| Query | Expected |
| --- | --- |
| `up` | 1 for every target |
| `up{job="node"}` | 3 series, one per tier |
| `mysql_up` | 1 |
| `nginx_up` | 1 |
| `node_load1` | a small number per VM |
| `rate(nginx_http_requests_total[5m])` | > 0 after you open the app in a browser |

---

## 6. Troubleshooting

| Symptom | Likely cause | Check |
| --- | --- | --- |
| `Exec format error` in journalctl | Wrong build (darwin or arm64) | Tarball name must contain `linux-amd64`; re-download |
| `status=203/EXEC` | Wrong path in `ExecStart`, or binary not executable | `ls -l /opt/<tool>/`, check the symlink |
| `status=217/USER` | Service user doesn't exist | `id <user>` |
| `bind: address already in use` | Another process is on the port | `ss -tlnp \| grep <port>` |
| Target `DOWN`, `context deadline exceeded` | Security group / VPC blocks the port | `curl` from the monitoring VM hangs |
| Target `DOWN`, `connection refused` | Exporter not running on that VM | `systemctl status <service>` on that VM |
| Reload rejected | YAML error in the config | `/opt/prometheus/promtool check config /etc/prometheus/prometheus.yml` |
| Config change has no effect | Edited the wrong file | `systemctl cat prometheus \| grep config.file` |

Rule: **read the error first.** Don't add `-q` or `-s` to commands while debugging.

---

## 7. Upgrading later (the symlink payoff)

```bash
cd /opt
VERSION=<new-version>
# download + extract + rename to /opt/prometheus-${VERSION}, delete tarball (as in 2.1–2.2)
chown -R prometheus:prometheus /opt/prometheus-${VERSION}
ln -sfn /opt/prometheus-${VERSION} /opt/prometheus
systemctl restart prometheus
```

Rollback: point the symlink back to the old folder and restart. No systemd edits needed.

The config in `/etc/prometheus` and the data in `/var/lib/prometheus` are untouched by an upgrade.
