# 04. nginx-prometheus-exporter — manual install, step by step

Lab: expense app on VMs (frontend, backend, MySQL behind the ALB) plus a separate monitoring VM running Prometheus.
Approach: install by hand first (v1), understand every file, automate in the next version.

**Series order:** 01 Prometheus (default config, no targets) → 02 node_exporter → 03 mysqld_exporter → 04 nginx exporter → back to 01, section 4: add the targets.

| Install on | Port | Version | Download source | Runs as user |
| --- | --- | --- | --- | --- |
| frontend VM | 9113 | 1.5.3 | https://github.com/nginx/nginx-prometheus-exporter/releases (not on prometheus.io; maintained by NGINX) | `nginx_exporter` |

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

## 2. nginx-prometheus-exporter (frontend VM only)

nginx doesn't speak Prometheus. It has a tiny built-in status page (`stub_status`). The exporter reads that page and translates it.

```
Prometheus --(9113)--> nginx-prometheus-exporter --(127.0.0.1:8081/stub_status)--> nginx
```

### 2.1 Confirm nginx has the stub_status module

```bash
nginx -V 2>&1 | grep -o with-http_stub_status_module
```

You should see `with-http_stub_status_module`. Distro nginx packages include it.

### 2.2 Add a private status page

```bash
cat > /etc/nginx/conf.d/stub_status.conf <<'CONF'
server {
    listen 127.0.0.1:8081;
    server_name localhost;

    location /stub_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
CONF
```

Why `127.0.0.1:8081`: the status page is only for the exporter on the same VM. It is never exposed to the internet or the ALB.

### 2.3 Test and reload nginx

```bash
nginx -t
systemctl reload nginx
curl -s http://127.0.0.1:8081/stub_status
```

Expected output:

```
Active connections: 1
server accepts handled requests
 12 12 15
Reading: 0 Writing: 1 Waiting: 0
```

Explain each number to students: active connections, total accepted, total handled, total requests, and the reading/writing/waiting split.

### 2.4 Download and extract into /opt

Source: this exporter is **not** on prometheus.io/download (the Prometheus project doesn't maintain it). The official source is NGINX's own release page: https://github.com/nginx/nginx-prometheus-exporter/releases.

```bash
cd /opt
VERSION=1.5.3
curl -fLO https://github.com/nginx/nginx-prometheus-exporter/releases/download/v${VERSION}/nginx-prometheus-exporter_${VERSION}_linux_amd64.tar.gz
mkdir -p /opt/nginx-prometheus-exporter-${VERSION}
tar -xzf nginx-prometheus-exporter_${VERSION}_linux_amd64.tar.gz -C /opt/nginx-prometheus-exporter-${VERSION}
rm -f /opt/nginx-prometheus-exporter_${VERSION}_linux_amd64.tar.gz
```

Note the file name differs from the Prometheus exporters: underscores and `linux_amd64`. The tarball has no top-level folder (it would spill the binary, LICENSE and README straight into `/opt`), so we extract into the versioned folder with `-C`.

### 2.5 Symlink and system user

```bash
ln -sfn /opt/nginx-prometheus-exporter-${VERSION} /opt/nginx-prometheus-exporter
/opt/nginx-prometheus-exporter/nginx-prometheus-exporter --version
id nginx_exporter || useradd --system --no-create-home --shell /sbin/nologin nginx_exporter
chown -R nginx_exporter:nginx_exporter /opt/nginx-prometheus-exporter-${VERSION}
```

### 2.6 systemd unit

```bash
cat > /etc/systemd/system/nginx-prometheus-exporter.service <<'UNIT'
[Unit]
Description=NGINX Prometheus Exporter
Wants=network-online.target
After=network-online.target nginx.service

[Service]
User=nginx_exporter
Group=nginx_exporter
Type=simple
ExecStart=/opt/nginx-prometheus-exporter/nginx-prometheus-exporter \
  --nginx.scrape-uri=http://127.0.0.1:8081/stub_status \
  --web.listen-address=:9113
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
UNIT
```

### 2.7 Start, enable, verify

```bash
systemctl daemon-reload
systemctl enable --now nginx-prometheus-exporter
systemctl status nginx-prometheus-exporter --no-pager
ss -tlnp | grep 9113
curl -s http://localhost:9113/metrics | grep -E '^nginx_up|^nginx_connections_active|^nginx_http_requests_total'
```

`nginx_up 1` = the exporter could read `stub_status`.

---

## 3. Open the port (AWS security group)

| Security group | Inbound port | Source |
| --- | --- | --- |
| frontend SG | 9113 | monitoring SG |

Test from the **monitoring VM**:

```bash
curl -s http://frontend.daws90s.shop:9113/metrics | grep nginx_up
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
| `nginx_up 0` | `stub_status` not reachable at the scrape URI | `curl http://127.0.0.1:8081/stub_status` |
| `nginx -t` fails | Typo in `stub_status.conf` or module missing | `nginx -V 2>&1 \| grep -o with-http_stub_status_module` |

Rule: **read the error first.** Don't add `-q` or `-s` to commands while debugging.

---

## 5. Upgrading later (the symlink payoff)

```bash
cd /opt
VERSION=<new-version>
# download + mkdir /opt/nginx-prometheus-exporter-${VERSION} + tar -C + delete tarball (as in 2.4)
chown -R nginx_exporter:nginx_exporter /opt/nginx-prometheus-exporter-${VERSION}
ln -sfn /opt/nginx-prometheus-exporter-${VERSION} /opt/nginx-prometheus-exporter
systemctl restart nginx-prometheus-exporter
```

Rollback: point the symlink back to the old folder and restart. No systemd edits needed.

**Next:** back to 01 Prometheus, section 4 — add the targets.
