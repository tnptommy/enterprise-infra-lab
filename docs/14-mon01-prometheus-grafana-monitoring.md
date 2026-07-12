# 14 — MON01: Prometheus & Grafana Monitoring

This document adds **Prometheus** and **Grafana** to MON01, alongside Zabbix ([`12`](./12-mon01-zabbix-server-configuration.md)) and Wazuh ([`13`](./13-mon01-wazuh-manager-configuration.md)) — a separate metrics-monitoring ecosystem, not a replacement for Zabbix. Both are built from source, consistent with the rest of this VM's software.

No new VM is needed — this continues directly on MON01. This document also installs **node_exporter** on MON01 itself, both for self-monitoring and to establish the pattern [`17-agent-deployment-all-vms.md`](./17-agent-deployment-all-vms.md) will repeat on every other VM in this lab.

| Component | Version used | Source |
|---|---|---|
| Prometheus | 3.13.1 (current LTS, supported through July 2027) | https://github.com/prometheus/prometheus |
| Grafana | 13.1.0 (current stable) | https://github.com/grafana/grafana |
| node_exporter | 1.9.1 (current stable) | https://github.com/prometheus/node_exporter |

## Port planning — checked against everything already running on MON01

MON01 already has several services bound to non-default ports after [`12`](./12-mon01-zabbix-server-configuration.md) and [`13`](./13-mon01-wazuh-manager-configuration.md) — worth listing explicitly before adding three more, rather than assuming the defaults are free:

| Port | Already in use by |
|---|---|
| 443 | Wazuh Dashboard |
| 1514, 1515 | Wazuh Manager (agent events/enrollment) |
| 8443 | Zabbix Frontend (HTTPS-only, per [`12`'s Step 19](./12-mon01-zabbix-server-configuration.md#step-19--enable-https-only-access)) |
| 9200 | Wazuh Indexer |
| 10050, 10051 | Zabbix Agent 2 |
| 55000 | Wazuh API |
| 80, 8080 | *(both disabled — neither service uses plain HTTP on this VM)* |

None of these overlap with Prometheus's, Grafana's, or node_exporter's defaults:

| Port | This document |
|---|---|
| 9443 | Grafana web UI (HTTPS-only) |
| 9090 | Prometheus web UI / API |
| 9100 | node_exporter (scraped by Prometheus, not browsed directly) |

---

## Table of contents

- [Step 1 — Install Go](#step-1--install-go)
- [Step 2 — Build Prometheus from source](#step-2--build-prometheus-from-source)
- [Step 3 — Configure Prometheus](#step-3--configure-prometheus)
- [Step 4 — Create the Prometheus systemd service](#step-4--create-the-prometheus-systemd-service)
- [Step 5 — Install node_exporter on MON01](#step-5--install-node_exporter-on-mon01)
- [Step 6 — Install Node.js and Yarn for the Grafana frontend](#step-6--install-nodejs-and-yarn-for-the-grafana-frontend)
- [Step 7 — Build Grafana from source](#step-7--build-grafana-from-source)
- [Step 8 — Configure Grafana](#step-8--configure-grafana)
- [Step 9 — Create the Grafana systemd service](#step-9--create-the-grafana-systemd-service)
- [Step 10 — Add Prometheus as a Grafana data source](#step-10--add-prometheus-as-a-grafana-data-source)
- [Step 11 — Firewall](#step-11--firewall)
- [Step 12 — DNS](#step-12--dns)
- [Step 13 — Final verification checklist](#step-13--final-verification-checklist)
- [Next step](#next-step)

---

## Step 1 — Install Go

Both Prometheus and Grafana's backend are Go projects, and both need a reasonably current Go toolchain — unlike Filebeat's old 7.10.2 codebase in [`13`](./13-mon01-wazuh-manager-configuration.md#step-7--install-and-configure-filebeat), there's no old-version pinning concern here, so install directly from golang.org:

```bash
cd ~
wget https://go.dev/dl/go1.26.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.26.5.linux-amd64.tar.gz
echo 'export PATH=/usr/local/go/bin:$PATH' | sudo tee /etc/profile.d/go.sh
source /etc/profile.d/go.sh
go version
```
Expect `go version go1.26.5 linux/amd64`.

---

## Step 2 — Build Prometheus from source

```bash
cd ~
git clone --depth 1 --branch v3.13.1 https://github.com/prometheus/prometheus.git
cd prometheus
make build
```
This compiles both `prometheus` (the server) and `promtool` (its config-validation/admin CLI) — expect several minutes, considerably lighter than the Wazuh Dashboard build in [`13`](./13-mon01-wazuh-manager-configuration.md#step-8--build-the-wazuh-dashboard-package).

Install the binaries and the web UI's static assets:
```bash
sudo mkdir -p /opt/prometheus
sudo cp prometheus promtool /opt/prometheus/
sudo cp -r consoles console_libraries /opt/prometheus/

sudo groupadd --system prometheus
sudo useradd --system -g prometheus -d /opt/prometheus -s /sbin/nologin prometheus
```

---

## Step 3 — Configure Prometheus

Point Prometheus's own data directory at the data disk mounted in [`12`'s Step 5](./12-mon01-zabbix-server-configuration.md#step-5--partition-and-mount-the-data-disk) — time-series data grows continuously, exactly the kind of thing that disk was set aside for:
```bash
sudo mkdir -p /mnt/data/prometheus
sudo chown prometheus:prometheus /mnt/data/prometheus
```

```bash
sudo mkdir -p /etc/prometheus
sudo tee /etc/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance: 'mon01'
EOF

sudo chown -R prometheus:prometheus /etc/prometheus
```
The `node_exporter` job above only has MON01 itself as a target for now — [`17-agent-deployment-all-vms.md`](./17-agent-deployment-all-vms.md) adds a target line here for every other VM once node_exporter is deployed to each.

---

## Step 4 — Create the Prometheus systemd service

```bash
sudo tee /etc/systemd/system/prometheus.service << 'EOF'
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/opt/prometheus/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/mnt/data/prometheus \
  --web.console.templates=/opt/prometheus/consoles \
  --web.console.libraries=/opt/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

Verify:
```bash
curl -s http://localhost:9090/-/healthy
```
Expect `Prometheus Server is Healthy.`

---

## Step 5 — Install node_exporter on MON01

node_exporter is a small, self-contained Go binary (no source build needed — the project ships pre-built static binaries, and there's nothing meaningfully "built from source" about a single static Go binary with no configuration compiled in):

```bash
cd ~
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
tar -xzf node_exporter-1.9.1.linux-amd64.tar.gz
sudo cp node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/

sudo useradd --system -s /sbin/nologin node_exporter 2>/dev/null

sudo tee /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

Verify Prometheus is actually scraping it — check the target's `up` status rather than just that the process is running:
```bash
curl -s 'http://localhost:9090/api/v1/query?query=up{job="node_exporter"}' | grep -o '"value":\[[^]]*\]'
```
Expect the value `1` (not `0`) — confirms Prometheus successfully scraped node_exporter, not just that both processes happen to be alive independently.

---

## Step 6 — Install Node.js and Yarn for the Grafana frontend

```bash
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
sudo dnf install -y nodejs
node --version

sudo npm install -g yarn
```
(Same NodeSource approach as [`13`'s Dashboard build](./13-mon01-wazuh-manager-configuration.md#build-1--the-dashboard-base) — Rocky Linux 10 dropped `dnf module` streams entirely, so this is the reliable way to pin a specific Node major version.)

---

## Step 7 — Build Grafana from source

Same non-root requirement as the Wazuh Dashboard build in [`13`](./13-mon01-wazuh-manager-configuration.md#set-up-a-non-root-build-user) — Grafana's frontend build tooling also refuses to run as root. Reuse the `builder` user created there if this VM already has it; otherwise create it now:
```bash
id builder 2>/dev/null || useradd -m -s /bin/bash builder
chown -R builder:builder ~/prometheus 2>/dev/null
su - builder
```

As `builder`:
```bash
git clone --depth 1 --branch v13.1.0 https://github.com/grafana/grafana.git
cd grafana
export PATH=/usr/local/go/bin:$PATH

yarn install --immutable
```
This is considerably lighter than the Wazuh Dashboard's multi-repo plugin assembly in [`13`](./13-mon01-wazuh-manager-configuration.md#step-8--build-the-wazuh-dashboard-package) — Grafana is a single self-contained repository with one straightforward build.

Build the backend (Go) and frontend (TypeScript/React) separately:
```bash
go run build.go build
yarn build
```
Expect this to take a while (the frontend build in particular) — normal, not a sign of a problem.

Confirm both halves built:
```bash
ls bin/linux-amd64/grafana
ls public/build/*.js | head -3
```

---

## Step 8 — Configure Grafana

Switch back to root:
```bash
exit
```

```bash
sudo mkdir -p /opt/grafana
sudo cp -r /home/builder/grafana/bin/linux-amd64/* /opt/grafana/
sudo cp -r /home/builder/grafana/public /opt/grafana/
sudo cp -r /home/builder/grafana/conf /opt/grafana/

sudo groupadd --system grafana
sudo useradd --system -g grafana -d /opt/grafana -s /sbin/nologin grafana
```

Point Grafana's data directory at the data disk, same reasoning as Prometheus in [Step 3](#step-3--configure-prometheus):
```bash
sudo mkdir -p /mnt/data/grafana
sudo chown grafana:grafana /mnt/data/grafana
```

Generate a self-signed certificate, same pattern as Zabbix's in [`12`'s Step 19](./12-mon01-zabbix-server-configuration.md#step-19--enable-https-only-access):
```bash
sudo mkdir -p /opt/grafana/certs
sudo openssl req -x509 -nodes -days 825 \
  -newkey rsa:2048 \
  -keyout /opt/grafana/certs/grafana.key \
  -out /opt/grafana/certs/grafana.crt \
  -subj "/C=VN/O=CorpLab/CN=grafana.corp-lab.com.vn"
```

```bash
sudo mkdir -p /etc/grafana
sudo tee /etc/grafana/grafana.ini << 'EOF'
[paths]
data = /mnt/data/grafana
logs = /var/log/grafana
plugins = /mnt/data/grafana/plugins

[server]
protocol = https
http_port = 9443
domain = grafana.corp-lab.com.vn
cert_file = /opt/grafana/certs/grafana.crt
cert_key = /opt/grafana/certs/grafana.key

[security]
admin_user = admin
admin_password = admin
EOF

sudo mkdir -p /var/log/grafana
sudo chown -R grafana:grafana /etc/grafana /var/log/grafana /mnt/data/grafana /opt/grafana/certs
sudo chmod 400 /opt/grafana/certs/grafana.key
```
The default `admin`/`admin` credentials are changed in [Step 13](#step-13--final-verification-checklist) — Grafana forces a password change on first login by default, so this isn't a lingering security gap the way an unchanged Zabbix/Wazuh default password would be.

---

## Step 9 — Create the Grafana systemd service

```bash
sudo tee /etc/systemd/system/grafana.service << 'EOF'
[Unit]
Description=Grafana
After=network.target

[Service]
User=grafana
Group=grafana
Type=simple
WorkingDirectory=/opt/grafana
ExecStart=/opt/grafana/grafana server \
  --config=/etc/grafana/grafana.ini \
  --homepath=/opt/grafana
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now grafana
sudo systemctl status grafana
```

Verify:
```bash
curl -sk https://localhost:9443/api/health
```
Expect a JSON response with `"database": "ok"`.

---

## Step 10 — Add Prometheus as a Grafana data source

1. Browse to `https://192.168.10.40:9443` (accept the self-signed certificate warning — see [Step 11](#step-11--firewall) for the firewall step that makes this reachable, and [Step 13](#step-13--final-verification-checklist) for locking down the default credentials).
2. Log in with `admin`/`admin`, set a new password when prompted.
3. **Connections → Data sources → Add data source → Prometheus**.
4. **Prometheus server URL**: `http://localhost:9090`.
5. **Save & test** — expect a green "Successfully queried the Prometheus API" confirmation.

---

## Step 11 — Firewall

```bash
sudo firewall-cmd --permanent --add-port=9443/tcp
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --reload
```
`9100` (node_exporter) is intentionally **not** opened here — it only needs to be reachable from Prometheus itself on the same host (`localhost:9100` in [Step 3](#step-3--configure-prometheus)'s scrape config), not from the network at large. This changes once [`17-agent-deployment-all-vms.md`](./17-agent-deployment-all-vms.md) deploys node_exporter to *other* VMs, where Prometheus on MON01 needs to reach it remotely — that firewall rule belongs in that later document, on each of those VMs, not here.

---

## Step 12 — DNS

Add a CNAME for Grafana on DC01, matching the pattern already established for every other tool on this VM in [`06`'s CNAME table](./06-dc01-active-directory-dns-dhcp.md#step-5--configure-dns) — Prometheus itself doesn't get one, since its web UI is a debugging/administration interface Grafana front-ends for actual dashboard use, not something meant for regular browsing:

On **DC01**:
```powershell
Add-DnsServerResourceRecordCName -ZoneName "corp-lab.com.vn" -Name "grafana" -HostNameAlias "mon01.corp-lab.com.vn."
```

Verify from MON01 or any domain-joined machine:
```bash
nslookup grafana.corp-lab.com.vn
```

---

## Step 13 — Final verification checklist

1. **All three services running:**
```bash
sudo systemctl status prometheus grafana node_exporter
```

2. **Prometheus is healthy and scraping successfully** (repeat the checks from [Step 4](#step-4--create-the-prometheus-systemd-service) and [Step 5](#step-5--install-node_exporter-on-mon01)).

3. **Grafana is reachable and the Prometheus data source works** (repeat the check from [Step 10](#step-10--add-prometheus-as-a-grafana-data-source)).

4. **Default Grafana password has been changed** — log in at `https://grafana.corp-lab.com.vn:9443`, confirm it's no longer accepting `admin`/`admin`.

5. **No port conflicts with Zabbix or Wazuh:**
```bash
sudo ss -tlnp | grep -E ":9443|:9090|:9100"
```
Confirm exactly one process owns each port, and none of them are `httpd`, `zabbix_server`, or any `wazuh-*`/`java` process — a genuine conflict would show as a bind failure in that service's own systemd status instead, but this is worth checking directly given how much port-juggling this VM has already needed across [`12`](./12-mon01-zabbix-server-configuration.md) and [`13`](./13-mon01-wazuh-manager-configuration.md).

6. **DNS resolves:**
```bash
nslookup grafana.corp-lab.com.vn
```

7. **Credentials are stored, not memorized** — confirm the new Grafana `admin` password is saved in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `MON01_10.40` group, entry "Grafana admin".

8. **Disk cleanup** — same lesson as [`13`'s Dashboard build](./13-mon01-wazuh-manager-configuration.md#step-8--build-the-wazuh-dashboard-package): Grafana's `node_modules` and build output run several GB. Check disk usage and clean up the source trees now that both binaries are installed:
```bash
df -h /
du -sh ~/prometheus /home/builder/grafana 2>/dev/null
rm -rf ~/prometheus /home/builder/grafana
df -h /
```

If all eight checks pass, Prometheus and Grafana are ready on MON01, running alongside Zabbix ([`12`](./12-mon01-zabbix-server-configuration.md)) and Wazuh ([`13`](./13-mon01-wazuh-manager-configuration.md)) — three independent monitoring ecosystems sharing this one VM without port conflicts.

---

## Next step

Continue to [`15-log01-elasticsearch-logstash-kibana.md`](./15-log01-elasticsearch-logstash-kibana.md) to build LOG01.
