# 16 — LOG02: OpenSearch Deployment

This document builds **LOG02**, a new Rocky Linux 10 VM, and installs **OpenSearch** and **OpenSearch Dashboards** — both built from source. This is the second of two competing log analytics stacks in this lab, alongside LOG01's Elastic Stack ([`15`](./15-log01-elasticsearch-logstash-kibana.md)) — deliberately built side by side rather than choosing one, for hands-on comparison between the two.

**Good news relative to [`15`](./15-log01-elasticsearch-logstash-kibana.md) and [`13`](./13-mon01-wazuh-manager-configuration.md):** vanilla OpenSearch bundles its default plugins (including the security plugin) directly into the standard build task — unlike Wazuh's fork in [`13`](./13-mon01-wazuh-manager-configuration.md#step-4--build-the-wazuh-indexer-package), which split plugins into separate repositories assembled by a dedicated (and, at the time, poorly documented for the beta this lab first attempted) packaging pipeline. Building plain upstream OpenSearch is much closer to a normal single-repo build.

| Component | Version used | Source |
|---|---|---|
| OpenSearch | 3.7.0 (current stable) | https://github.com/opensearch-project/OpenSearch |
| OpenSearch Dashboards | 3.7.0 (matches the Data node version — Dashboards and OpenSearch are versioned in lockstep) | https://github.com/opensearch-project/OpenSearch-Dashboards |

## Port planning

| Port | Service |
|---|---|
| 9200 | OpenSearch REST API |
| 9300 | OpenSearch transport (inter-node, unused in this single-node deployment) |
| 5601 | OpenSearch Dashboards web UI |

No conflict with LOG01's Kibana (also 5601) since these are separate VMs.

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
- [Step 9 — Build OpenSearch from source](#step-9--build-opensearch-from-source)
- [Step 10 — Configure OpenSearch](#step-10--configure-opensearch)
- [Step 11 — Create the OpenSearch systemd service](#step-11--create-the-opensearch-systemd-service)
- [Step 12 — Initialize OpenSearch security](#step-12--initialize-opensearch-security)
- [Step 13 — Build OpenSearch Dashboards from source](#step-13--build-opensearch-dashboards-from-source)
- [Step 14 — Configure OpenSearch Dashboards](#step-14--configure-opensearch-dashboards)
- [Step 15 — Create the OpenSearch Dashboards systemd service](#step-15--create-the-opensearch-dashboards-systemd-service)
- [Step 16 — Firewall](#step-16--firewall)
- [Step 17 — DNS](#step-17--dns)
- [Step 18 — Logrotate](#step-18--logrotate)
- [Step 19 — Disk cleanup](#step-19--disk-cleanup)
- [Step 20 — Final verification checklist](#step-20--final-verification-checklist)
- [Next step](#next-step)

---

## VM specification

| Setting | Value |
|---|---|
| VM name (VMware Library) | `LOG02_10.51` |
| OS hostname | `LOG02` |
| vCPU | 4 |
| RAM | 8 GB |
| Disk 1 (OS + swap) | 60 GB (40 GB OS + 20 GB swap) |
| Disk 2 (data) | 20 GB — OpenSearch data |
| NIC 1 (NAT) | DHCP |
| NIC 2 (Host-only) | Static — `192.168.10.51` |

---

## Step 1 — Clone the golden baseline and provision disks

Same procedure as [`15`'s Step 1](./15-log01-elasticsearch-logstash-kibana.md#step-1--clone-the-golden-baseline-and-provision-disks):

1. Right-click `GoldenBaseline-Rocky10` → **Manage → Clone…** → **Create a full clone**.
2. Name the clone `LOG02_10.51`.
3. Adjust vCPU/RAM per the [VM specification](#vm-specification) above.
4. Expand the OS disk to **60 GB** while powered off.
5. Add a second **20 GB** disk while powered off.
6. Power on and reconnect via PuTTY.

---

## Step 2 — Set hostname and static IP

```bash
sudo hostnamectl set-hostname LOG02
```

```bash
sudo nmcli con mod "Wired connection 1" ipv4.addresses 192.168.10.51/24
sudo nmcli con mod "Wired connection 1" ipv4.dns 192.168.10.10
sudo nmcli con mod "Wired connection 1" ipv4.method manual
sudo nmcli con up "Wired connection 1"

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
```bash
sudo growpart /dev/nvme0n1 3
sudo pvresize /dev/nvme0n1p3
sudo lvextend -l +100%FREE /dev/mapper/<actual-vg-lv-path-from-lvs>
sudo xfs_growfs /
```

---

## Step 4 — Create swap

```bash
sudo fallocate -l 20G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

echo 'vm.swappiness=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## Step 5 — Partition and mount the data disk

```bash
lsblk
sudo parted /dev/nvme0n2 --script mklabel gpt mkpart primary 0% 100%
lsblk
```
Confirm the partition shows close to the full 20 GB before formatting.

```bash
sudo mkfs.xfs /dev/nvme0n2p1
sudo mkdir -p /mnt/data
sudo mount /dev/nvme0n2p1 /mnt/data

sudo blkid /dev/nvme0n2p1
echo 'UUID=<uuid-from-above> /mnt/data xfs defaults 0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload

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
realm list
```

---

## Step 7 — Add DNS records on DC01

On **DC01**:
```powershell
Add-DnsServerResourceRecordA -ZoneName "corp-lab.com.vn" -Name "log02" -IPv4Address "192.168.10.51"
Add-DnsServerResourceRecordPtr -ZoneName "10.168.192.in-addr.arpa" -Name "51" -PtrDomainName "log02.corp-lab.com.vn."
```

---

## Step 8 — Install build dependencies

```bash
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y git curl wget openssl java-21-openjdk-devel
```

Redirect Gradle's build cache to the data disk before building anything, same reasoning as [`15`'s Step 8](./15-log01-elasticsearch-logstash-kibana.md#step-8--install-build-dependencies):
```bash
mkdir -p /mnt/data/gradle-cache
export GRADLE_USER_HOME=/mnt/data/gradle-cache
echo 'export GRADLE_USER_HOME=/mnt/data/gradle-cache' >> ~/.bashrc
```

---

## Step 9 — Build OpenSearch from source

```bash
cd ~
git clone --depth 1 --branch 3.7.0 https://github.com/opensearch-project/OpenSearch.git
cd OpenSearch
./gradlew :distribution:archives:linux-tar:assemble
```
Expect a long build (though the earlier build of Wazuh Indexer in [`13`](./13-mon01-wazuh-manager-configuration.md) — a fork of this exact codebase — took roughly an hour; this should be comparable or somewhat faster given no Wazuh-specific plugin overlay to also compile).

```bash
find distribution/archives/linux-tar/build/distributions -iname "*.tar.gz"
```
**Correction to an assumption made earlier in this document: the security plugin is *not* bundled into this build, confirmed the hard way.** `plugins/` comes back completely empty after installing — unlike a Wazuh Indexer *package* (which bundles it during the official assemble pipeline), plain OpenSearch built directly from this repo's own `linux-tar` task ships as a bare core only. The fix is simpler than Wazuh's multi-repo assembly, though: `opensearch-security` is published as a standard plugin on Maven Central and installs via the normal plugin mechanism — covered in [Step 12](#step-12--initialize-opensearch-security).

---

## Step 10 — Configure OpenSearch

```bash
sudo mkdir -p /opt/opensearch
sudo tar -xzf <path-from-find-command-above> -C /opt/opensearch --strip-components=1

sudo groupadd --system opensearch
sudo useradd --system -g opensearch -d /opt/opensearch -s /sbin/nologin opensearch
```

```bash
sudo mkdir -p /mnt/data/opensearch
sudo chown opensearch:opensearch /mnt/data/opensearch
```

```bash
sudo tee /opt/opensearch/config/opensearch.yml << 'EOF'
cluster.name: corp-lab-logs-alt
node.name: log02
path.data: /mnt/data/opensearch
path.logs: /var/log/opensearch
network.host: 192.168.10.51
discovery.type: single-node
EOF

sudo mkdir -p /var/log/opensearch
sudo chown -R opensearch:opensearch /opt/opensearch /var/log/opensearch
```

```bash
printf '%s\n' '-Xms3g' '-Xmx3g' | sudo tee -a /opt/opensearch/config/jvm.options
cat -A /opt/opensearch/config/jvm.options | grep -A1 "Xms\|Xmx"
```
Same leading-whitespace caution as every other JVM heap setting in this lab.

---

## Step 11 — Create the OpenSearch systemd service

```bash
sudo tee /etc/systemd/system/opensearch.service << 'EOF'
[Unit]
Description=OpenSearch
After=network.target

[Service]
User=opensearch
Group=opensearch
LimitNOFILE=65535
LimitMEMLOCK=infinity
ExecStart=/opt/opensearch/bin/opensearch
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now opensearch
sudo systemctl status opensearch
```

---

## Step 12 — Initialize OpenSearch security

**Install the security plugin — it isn't bundled into the plain `linux-tar` build**, per the correction in [Step 9](#step-9--build-opensearch-from-source). It's published on Maven Central and installs through OpenSearch's normal plugin mechanism, the same tool used to install any other OpenSearch plugin:
```bash
cd /opt/opensearch
sudo ./bin/opensearch-plugin install --batch org.opensearch.plugin:opensearch-security:3.7.0.0
```
> If this specific version string doesn't resolve (plugin release versions can lag slightly behind OpenSearch core's own version — check `https://repo1.maven.org/maven2/org/opensearch/plugin/opensearch-security/` directly for what's actually published if `3.7.0.0` isn't there by the time you're reading this), that's a real possibility worth checking for rather than assuming the exact version always lines up.

Installing `opensearch-security` prints a warning about a missing `workload-management` dependency — install that too, using its simpler official install syntax (no explicit version needed; the plugin installer resolves the matching one on its own, and note this artifact's name has **no** `opensearch-` prefix, unlike `opensearch-security`):
```bash
sudo ./bin/opensearch-plugin install --batch workload-management
```

Confirm both installed:
```bash
ls /opt/opensearch/plugins/
```
Expect `opensearch-security` and `workload-management`.

**Set the admin password before running the install script — this OpenSearch version requires it upfront, rather than defaulting to a demo password you change afterward.** The tools also need their execute bit set explicitly; a Maven-installed plugin doesn't set this the way a package install would:
```bash
sudo chmod +x /opt/opensearch/plugins/opensearch-security/tools/*.sh

export OPENSEARCH_INITIAL_ADMIN_PASSWORD='Your-Strong-Password-Here'
/opt/opensearch/plugins/opensearch-security/tools/install_demo_configuration.sh -y
```
Without `OPENSEARCH_INITIAL_ADMIN_PASSWORD` set, this fails with `No custom admin password found` rather than falling back to a demo `admin`/`admin` credential — a deliberate hardening change versus older OpenSearch versions. Copy this password into [KeePass](./03-remote-access-tooling-setup.md#keepass) under a new `LOG02_10.51` group, entry "OpenSearch admin", immediately — this is the real `admin` password, not a placeholder to be changed later.

The script itself prints the exact `securityadmin.sh` command needed next, including the actual certificate filenames it generated — run what it prints rather than assuming the filenames below match exactly. **Add `-h`/`-p` explicitly, pointing at this VM's actual IP, not the default `localhost`** — `network.host: 192.168.10.51` in [Step 10](#step-10--configure-opensearch) makes OpenSearch bind only that specific address, not `127.0.0.1`/`localhost` as well, so the tool's default connection target (`localhost:9200`) fails with a misleadingly generic `Seems there is no OpenSearch running` error even though the cluster is actually healthy and reachable — confirmed the hard way by checking `ss -tlnp` and finding the listening socket bound only to the real IP:
```bash
sudo "/opt/opensearch/plugins/opensearch-security/tools/securityadmin.sh" \
  -cd "/opt/opensearch/config/opensearch-security" \
  -icl \
  -key "/opt/opensearch/config/kirk-key.pem" \
  -cert "/opt/opensearch/config/kirk.pem" \
  -cacert "/opt/opensearch/config/root-ca.pem" \
  -nhnv \
  -h 192.168.10.51 \
  -p 9200
```
> A handful of `ERROR`/exception stack traces (`ClosedSelectorException`, `ConcurrentModificationException`) printed *after* `Done with success` are harmless connection-pool cleanup noise as the tool exits — not a sign anything actually failed, as long as `Done with success` appeared first.

> **If OpenSearch itself crashes on startup with `NoClassDefFoundError: org/bouncycastle/jcajce/provider/BouncyCastleFipsProvider`** (checked via `sudo tail -f /var/log/opensearch/<cluster-name>.log` after a `systemctl restart opensearch`), this means the Maven-based plugin install in this step didn't pull in every transitive dependency the way a full package install would — specifically the BouncyCastle FIPS crypto JARs `opensearch-security` needs. Confirmed the hard way: three JARs were missing and had to be added manually. Two were available directly from this repo's own build output (adjust the path if your `OpenSearch` checkout lives somewhere other than `~/OpenSearch`):
> ```bash
> sudo cp ~/OpenSearch/distribution/archives/linux-tar/build/install/opensearch-*/lib/tools/plugin-cli/bc-fips-2.1.2.jar /opt/opensearch/plugins/opensearch-security/
> ```
> The other two weren't present anywhere in the built tree and needed downloading directly from Maven Central — check `~/OpenSearch/libs/ssl-config/licenses/*.sha1` filenames for the exact version numbers actually expected (checksums are committed there even when the JARs themselves aren't bundled) rather than guessing:
> ```bash
> cd /opt/opensearch/plugins/opensearch-security/
> sudo curl -O https://repo1.maven.org/maven2/org/bouncycastle/bcutil-fips/2.1.4/bcutil-fips-2.1.4.jar
> sudo curl -O https://repo1.maven.org/maven2/org/bouncycastle/bcpkix-fips/2.1.9/bcpkix-fips-2.1.9.jar
> ```
> Also confirm the plugin's shell scripts are actually executable if you haven't already followed this step's earlier `chmod +x` — a Maven-installed plugin doesn't set this the way a package install does, and a missing execute bit produces a plain `command not found` that has nothing to do with the path being wrong.
>
> Restart OpenSearch after adding any of these JARs before retrying `securityadmin.sh`.

Verify:
```bash
curl -k -u admin:'Your-New-Strong-Password' https://192.168.10.51:9200
```

---

## Step 13 — Build OpenSearch Dashboards from source

Same non-root requirement as Kibana in [`15`](./15-log01-elasticsearch-logstash-kibana.md#step-16--build-kibana-from-source):
```bash
id builder 2>/dev/null || useradd -m -s /bin/bash builder

sudo mkdir -p /mnt/data/gradle-cache-builder
sudo chown builder:builder /mnt/data/gradle-cache-builder

su - builder
```

As `builder`:
```bash
git clone --depth 1 --branch 3.7.0 https://github.com/opensearch-project/OpenSearch-Dashboards.git
cd OpenSearch-Dashboards
```

**Check `.nvmrc` before installing Node — same lesson from Grafana and Kibana**, don't assume a version:
```bash
cat .nvmrc
```

```bash
which nvm || curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```
**`source ~/.bashrc` alone isn't reliably enough to load `nvm` into the current shell right after a fresh install — confirmed the hard way** (`nvm: command not found` even immediately after the installer finished). Load it explicitly instead, exactly as the installer's own output suggests:
```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm --version
```

```bash
nvm install "$(cat .nvmrc)"
nvm use "$(cat .nvmrc)"
node --version
```

```bash
npm install -g yarn
yarn --version
```

```bash
yarn osd bootstrap
```
Same SSL/network transient-failure pattern as Grafana's `yarn osd bootstrap` in [`14`](./14-mon01-prometheus-grafana-monitoring.md#step-7--build-grafana-from-source) can happen here too — delete `node_modules` and retry if it fails partway through:
```bash
rm -rf node_modules
yarn osd bootstrap
```

```bash
yarn build --linux --skip-os-packages
```

Confirm the output — check rather than assume:
```bash
find target -maxdepth 1 -iname "*.tar.gz" 2>/dev/null
```
Expect **four** archives despite the `--linux` flag (`linux-x64`, `linux-arm64`, `darwin-x64`, `darwin-arm64`) — this build target still produces all supported platforms regardless of that flag. Use the `linux-x64` one in [Step 14](#step-14--configure-opensearch-dashboards).

---

## Step 14 — Configure OpenSearch Dashboards

```bash
exit
```

```bash
sudo mkdir -p /opt/opensearch-dashboards
sudo tar -xzf /home/builder/OpenSearch-Dashboards/target/opensearch-dashboards-3.7.0-SNAPSHOT-linux-x64.tar.gz -C /opt/opensearch-dashboards --strip-components=1

sudo groupadd --system opensearch-dashboards
sudo useradd --system -g opensearch-dashboards -d /opt/opensearch-dashboards -s /sbin/nologin opensearch-dashboards
```

**Copy the CA certificate into Dashboards' own config directory first — the config below references it, and skipping this step causes Dashboards to fail connecting to OpenSearch with `ConnectionError: unable to verify the first certificate`, confirmed live:**
```bash
sudo cp /opt/opensearch/config/root-ca.pem /opt/opensearch-dashboards/config/
```

Generate a self-signed certificate for the browser-facing side of Dashboards — separate from the certificate copied above, which is for Dashboards' own connection *to* OpenSearch:
```bash
sudo mkdir -p /opt/opensearch-dashboards/certs
sudo openssl req -x509 -nodes -days 825 \
  -newkey rsa:2048 \
  -keyout /opt/opensearch-dashboards/certs/dashboards.key \
  -out /opt/opensearch-dashboards/certs/dashboards.crt \
  -subj "/C=VN/O=CorpLab/CN=opensearch.corp-lab.com.vn"
```

```bash
sudo tee /opt/opensearch-dashboards/config/opensearch_dashboards.yml << 'EOF'
server.host: "0.0.0.0"
server.port: 5601
server.name: "log02"
server.ssl.enabled: true
server.ssl.certificate: /opt/opensearch-dashboards/certs/dashboards.crt
server.ssl.key: /opt/opensearch-dashboards/certs/dashboards.key

opensearch.hosts: ["https://192.168.10.51:9200"]
opensearch.ssl.verificationMode: certificate
opensearch.ssl.certificateAuthorities: ["/opt/opensearch-dashboards/config/root-ca.pem"]
opensearch.username: "kibanaserver"
opensearch.password: "kibanaserver"
opensearch.requestHeadersWhitelist: ["securitytenant","Authorization"]
EOF

sudo mkdir -p /var/log/opensearch-dashboards
sudo chown -R opensearch-dashboards:opensearch-dashboards /opt/opensearch-dashboards /var/log/opensearch-dashboards
```
`kibanaserver`/`kibanaserver` is the demo configuration's default internal service account from [Step 12](#step-12--initialize-opensearch-security)'s `install_demo_configuration.sh` — leave this as-is for the lab, or rotate it the same way the `admin` password was changed if you'd rather not use the demo default long-term.

---

## Step 15 — Create the OpenSearch Dashboards systemd service

```bash
sudo tee /etc/systemd/system/opensearch-dashboards.service << 'EOF'
[Unit]
Description=OpenSearch Dashboards
After=network.target opensearch.service

[Service]
User=opensearch-dashboards
Group=opensearch-dashboards
ExecStart=/opt/opensearch-dashboards/bin/opensearch-dashboards
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now opensearch-dashboards
sudo systemctl status opensearch-dashboards
```

Verify:
```bash
curl -sk https://localhost:5601/api/status | python3 -m json.tool
```

---

## Step 16 — Firewall

```bash
sudo firewall-cmd --permanent --add-port=9200/tcp
sudo firewall-cmd --permanent --add-port=5601/tcp
sudo firewall-cmd --reload
```
`9300` intentionally not opened, same reasoning as [`15`'s Step 19](./15-log01-elasticsearch-logstash-kibana.md#step-19--firewall) — single-node, nothing else needs to reach the transport port.

---

## Step 17 — DNS

On **DC01**:
```powershell
Add-DnsServerResourceRecordCName -ZoneName "corp-lab.com.vn" -Name "opensearch" -HostNameAlias "log02.corp-lab.com.vn."
```

```bash
nslookup opensearch.corp-lab.com.vn
```

---

## Step 18 — Logrotate

Same pattern as [`15`'s Step 21](./15-log01-elasticsearch-logstash-kibana.md#step-21--logrotate) — check OpenSearch's own built-in log4j2 rotation before assuming an external rule is needed:
```bash
grep -A3 "RollingFile\|SizeBasedTriggeringPolicy" /opt/opensearch/config/log4j2.properties
```

OpenSearch Dashboards (Node.js, like Kibana) needs an external rule:
```bash
sudo tee /etc/logrotate.d/opensearch-dashboards << 'EOF'
/var/log/opensearch-dashboards/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 opensearch-dashboards opensearch-dashboards
    sharedscripts
    postrotate
        systemctl reload opensearch-dashboards >/dev/null 2>&1 || true
    endscript
}
EOF

sudo logrotate -d /etc/logrotate.d/opensearch-dashboards
```

---

## Step 19 — Disk cleanup

```bash
df -h /
du -sh ~/OpenSearch /home/builder/OpenSearch-Dashboards /mnt/data/gradle-cache /mnt/data/gradle-cache-builder 2>/dev/null

rm -rf ~/OpenSearch /home/builder/OpenSearch-Dashboards
df -h /
```

---

## Step 20 — Final verification checklist

1. **Both services running, and enabled to start on boot:**
```bash
sudo systemctl status opensearch opensearch-dashboards
sudo systemctl is-enabled opensearch opensearch-dashboards
```

2. **OpenSearch cluster health:**
```bash
curl -k -u admin:'Your-New-Strong-Password' https://192.168.10.51:9200/_cluster/health?pretty
```
Expect `green` or `yellow`.

3. **OpenSearch Dashboards reachable:**
```bash
curl -sk https://localhost:5601/api/status | python3 -m json.tool
```

4. **DNS resolves:**
```bash
nslookup opensearch.corp-lab.com.vn
```

5. **No leftover build artifacts:**
```bash
df -h /
```

6. **Credentials are stored, not memorized** — confirm the `admin` password is saved in [KeePass](./03-remote-access-tooling-setup.md#keepass) under a `LOG02_10.51` group.

If all six checks pass, OpenSearch and OpenSearch Dashboards are ready on LOG02 — the second log analytics stack running alongside LOG01's Elastic Stack, for direct comparison between the two.

---

## Next step

Continue to [`17-agent-deployment-all-vms.md`](./17-agent-deployment-all-vms.md) to roll out Zabbix Agent 2, Wazuh Agent, and node_exporter to every VM built so far, completing the monitoring coverage this lab has been building toward.
