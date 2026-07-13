# 15 — LOG01: Elasticsearch, Logstash, and Kibana

This document builds **LOG01**, a new Rocky Linux 10 VM, and installs the ELK stack — Elasticsearch, Logstash, and Kibana — all **built from source**. This is one of the heaviest source builds in this entire lab: Elasticsearch and Kibana are the actual upstream projects that Wazuh Indexer and Wazuh Dashboard in [`13`](./13-mon01-wazuh-manager-configuration.md) forked from, so expect comparable build complexity and time — budget real hours for this document, not minutes.

Every lesson learned building MON01's Wazuh stack and Prometheus/Grafana stack in [`12`](./12-mon01-zabbix-server-configuration.md), [`13`](./13-mon01-wazuh-manager-configuration.md), and [`14`](./14-mon01-prometheus-grafana-monitoring.md) is applied here from the start, rather than discovered live again:
- Go/Node build caches redirected to the data disk before any build starts, not after hitting a disk-full crisis.
- Node.js version pinned via `.node-version`/`nvm` for Kibana specifically, rather than assuming whatever Node version is already on the system will work.
- Build output paths confirmed with `find` before being assumed.
- A dedicated non-root build user, since Kibana's frontend tooling — like Grafana's and the Wazuh Dashboard's — refuses to run as root.
- Logrotate and boot-enable verification included from the start, not bolted on afterward.

| Component | Version used | Source |
|---|---|---|
| Elasticsearch | 9.4.3 (current stable — Elastic Stack ships Elasticsearch, Logstash, and Kibana in lockstep, always the same version across all three) | https://github.com/elastic/elasticsearch |
| Logstash | 9.4.3 | https://github.com/elastic/logstash |
| Kibana | 9.4.3 | https://github.com/elastic/kibana |

## Port planning

A fresh VM, so no conflicts to check against anything else running here — still worth listing explicitly, matching the practice established in [`14`](./14-mon01-prometheus-grafana-monitoring.md#port-planning--checked-against-everything-already-running-on-mon01):

| Port | Service |
|---|---|
| 9200 | Elasticsearch REST API |
| 9300 | Elasticsearch transport (inter-node, unused in this single-node deployment but still opened by default) |
| 5044 | Logstash Beats input (Filebeat from WEB01's Suricata data, once [`17`](./17-agent-deployment-all-vms.md) wires that up) |
| 9600 | Logstash monitoring API |
| 5601 | Kibana web UI |

---

## Table of contents

- [VM specification](#vm-specification)
- [Step 1 — Clone the golden baseline and provision disks](#step-1--clone-the-golden-baseline-and-provision-disks)
- [Step 2 — Set hostname and static IP](#step-2--set-hostname-and-static-ip)
- [Step 3 — Grow the root filesystem](#step-3--grow-the-root-filesystem)
- [Step 4 — Create swap](#step-4--create-swap)
- [Step 5 — Partition and mount the data disk](#step-5--partition-and-mount-the-data-disk)
- [Step 6 — Domain-join](#step-6--domain-join)
- [Step 7 — Add DNS records on DC01](#step-7--add-dns-records-on-dc01)
- [Step 8 — Install build dependencies](#step-8--install-build-dependencies)
- [Step 9 — Build Elasticsearch from source](#step-9--build-elasticsearch-from-source)
- [Step 10 — Configure Elasticsearch](#step-10--configure-elasticsearch)
- [Step 11 — Create the Elasticsearch systemd service](#step-11--create-the-elasticsearch-systemd-service)
- [Step 12 — Initialize Elasticsearch security](#step-12--initialize-elasticsearch-security)
- [Step 13 — Build Logstash from source](#step-13--build-logstash-from-source)
- [Step 14 — Configure Logstash](#step-14--configure-logstash)
- [Step 15 — Create the Logstash systemd service](#step-15--create-the-logstash-systemd-service)
- [Step 16 — Build Kibana from source](#step-16--build-kibana-from-source)
- [Step 17 — Configure Kibana](#step-17--configure-kibana)
- [Step 18 — Create the Kibana systemd service](#step-18--create-the-kibana-systemd-service)
- [Step 19 — Firewall](#step-19--firewall)
- [Step 20 — DNS](#step-20--dns)
- [Step 21 — Logrotate](#step-21--logrotate)
- [Step 22 — Disk cleanup](#step-22--disk-cleanup)
- [Step 23 — Final verification checklist](#step-23--final-verification-checklist)
- [Next step](#next-step)

---

## VM specification

| Setting | Value |
|---|---|
| VM name (VMware Library) | `LOG01_10.50` |
| OS hostname | `LOG01` |
| vCPU | 4 |
| RAM | 8 GB |
| Disk 1 (OS + swap) | 60 GB (40 GB OS + 20 GB swap) |
| Disk 2 (data) | 20 GB — Elasticsearch index data |
| NIC 1 (NAT) | DHCP |
| NIC 2 (Host-only) | Static — `192.168.10.50` |

---

## Step 1 — Clone the golden baseline and provision disks

Follow [`05-golden-baseline-rocky-linux-10.md`'s cloning instructions](./05-golden-baseline-rocky-linux-10.md#cloning-this-baseline-later):

1. Right-click `GoldenBaseline-Rocky10` → **Manage → Clone…** → **Create a full clone**.
2. Name the clone `LOG01_10.50`.
3. Adjust vCPU/RAM per the [VM specification](#vm-specification) above.
4. With the VM still powered off, expand the OS disk to **60 GB**: **VM → Settings → Hard Disk → Utilities → Expand…**.
5. Add the second disk while still powered off: **VM → Settings → Add… → Hard Disk → Create a new virtual disk**, size **20 GB**.
6. Power on the VM and reconnect via PuTTY using its NAT-assigned IP (`ip a`).

---

## Step 2 — Set hostname and static IP

```bash
sudo hostnamectl set-hostname LOG01
```

```bash
sudo nmcli con mod "Wired connection 1" ipv4.addresses 192.168.10.50/24
sudo nmcli con mod "Wired connection 1" ipv4.dns 192.168.10.10
sudo nmcli con mod "Wired connection 1" ipv4.method manual
sudo nmcli con up "Wired connection 1"
```

Stop NIC 1 (NAT) from polluting DNS resolution — required before domain-join in Step 6 will work:
```bash
nmcli con show
sudo nmcli con modify "ens160" ipv4.ignore-auto-dns yes
sudo nmcli con up "ens160"
cat /etc/resolv.conf
```

---

## Step 3 — Grow the root filesystem

```bash
sudo dnf install -y cloud-utils-growpart

lsblk
sudo vgs
sudo lvs
```

Using the actual names confirmed above (never assume `/dev/sda` or a volume group named `rl`):
```bash
sudo growpart /dev/nvme0n1 3
sudo pvresize /dev/nvme0n1p3
sudo lvextend -l +100%FREE /dev/mapper/<actual-vg-lv-path-from-lvs>
sudo xfs_growfs /
```

---

## Step 4 — Create swap

20 GB (matching the 8 GB RAM × 2, rounded up to the nearest 10 GB per this guide's convention — 16 GB rounds to 20 GB):

```bash
sudo fallocate -l 20G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

LOG01 runs nothing but JVM-based services (Elasticsearch, Logstash) — tune `vm.swappiness` down now, same reasoning as [MON01's Wazuh Indexer](./13-mon01-wazuh-manager-configuration.md#step-4--create-swap):
```bash
echo 'vm.swappiness=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## Step 5 — Partition and mount the data disk

Same procedure as [`12`'s Step 5](./12-mon01-zabbix-server-configuration.md#step-5--partition-and-mount-the-data-disk) — mount this now, before installing anything, so Elasticsearch's index data goes straight to the data disk from the start rather than needing a live migration later:

```bash
lsblk
```

```bash
sudo parted /dev/nvme0n2 --script mklabel gpt mkpart primary 0% 100%
lsblk
```
Confirm the new partition shows close to the full 20 GB before formatting — a partition showing only a few dozen MB means `parted`'s `mkpart` range wasn't applied correctly.

```bash
sudo mkfs.xfs /dev/nvme0n2p1
sudo mkdir -p /mnt/data
sudo mount /dev/nvme0n2p1 /mnt/data
```

```bash
sudo blkid /dev/nvme0n2p1
```
```bash
echo 'UUID=<uuid-from-above> /mnt/data xfs defaults 0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
```

Test the fstab entry before trusting it to survive a real reboot:
```bash
sudo umount /mnt/data
sudo mount -a
df -h /mnt/data
```

---

## Step 6 — Domain-join

```bash
sudo dnf install -y realmd sssd-common sssd-ad adcli krb5-workstation samba-common-tools oddjob oddjob-mkhomedir

sudo realm discover corp-lab.com.vn
sudo realm join -U administrator corp-lab.com.vn
```
Enter the `CORP-LAB\Administrator` password from [KeePass](./03-remote-access-tooling-setup.md#keepass) when prompted.

```bash
realm list
```

---

## Step 7 — Add DNS records on DC01

On **DC01**:
```powershell
Add-DnsServerResourceRecordA -ZoneName "corp-lab.com.vn" -Name "log01" -IPv4Address "192.168.10.50"
Add-DnsServerResourceRecordPtr -ZoneName "10.168.192.in-addr.arpa" -Name "50" -PtrDomainName "log01.corp-lab.com.vn."
```

---

## Step 8 — Install build dependencies

```bash
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y git curl wget openssl java-21-openjdk-devel
```
`java-21-openjdk-devel` is required for both Elasticsearch and Logstash's Gradle-based builds — both are JVM projects with their own build tooling that needs a real JDK, not just a runtime.

Go is not needed anywhere in this document (unlike [`14`](./14-mon01-prometheus-grafana-monitoring.md)) — nothing here is Go-based. Node.js is needed later, specifically for Kibana in [Step 16](#step-16--build-kibana-from-source), installed there via `nvm` against Kibana's own `.node-version` rather than a system-wide install here — the same lesson learned building Grafana in [`14`](./14-mon01-prometheus-grafana-monitoring.md#step-7--build-grafana-from-source), where installing Node.js too early and system-wide led to a version mismatch discovered only after a failed build.

Redirect Gradle's build cache to the data disk **before** building anything — Elasticsearch's dependency tree is large enough that this matters as much here as it did for Grafana in [`14`](./14-mon01-prometheus-grafana-monitoring.md#step-1--install-go):
```bash
mkdir -p /mnt/data/gradle-cache
mkdir -p ~/.gradle
echo "org.gradle.caching=true" >> ~/.gradle/gradle.properties
export GRADLE_USER_HOME=/mnt/data/gradle-cache
echo 'export GRADLE_USER_HOME=/mnt/data/gradle-cache' >> ~/.bashrc
```

---

## Step 9 — Build Elasticsearch from source

```bash
cd ~
git clone --depth 1 --branch v9.4.3 https://github.com/elastic/elasticsearch.git
cd elasticsearch
./gradlew localDistro
```
This is the heaviest build in this document — expect well over an hour, significant CPU/RAM usage, and a very large dependency download on first run. This is normal, matching what was seen building Wazuh Indexer in [`13`](./13-mon01-wazuh-manager-configuration.md#step-4--build-the-wazuh-indexer-package) — Elasticsearch is the actual upstream project that fork was built from, so comparable scale is expected, not a sign anything is wrong.

Locate the actual build output rather than assuming a path — the exact directory structure can shift between versions, the same lesson learned the hard way with both Prometheus's `consoles` directory and Grafana's `bin/linux/amd64` path in [`14`](./14-mon01-prometheus-grafana-monitoring.md):
```bash
find distribution -maxdepth 4 -iname "elasticsearch-9.4.3*" -type d 2>/dev/null
```

---

## Step 10 — Configure Elasticsearch

Install to `/opt`, using whatever path the `find` command above actually confirmed:
```bash
sudo cp -r <path-from-find-command-above> /opt/elasticsearch

sudo groupadd --system elasticsearch
sudo useradd --system -g elasticsearch -d /opt/elasticsearch -s /sbin/nologin elasticsearch
```

Point data at the mounted data disk from [Step 5](#step-5--partition-and-mount-the-data-disk):
```bash
sudo mkdir -p /mnt/data/elasticsearch
sudo chown elasticsearch:elasticsearch /mnt/data/elasticsearch
```

```bash
sudo tee /opt/elasticsearch/config/elasticsearch.yml << 'EOF'
cluster.name: corp-lab-logs
node.name: log01
path.data: /mnt/data/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 192.168.10.50
discovery.type: single-node
EOF

sudo mkdir -p /var/log/elasticsearch
sudo chown -R elasticsearch:elasticsearch /opt/elasticsearch /var/log/elasticsearch
```

Set the JVM heap to roughly half this VM's 8 GB RAM, leaving room for Logstash, Kibana, and the OS:
```bash
sudo tee /opt/elasticsearch/config/jvm.options.d/heap.options << 'EOF'
-Xms3g
-Xmx3g
EOF
```
Same formatting caution as every other JVM heap setting in this lab — no leading whitespace on either line, or the parser rejects the file outright.

---

## Step 11 — Create the Elasticsearch systemd service

```bash
sudo tee /etc/systemd/system/elasticsearch.service << 'EOF'
[Unit]
Description=Elasticsearch
After=network.target

[Service]
User=elasticsearch
Group=elasticsearch
LimitNOFILE=65535
LimitMEMLOCK=infinity
ExecStart=/opt/elasticsearch/bin/elasticsearch
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now elasticsearch
```

Wait 30–60 seconds for startup, then check:
```bash
sudo systemctl status elasticsearch
sudo tail -30 /var/log/elasticsearch/corp-lab-logs.log
```

---

## Step 12 — Initialize Elasticsearch security

Modern Elasticsearch enables security (TLS, authentication) by default — generate the initial `elastic` superuser password:
```bash
sudo /opt/elasticsearch/bin/elasticsearch-reset-password -u elastic --auto
```
Copy the generated password immediately into [KeePass](./03-remote-access-tooling-setup.md#keepass) under a new `LOG01_10.50` group, entry "Elasticsearch elastic user".

Verify:
```bash
curl -k -u elastic:'Password-From-Above' https://192.168.10.50:9200
```
Expect a JSON response describing the cluster, with `"tagline" : "You Know, for Search"`.

> If Elasticsearch started without TLS auto-configured (no certs generated, plain HTTP on 9200), that's a sign auto-configuration didn't trigger — this normally happens automatically on first start of a fresh data directory. Check `/var/log/elasticsearch/corp-lab-logs.log` for TLS-related errors before assuming everything is fine on HTTP; running a security-relevant service without encryption isn't the intended state here.

---

## Step 13 — Build Logstash from source

```bash
cd ~
git clone --depth 1 --branch v9.4.3 https://github.com/elastic/logstash.git
cd logstash
./gradlew assembleTarDistribution
```
Logstash's build is JRuby-based on top of the JVM — expect this to be lighter than Elasticsearch's build but still take a meaningful amount of time.

**`./gradlew assemble` alone isn't the right task here — confirmed the hard way.** It only compiles Logstash's core into a bare `.jar` (`build/libs/logstash-9.4.3.jar`), not a runnable distribution with `bin/`, `config/`, and the bundled JRuby runtime. `assembleTarDistribution` is the task that actually produces that full `.tar.gz` package, found by grepping this checked-out tree's own `build.gradle` for distribution-related task names (`grep -n "^task\|register(\"" build.gradle | grep -iE "tar|zip|dist"`) rather than guessing — the same lesson already learned with Prometheus's `consoles` directory and Grafana's `bin/linux/amd64` path in [`14`](./14-mon01-prometheus-grafana-monitoring.md).

```bash
find . -maxdepth 3 -iname "logstash-9.4.3*.tar.gz" 2>/dev/null
```

---

## Step 14 — Configure Logstash

```bash
sudo mkdir -p /opt/logstash
sudo tar -xzf <path-from-find-command-above> -C /opt/logstash --strip-components=1

sudo groupadd --system logstash
sudo useradd --system -g logstash -d /opt/logstash -s /sbin/nologin logstash
```

Create a pipeline that receives Filebeat input (from WEB01's Suricata data, once [`17-agent-deployment-all-vms.md`](./17-agent-deployment-all-vms.md) configures that Filebeat instance to ship here) and writes to the local Elasticsearch:
```bash
sudo mkdir -p /etc/logstash/conf.d
sudo tee /etc/logstash/conf.d/suricata.conf << 'EOF'
input {
  beats {
    port => 5044
  }
}

output {
  if "suricata" in [tags] {
    elasticsearch {
      hosts => ["https://192.168.10.50:9200"]
      user => "elastic"
      password => "Password-From-Step-12"
      ssl_certificate_authorities => ["/opt/elasticsearch/config/certs/http_ca.crt"]
      index => "suricata-%{+YYYY.MM.dd}"
    }
  } else if "winlogbeat" in [tags] {
    elasticsearch {
      hosts => ["https://192.168.10.50:9200"]
      user => "elastic"
      password => "Password-From-Step-12"
      ssl_certificate_authorities => ["/opt/elasticsearch/config/certs/http_ca.crt"]
      index => "winlogbeat-%{+YYYY.MM.dd}"
    }
  } else if "iis" in [tags] {
    elasticsearch {
      hosts => ["https://192.168.10.50:9200"]
      user => "elastic"
      password => "Password-From-Step-12"
      ssl_certificate_authorities => ["/opt/elasticsearch/config/certs/http_ca.crt"]
      index => "iis-%{+YYYY.MM.dd}"
    }
  } else {
    elasticsearch {
      hosts => ["https://192.168.10.50:9200"]
      user => "elastic"
      password => "Password-From-Step-12"
      ssl_certificate_authorities => ["/opt/elasticsearch/config/certs/http_ca.crt"]
      index => "beats-unclassified-%{+YYYY.MM.dd}"
    }
  }
}
EOF

sudo mkdir -p /var/log/logstash
sudo chown -R logstash:logstash /opt/logstash /etc/logstash /var/log/logstash
```
No Filebeat is actually pointed at port 5044 yet — this pipeline sits ready and idle until [`17`](./17-agent-deployment-all-vms.md) configures WEB01's Filebeat output (tagged `suricata`), Winlogbeat on the Windows VMs (tagged `winlogbeat`), and WINAPP01's own Filebeat instance for IIS logs (tagged `iis`) to ship here, fulfilling the integration [`08-web01-suricata-nids.md`](./08-web01-suricata-nids.md) originally noted as a forward pointer. A single Beats input on one port accepts connections from every Beat type simultaneously — the `tags`-based conditional above is what keeps their data separated into distinct indices rather than mixed together, and the `else` branch is a safety net catching anything that arrives untagged rather than silently dropping or misfiling it.

---

## Step 15 — Create the Logstash systemd service

```bash
sudo tee /etc/systemd/system/logstash.service << 'EOF'
[Unit]
Description=Logstash
After=network.target elasticsearch.service

[Service]
User=logstash
Group=logstash
Environment=LS_HOME=/opt/logstash
Environment=LS_SETTINGS_DIR=/opt/logstash/config
ExecStart=/opt/logstash/bin/logstash --path.settings /opt/logstash/config
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now logstash
sudo systemctl status logstash
```

Verify Logstash's own API responds (confirms the process started cleanly, independent of whether any data has flowed through it yet):
```bash
curl -s http://localhost:9600 | python3 -m json.tool
```

---

## Step 16 — Build Kibana from source

Same non-root requirement as Grafana in [`14`](./14-mon01-prometheus-grafana-monitoring.md#step-7--build-grafana-from-source) and the Wazuh Dashboard in [`13`](./13-mon01-wazuh-manager-configuration.md#set-up-a-non-root-build-user) — Kibana's frontend build tooling also refuses to run as root:
```bash
id builder 2>/dev/null || useradd -m -s /bin/bash builder

sudo mkdir -p /mnt/data/gradle-cache-builder
sudo chown builder:builder /mnt/data/gradle-cache-builder

su - builder
```

As `builder`:
```bash
git clone --depth 1 --branch v9.4.3 https://github.com/elastic/kibana.git
cd kibana
```

**Check `.node-version` before installing anything — don't assume a Node version.** This is the exact lesson learned building Grafana in [`14`](./14-mon01-prometheus-grafana-monitoring.md#step-7--build-grafana-from-source), where guessing at a Node version instead of checking the repo's own pinned version cost an entire failed build:
```bash
cat .node-version
```

Install `nvm` and the exact version that file specifies:
```bash
which nvm || (curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash && source ~/.bashrc)

nvm install "$(cat .node-version)"
nvm use "$(cat .node-version)"
node --version
```

Install Yarn v1 specifically (Kibana's bootstrap tooling expects it, not Yarn v2+/Berry):
```bash
npm install -g yarn@1
yarn --version
```

Bootstrap (Kibana's own term for `yarn install` plus its internal package-linking step):
```bash
yarn kbn bootstrap
```

Build:
```bash
node scripts/build --skip-os-packages
```
Expect this to take a considerable amount of time — Kibana's frontend is large, and this step compiles all of it.

Confirm the build output — check rather than assume, matching this document's established pattern:
```bash
find build -maxdepth 2 -type d 2>/dev/null
```

---

## Step 17 — Configure Kibana

Switch back to root:
```bash
exit
```

Install from whatever path [Step 16](#step-16--build-kibana-from-source)'s `find` command actually confirmed:
```bash
sudo cp -r /home/builder/kibana/build/<confirmed-subdirectory> /opt/kibana

sudo groupadd --system kibana
sudo useradd --system -g kibana -d /opt/kibana -s /sbin/nologin kibana
```

Generate a self-signed certificate for the browser-facing side of Kibana — separate from the certificates Kibana already uses to talk *to* Elasticsearch:
```bash
sudo mkdir -p /opt/kibana/certs
sudo openssl req -x509 -nodes -days 825 \
  -newkey rsa:2048 \
  -keyout /opt/kibana/certs/kibana.key \
  -out /opt/kibana/certs/kibana.crt \
  -subj "/C=VN/O=CorpLab/CN=kibana.corp-lab.com.vn"
```

```bash
sudo tee /opt/kibana/config/kibana.yml << 'EOF'
server.host: "0.0.0.0"
server.port: 5601
server.name: "log01"
server.ssl.enabled: true
server.ssl.certificate: /opt/kibana/certs/kibana.crt
server.ssl.key: /opt/kibana/certs/kibana.key

elasticsearch.hosts: ["https://192.168.10.50:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "Password-Set-Below"
elasticsearch.ssl.certificateAuthorities: ["/opt/elasticsearch/config/certs/http_ca.crt"]
EOF

sudo mkdir -p /var/log/kibana
sudo chown -R kibana:kibana /opt/kibana /var/log/kibana
```

Set the `kibana_system` service account password (a dedicated, lower-privilege account Kibana uses to talk to Elasticsearch — not the `elastic` superuser from [Step 12](#step-12--initialize-elasticsearch-security)):
```bash
sudo /opt/elasticsearch/bin/elasticsearch-reset-password -u kibana_system --auto
```
Copy the generated password into [KeePass](./03-remote-access-tooling-setup.md#keepass) alongside the `elastic` user's, and update `elasticsearch.password` in `kibana.yml` above to match.

---

## Step 18 — Create the Kibana systemd service

```bash
sudo tee /etc/systemd/system/kibana.service << 'EOF'
[Unit]
Description=Kibana
After=network.target elasticsearch.service

[Service]
User=kibana
Group=kibana
ExecStart=/opt/kibana/bin/kibana --config /opt/kibana/config/kibana.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now kibana
sudo systemctl status kibana
```

Verify:
```bash
curl -sk https://localhost:5601/api/status | python3 -m json.tool
```
Expect `"level": "available"` for the overall status.

---

## Step 19 — Firewall

```bash
sudo firewall-cmd --permanent --add-port=9200/tcp
sudo firewall-cmd --permanent --add-port=5044/tcp
sudo firewall-cmd --permanent --add-port=9600/tcp
sudo firewall-cmd --permanent --add-port=5601/tcp
sudo firewall-cmd --reload
```
`9300` (Elasticsearch transport) is intentionally **not** opened — this is a single-node deployment with no other Elasticsearch nodes to talk to, so there's nothing on the network that needs to reach it.

---

## Step 20 — DNS

The `kibana.corp-lab.com.vn` CNAME was already planned in [`06`'s CNAME table](./06-dc01-active-directory-dns-dhcp.md#step-5--configure-dns) — create it now that LOG01 actually exists:

On **DC01**:
```powershell
Add-DnsServerResourceRecordCName -ZoneName "corp-lab.com.vn" -Name "kibana" -HostNameAlias "log01.corp-lab.com.vn."
```

```bash
nslookup kibana.corp-lab.com.vn
```

---

## Step 21 — Logrotate

Elasticsearch, Logstash, and Kibana are all JVM/Node processes that write their own log files directly to disk with no built-in rotation assumed here (Elasticsearch specifically does have its own internal log4j2 rolling policy by default — verify rather than assume, same as [MON01's Wazuh Indexer check](./13-mon01-wazuh-manager-configuration.md#step-15--logrotate)):
```bash
grep -A3 "RollingFile\|SizeBasedTriggeringPolicy" /opt/elasticsearch/config/log4j2.properties
```
If that's already configured, Elasticsearch doesn't need an external logrotate rule. Logstash and Kibana do:

```bash
sudo tee /etc/logrotate.d/logstash << 'EOF'
/var/log/logstash/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 logstash logstash
    sharedscripts
    postrotate
        systemctl reload logstash >/dev/null 2>&1 || true
    endscript
}
EOF

sudo tee /etc/logrotate.d/kibana << 'EOF'
/var/log/kibana/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 kibana kibana
    sharedscripts
    postrotate
        systemctl reload kibana >/dev/null 2>&1 || true
    endscript
}
EOF

sudo logrotate -d /etc/logrotate.d/logstash
sudo logrotate -d /etc/logrotate.d/kibana
```

---

## Step 22 — Disk cleanup

Same lesson as every other from-source build in this lab — source trees and build caches run to many gigabytes and have no further use once the binaries are installed:
```bash
df -h /
du -sh ~/elasticsearch ~/logstash /home/builder/kibana /mnt/data/gradle-cache /mnt/data/gradle-cache-builder 2>/dev/null
```

```bash
rm -rf ~/elasticsearch ~/logstash /home/builder/kibana
df -h /
```
The Gradle caches on the data disk (`/mnt/data/gradle-cache*`) are worth keeping if you expect to rebuild any of these later — they're on the spacious data disk, not the tight OS disk, so there's less urgency to remove them compared to source trees on `/`.

---

## Step 23 — Final verification checklist

1. **All three services running, and enabled to start on boot:**
```bash
sudo systemctl status elasticsearch logstash kibana
sudo systemctl is-enabled elasticsearch logstash kibana
```

2. **Elasticsearch cluster health:**
```bash
curl -k -u elastic:'Password-From-Step-12' https://192.168.10.50:9200/_cluster/health?pretty
```
Expect `"status": "green"` or `"yellow"` (yellow is normal for a single-node deployment).

3. **Kibana reachable and reporting healthy:**
```bash
curl -sk https://localhost:5601/api/status | python3 -m json.tool
```

4. **Logstash's pipeline loaded without error** (repeat the check from [Step 15](#step-15--create-the-logstash-systemd-service)).

5. **DNS resolves:**
```bash
nslookup kibana.corp-lab.com.vn
```

6. **No leftover build artifacts on the OS disk:**
```bash
df -h /
```
Should be comfortably under 80% — if not, revisit [Step 22](#step-22--disk-cleanup).

7. **Credentials are stored, not memorized** — confirm the `elastic` and `kibana_system` passwords are saved in [KeePass](./03-remote-access-tooling-setup.md#keepass) under a `LOG01_10.50` group.

If all seven checks pass, the ELK stack is ready on LOG01 — Elasticsearch and Kibana running, and Logstash's Beats input sitting ready for WEB01's Suricata data once [`17-agent-deployment-all-vms.md`](./17-agent-deployment-all-vms.md) completes that connection.

---

## Next step

Continue to [`16-log02-opensearch-deployment.md`](./16-log02-opensearch-deployment.md) to build LOG02.
