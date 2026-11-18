# 03. mysqld_exporter — manual install, step by step

Lab: expense app on VMs (frontend, backend, MySQL behind the ALB) plus a separate monitoring VM running Prometheus.
Approach: install by hand first (v1), understand every file, automate in the next version.

**Series order:** 01 Prometheus (default config, no targets) → 02 node_exporter → 03 mysqld_exporter → 04 nginx exporter → back to 01, section 4: add the targets.

| Install on | Port | Version | Download source | Runs as user |
| --- | --- | --- | --- | --- |
| mysql VM | 9104 | 0.20.0 | https://prometheus.io/download/#mysqld_exporter | `mysqld_exporter` |

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

## 2. mysqld_exporter (mysql VM only)

The exporter needs a MySQL login. We create a dedicated read-only user with a connection limit, so the exporter can never write data or hog connections.

### 2.1 Create the MySQL user

```bash
mysql -u root -p
```

```sql
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'Exp0rter@123' WITH MAX_USER_CONNECTIONS 3;
CREATE USER 'exporter'@'127.0.0.1' IDENTIFIED BY 'Exp0rter@123' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'127.0.0.1';
FLUSH PRIVILEGES;
EXIT;
```

| Grant | Why the exporter needs it |
| --- | --- |
| `PROCESS` | Read `SHOW PROCESSLIST`, InnoDB status |
| `REPLICATION CLIENT` | Read replica/binlog status |
| `SELECT ON *.*` | Read `performance_schema` and `information_schema` tables |

Test the login over TCP (the same way the exporter will connect):

```bash
mysql -u exporter -p -h 127.0.0.1 -e "SHOW GLOBAL STATUS LIKE 'Uptime';"
```

Use a stronger password than the sample above outside the lab.

### 2.2 Download and extract into /opt

Source: https://prometheus.io/download/#mysqld_exporter (file `mysqld_exporter-<version>.linux-amd64.tar.gz`).

```bash
cd /opt
VERSION=0.20.0
curl -fLO https://github.com/prometheus/mysqld_exporter/releases/download/v${VERSION}/mysqld_exporter-${VERSION}.linux-amd64.tar.gz
tar -xzf mysqld_exporter-${VERSION}.linux-amd64.tar.gz
```

### 2.3 Rename to a versioned folder, symlink, clean up

```bash
mv /opt/mysqld_exporter-${VERSION}.linux-amd64 /opt/mysqld_exporter-${VERSION}
rm -f /opt/mysqld_exporter-${VERSION}.linux-amd64.tar.gz
ln -sfn /opt/mysqld_exporter-${VERSION} /opt/mysqld_exporter
/opt/mysqld_exporter/mysqld_exporter --version
```

### 2.4 System user

```bash
id mysqld_exporter || useradd --system --no-create-home --shell /sbin/nologin mysqld_exporter
chown -R mysqld_exporter:mysqld_exporter /opt/mysqld_exporter-${VERSION}
```

### 2.5 Credentials file

```bash
mkdir -p /etc/mysqld_exporter
cat > /etc/mysqld_exporter/.my.cnf <<'CNF'
[client]
user=exporter
password=Exp0rter@123
host=127.0.0.1
port=3306
CNF
chown mysqld_exporter:mysqld_exporter /etc/mysqld_exporter/.my.cnf
chmod 600 /etc/mysqld_exporter/.my.cnf
ls -l /etc/mysqld_exporter/.my.cnf
```

Why `host=127.0.0.1` and not `localhost`: with `localhost`, the MySQL client tries the Unix socket, and the socket path often doesn't match. `127.0.0.1` forces TCP.

Why `chmod 600`: the file holds a password; only the exporter user should read it.

### 2.6 systemd unit

```bash
cat > /etc/systemd/system/mysqld_exporter.service <<'UNIT'
[Unit]
Description=Prometheus MySQL Exporter
Wants=network-online.target
After=network-online.target mysqld.service

[Service]
User=mysqld_exporter
Group=mysqld_exporter
Type=simple
ExecStart=/opt/mysqld_exporter/mysqld_exporter \
  --config.my-cnf=/etc/mysqld_exporter/.my.cnf \
  --web.listen-address=:9104
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
UNIT
```

If your MySQL service is named `mysql` instead of `mysqld`, change `After=` accordingly (`systemctl list-units | grep -i mysql`).

### 2.7 Start, enable, verify

```bash
systemctl daemon-reload
systemctl enable --now mysqld_exporter
systemctl status mysqld_exporter --no-pager
ss -tlnp | grep 9104
curl -s http://localhost:9104/metrics | grep -E '^mysql_up|^mysql_global_status_uptime'
```

`mysql_up 1` = the exporter logged into MySQL. `mysql_up 0` = the exporter is running but can't log in: check the password, the grants and `host=127.0.0.1`.

**Teaching point:** Prometheus `up{job="mysql"} == 1` only means the exporter answered. `mysql_up == 1` means MySQL answered. UP ≠ healthy.

---

## 3. Open the port (AWS security group)

| Security group | Inbound port | Source |
| --- | --- | --- |
| mysql SG | 9104 | monitoring SG |

Test from the **monitoring VM**:

```bash
curl -s http://mysql.daws90s.shop:9104/metrics | grep mysql_up
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
| `mysql_up 0` | Wrong password, missing grants, or `host=localhost` | `journalctl -u mysqld_exporter`, test with `mysql -h 127.0.0.1 -u exporter -p` |
| `Access denied for user 'exporter'` | User created for the wrong host | `SELECT user,host FROM mysql.user WHERE user='exporter';` |

Rule: **read the error first.** Don't add `-q` or `-s` to commands while debugging.

---

## 5. Upgrading later (the symlink payoff)

```bash
cd /opt
VERSION=<new-version>
# download + extract + rename to /opt/mysqld_exporter-${VERSION}, delete tarball (as in 2.2–2.3)
chown -R mysqld_exporter:mysqld_exporter /opt/mysqld_exporter-${VERSION}
ln -sfn /opt/mysqld_exporter-${VERSION} /opt/mysqld_exporter
systemctl restart mysqld_exporter
```

Rollback: point the symlink back to the old folder and restart. No systemd edits needed.

**Next:** 04 nginx exporter.
