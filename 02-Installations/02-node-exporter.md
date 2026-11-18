# 02. node_exporter — manual install, step by step

Lab: expense app on VMs (frontend, backend, MySQL behind the ALB) plus a separate monitoring VM running Prometheus.
Approach: install by hand first (v1), understand every file, automate in the next version.

**Series order:** 01 Prometheus (default config, no targets) → 02 node_exporter → 03 mysqld_exporter → 04 nginx exporter → back to 01, section 4: add the targets.

| Install on | Port | Version | Download source | Runs as user |
| --- | --- | --- | --- | --- |
| frontend, backend, mysql VMs (all 3) | 9100 | 1.12.1 | https://prometheus.io/download/#node_exporter | `node_exporter` |

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

## 2. node_exporter (frontend, backend, mysql VMs)

Repeat sections 2.1–2.6 on **each** of the three VMs.

### 2.1 Download and extract into /opt

Source: https://prometheus.io/download/#node_exporter (file `node_exporter-<version>.linux-amd64.tar.gz`).

```bash
cd /opt
VERSION=1.12.1
curl -fLO https://github.com/prometheus/node_exporter/releases/download/v${VERSION}/node_exporter-${VERSION}.linux-amd64.tar.gz
tar -xzf node_exporter-${VERSION}.linux-amd64.tar.gz
```

Make sure the file name says `linux-amd64`. A `darwin` file is the macOS build and fails with `Exec format error`.

### 2.2 Rename to a versioned folder, symlink, clean up

```bash
mv /opt/node_exporter-${VERSION}.linux-amd64 /opt/node_exporter-${VERSION}
rm -f /opt/node_exporter-${VERSION}.linux-amd64.tar.gz
ln -sfn /opt/node_exporter-${VERSION} /opt/node_exporter
ls -l /opt | grep node_exporter
/opt/node_exporter/node_exporter --version
```

### 2.3 Create a system user (no login, no home)

```bash
id node_exporter || useradd --system --no-create-home --shell /sbin/nologin node_exporter
chown -R node_exporter:node_exporter /opt/node_exporter-${VERSION}
```

Why: the exporter only reads. If it is ever compromised, it can't log in or write anywhere important.

### 2.4 systemd unit

```bash
cat > /etc/systemd/system/node_exporter.service <<'UNIT'
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/opt/node_exporter/node_exporter --web.listen-address=:9100
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
UNIT
```

Note: `<<'UNIT'` is quoted, so the shell writes the file exactly as typed.

### 2.5 Start and enable

```bash
systemctl daemon-reload
systemctl enable --now node_exporter
systemctl status node_exporter --no-pager
```

If it isn't `active (running)`, read the error first:

```bash
journalctl -u node_exporter -n 50 --no-pager
```

### 2.6 Verify locally

```bash
ss -tlnp | grep 9100
curl -s http://localhost:9100/metrics | grep -E '^node_load1|^node_uname_info'
```

---

## 3. Open the port (AWS security group)

| Security group | Inbound port | Source |
| --- | --- | --- |
| frontend SG | 9100 | monitoring SG |
| backend SG | 9100 | monitoring SG |
| mysql SG | 9100 | monitoring SG |

Test from the **monitoring VM**:

```bash
curl -s http://frontend.daws90s.shop:9100/metrics | head -5
curl -s http://backend.daws90s.shop:9100/metrics | head -5
curl -s http://mysql.daws90s.shop:9100/metrics | head -5
```

If `curl` hangs, it's the security group or VPC (the monitoring VM must be in the same VPC). If it says `connection refused`, the port is reachable but the exporter isn't running.

---

## 4. Troubleshooting

| Symptom | Likely cause | Check |
| --- | --- | --- |
| `Exec format error` in journalctl | Wrong build (darwin or arm64) | Tarball name must contain `linux-amd64`; re-download |
| `status=203/EXEC` | Wrong path in `ExecStart`, or binary not executable | `ls -l /opt/<tool>/`, check the symlink |
| `status=217/USER` | Service user doesn't exist | `id <user>` |
| `bind: address already in use` | Another process is on the port | `ss -tlnp \| grep <port>` |

Rule: **read the error first.** Don't add `-q` or `-s` to commands while debugging.

---

## 5. Upgrading later (the symlink payoff)

```bash
cd /opt
VERSION=<new-version>
# download + extract + rename to /opt/node_exporter-${VERSION}, delete tarball (as in 2.1–2.2)
chown -R node_exporter:node_exporter /opt/node_exporter-${VERSION}
ln -sfn /opt/node_exporter-${VERSION} /opt/node_exporter
systemctl restart node_exporter
```

Rollback: point the symlink back to the old folder and restart. No systemd edits needed.

**Next:** 03 mysqld_exporter.
