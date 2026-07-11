# 13 — MON01: Wazuh Manager Configuration

This document adds **Wazuh 4.14.6** to MON01 alongside the Zabbix stack from [`12`](./12-mon01-zabbix-server-configuration.md) — Wazuh Indexer (an OpenSearch fork), Wazuh Manager (the analysis engine agents report to), Filebeat (ships Manager alerts to the Indexer), and Wazuh Dashboard (the web UI). Every component is **built from source**, using Wazuh's own officially documented packaging pipeline rather than downloading pre-built packages.

No new VM or network configuration is needed here — this continues directly on MON01 as already built in [`12`](./12-mon01-zabbix-server-configuration.md). A new disk-space consideration: building the Indexer and Dashboard from source needs meaningful scratch space (Docker images, Node/Gradle caches) — confirm at least 15–20 GB free on the OS disk before starting, beyond what's already accounted for in MON01's [VM specification](./12-mon01-zabbix-server-configuration.md#vm-specification).

| Component | Version used | Source |
|---|---|---|
| Wazuh Manager | 4.14.6, built from source | https://github.com/wazuh/wazuh |
| Wazuh Indexer | 4.14.6, built from source via Wazuh's official Docker packaging pipeline | https://documentation.wazuh.com/current/development/packaging/generate-indexer-package.html |
| Wazuh Dashboard | 4.14.6, built from source via Wazuh's official Docker packaging pipeline | https://documentation.wazuh.com/current/development/packaging/generate-dashboard-package.html |
| Filebeat | 7.10.2 (version pinned by Wazuh's own compatibility matrix — do not use a newer Filebeat) | https://packages.wazuh.com/4.x/tpl/wazuh/filebeat/ |

> **Why this looks different from a typical from-source guide, and different from this document's own earlier attempt at Wazuh 5.0:** an earlier version of this document attempted building Wazuh **5.0.0 (beta)** entirely from source and hit a genuine wall — 5.0 splits Indexer functionality across several repositories assembled by an internal Docker tool that wasn't consistently tagged or documented for external use at the beta stage. **Wazuh 4.14.6 doesn't have that problem** — Wazuh publishes an official, versioned, documented build pipeline for exactly this release, used to produce the very `.rpm` packages available on `packages.wazuh.com`. This document follows that official pipeline directly (Docker-based `build.sh`/`assemble.sh` for the Indexer, a parameterized Docker build for the Dashboard) rather than improvising, so what you get by the end is functionally identical to the official package — just compiled by you, from source you control, on this VM.

---

## Table of contents

- [Step 1 — Install Docker](#step-1--install-docker)
- [Step 2 — Build Wazuh Manager from source](#step-2--build-wazuh-manager-from-source)
- [Step 3 — Generate certificates](#step-3--generate-certificates)
- [Step 4 — Build the Wazuh Indexer package](#step-4--build-the-wazuh-indexer-package)
- [Step 5 — Install and configure Wazuh Indexer](#step-5--install-and-configure-wazuh-indexer)
- [Step 6 — Initialize Indexer security](#step-6--initialize-indexer-security)
- [Step 7 — Install and configure Filebeat](#step-7--install-and-configure-filebeat)
- [Step 8 — Build the Wazuh Dashboard package](#step-8--build-the-wazuh-dashboard-package)
- [Step 9 — Install and configure Wazuh Dashboard](#step-9--install-and-configure-wazuh-dashboard)
- [Step 10 — Start everything in order](#step-10--start-everything-in-order)
- [Step 11 — Log in and change default passwords](#step-11--log-in-and-change-default-passwords)
- [Step 12 — Configure File Integrity Monitoring (FIM)](#step-12--configure-file-integrity-monitoring-fim)
- [Step 13 — Suricata integration (forward pointer)](#step-13--suricata-integration-forward-pointer)
- [Step 14 — Firewall](#step-14--firewall)
- [Step 15 — Final verification checklist](#step-15--final-verification-checklist)
- [Next step](#next-step)

---

## Step 1 — Install Docker

Both the Indexer and Dashboard build pipelines rely on Docker to guarantee a consistent build environment (matching exactly what Wazuh's own CI uses) — install it now, before either build:

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

sudo systemctl enable --now docker
sudo docker run hello-world
```
> <img width="478" height="247" alt="image" src="https://github.com/user-attachments/assets/f901b351-5a8d-4bf3-8d1a-f9c65a02f131" />

The last command confirms Docker itself works before relying on it for a much longer build later.

---

## Step 2 — Build Wazuh Manager from source

```bash
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y cmake gcc gcc-c++ make automake autoconf libtool \
    openssl openssl-devel policycoreutils-python-utils procps-ng git curl wget \
    libstdc++-static

cd ~
git clone --branch v4.14.6 --depth 1 https://github.com/wazuh/wazuh.git wazuh-manager-src
cd wazuh-manager-src/src

make deps TARGET=server
make -j$(nproc) TARGET=server
```

Run the installer from the top of the extracted source tree (not `src/`):
```bash
cd ~/wazuh-manager-src
sudo ./install.sh
```
This launches an interactive wizard:
1. Language: `en`.
> <img width="374" height="205" alt="image" src="https://github.com/user-attachments/assets/5bfb0fc0-164f-4788-8a84-3a395b32a724" />
2. Confirm you've read the notice → **Enter**.
3. **1. Install Wazuh manager** (not agent).
4. Accept the default installation prefix (`/var/ossec` in 4.x).
5. Decline email notifications setup for this lab.
6. Accept default install of remaining optional modules → **Enter** through prompts.
> <img width="570" height="593" alt="image" src="https://github.com/user-attachments/assets/70571e83-7756-40e4-abe0-f37c59e2b51a" />
7. Wait for compilation and installation to complete.

Confirm it built and installed correctly:
```bash
sudo /var/ossec/bin/wazuh-control status
```
> <img width="414" height="191" alt="image" src="https://github.com/user-attachments/assets/4b739b5d-97c1-40c4-9f42-310b7baddfa5" />

(Expect it to report processes as **not running** at this point — nothing has been started yet, this just confirms the binaries exist.)

---

## Step 3 — Generate certificates

Wazuh's certificate tool is a shell script wrapper around `openssl` — nothing to build here beyond what's already installed:

```bash
cd ~
curl -sO https://packages.wazuh.com/4.14/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.14/config.yml
```

Edit `config.yml` to point every node block at MON01 itself (single-node deployment):
```bash
vi config.yml
```
```yaml
nodes:
  indexer:
    - name: node-1
      ip: "192.168.10.40"
  server:
    - name: wazuh-manager
      ip: "192.168.10.40"
  dashboard:
    - name: dashboard
      ip: "192.168.10.40"
```

```bash
chmod +x wazuh-certs-tool.sh
sudo bash wazuh-certs-tool.sh -A
```

---

## Step 4 — Build the Wazuh Indexer package

This follows Wazuh's official two-stage packaging pipeline: a **build stage** (compiles the core via Gradle, producing a "minimal" package with no plugins) and an **assemble stage** (bundles plugins, configuration, and service files into the final package) — the same process Wazuh's own CI uses to produce the packages normally downloaded from `packages.wazuh.com`.

```bash
cd ~
git clone https://github.com/wazuh/wazuh-indexer.git
cd wazuh-indexer
git checkout v4.14.6
```

**Build stage** — the scripts live in `packaging_scripts/` at this tag (not `build-scripts/`, which only exists on newer branches). Read the real instructions shipped in the source tree itself rather than relying on the web documentation, which can lag behind a specific tag:
```bash
cat packaging_scripts/README.md
```

Build using `baptizer.sh` to generate the correct package name, fed into `build.sh`:
```bash
bash packaging_scripts/build.sh -a x64 -d rpm -n $(bash packaging_scripts/baptizer.sh -a x64 -d rpm -m)
```
This produces a "min" package (no plugins yet) under `artifacts/dist/`.

**Assemble stage** — bundles the security plugin, default configuration, and service files onto the min package. Confirmed syntax, following the same `-a {arch} -d {distribution} -r {revision}` pattern documented for TAR in this tag's `packaging_scripts/README.md`:
```bash
bash packaging_scripts/assemble.sh -a x64 -d rpm -r 1
```
This extracts the min RPM (via `rpm2cpio`/`cpio`), installs plugins into it using the `opensearch-plugin` CLI tool, applies production configuration, and rebuilds the final RPM from `wazuh-indexer/distribution/packages/src/rpm/wazuh-indexer.rpm.spec`.

Confirm the final package exists:
```bash
ls artifacts/dist/*.rpm
```
Expect a file named like `wazuh-indexer-4.14.6-0.x86_64.rpm` (not `wazuh-indexer-min-...`, which is the intermediate, plugin-less package from the build stage alone).

> **This document's exact commands reflect what was confirmed by reading `packaging_scripts/README.md` directly in the checked-out `v4.14.6` tag — always prefer that file over this document or Wazuh's web documentation if they disagree**, since both had already drifted from this tag's actual script layout and flag casing (`-r` not `-R`, `packaging_scripts/` not `build-scripts/`, no `ci.sh`) by the time this was written.

---

## Step 5 — Install and configure Wazuh Indexer

```bash
Install whatever `.rpm` the assemble stage actually produced — confirm the exact filename first rather than assuming it matches this document's example:
```bash
ls ~/wazuh-indexer/artifacts/dist/*.rpm
sudo rpm -ivh ~/wazuh-indexer/artifacts/dist/wazuh-indexer-4.14.6-0.x86_64.rpm
```
Adjust the filename in the install command to match whatever `ls` actually showed if it differs.
```

Copy the generated certificates into place:
```bash
sudo mkdir -p /etc/wazuh-indexer/certs
sudo cp ~/wazuh-certificates/node-1.pem /etc/wazuh-indexer/certs/indexer.pem
sudo cp ~/wazuh-certificates/node-1-key.pem /etc/wazuh-indexer/certs/indexer-key.pem
sudo cp ~/wazuh-certificates/admin.pem /etc/wazuh-indexer/certs/
sudo cp ~/wazuh-certificates/admin-key.pem /etc/wazuh-indexer/certs/
sudo cp ~/wazuh-certificates/root-ca.pem /etc/wazuh-indexer/certs/
sudo chmod 500 /etc/wazuh-indexer/certs
sudo chmod 400 /etc/wazuh-indexer/certs/*
sudo chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs
```

Confirm `opensearch.yml` reflects single-node discovery:
```bash
grep "discovery.type" /etc/wazuh-indexer/opensearch.yml
```
Expect `discovery.type: single-node`.

Set the JVM heap to roughly half of MON01's RAM allocated to the Indexer, leaving room for Zabbix, Manager, Dashboard, and the OS itself. Append with `tee` rather than editing manually — OpenSearch's JVM options parser is strict about format and rejects any leading whitespace on a `-X...` line:
```bash
printf '%s\n' '-Xms4g' '-Xmx4g' | sudo tee -a /etc/wazuh-indexer/jvm.options
cat -A /etc/wazuh-indexer/jvm.options | grep -A1 "Xms\|Xmx"
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-indexer
```
(Don't start it yet — [Step 10](#step-10--start-everything-in-order) starts every component in the correct order together.)

---

## Step 6 — Initialize Indexer security

This one-time step loads the initial admin credentials and internal users into the Indexer — it can only run once the Indexer is actually running:

```bash
sudo systemctl start wazuh-indexer

sudo /usr/share/wazuh-indexer/bin/indexer-security-init.sh
```

Verify:
```bash
curl -k -u admin:admin https://192.168.10.40:9200
```
Expect a JSON response describing the cluster (the default `admin:admin` credential is changed in [Step 11](#step-11--log-in-and-change-default-passwords), not before).

---

## Step 7 — Install and configure Filebeat

Filebeat ships Wazuh Manager's alerts to the Indexer — the piece that actually needs the TLS certificates in the 4.x architecture:

```bash
sudo tee /etc/yum.repos.d/elastic.repo << 'EOF'
[elastic-7.x]
name=Elastic repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

sudo dnf install -y filebeat-7.10.2
```

Download Wazuh's Filebeat template and module (module version `0.5`, matching the `4.14.6` compatibility matrix — earlier 4.x releases used `0.4`, which won't have current field mappings):
```bash
sudo curl -so /etc/filebeat/filebeat.yml https://packages.wazuh.com/4.14/tpl/wazuh/filebeat/filebeat.yml
sudo curl -so /etc/filebeat/wazuh-template.json https://raw.githubusercontent.com/wazuh/wazuh/v4.14.6/extensions/elasticsearch/7.x/wazuh-template.json
sudo chmod go+r /etc/filebeat/wazuh-template.json

sudo mkdir -p /usr/share/filebeat/module
sudo curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.5.tar.gz | sudo tar -xvz -C /usr/share/filebeat/module
```

Confirm `/etc/filebeat/filebeat.yml`'s `hosts` entry points at MON01 itself:
```bash
sudo vi /etc/filebeat/filebeat.yml
```
```yaml
output.elasticsearch:
  hosts: ["127.0.0.1:9200"]
```

Copy the certificates Filebeat needs:
```bash
sudo mkdir -p /etc/filebeat/certs
sudo cp ~/wazuh-certificates/root-ca.pem /etc/filebeat/certs/
sudo cp ~/wazuh-certificates/node-1.pem /etc/filebeat/certs/filebeat.pem
sudo cp ~/wazuh-certificates/node-1-key.pem /etc/filebeat/certs/filebeat-key.pem
```

Store the Indexer credentials in Filebeat's keystore rather than plaintext in the config:
```bash
sudo filebeat keystore create
echo admin | sudo filebeat keystore add username --stdin --force
echo admin | sudo filebeat keystore add password --stdin --force
```
(Change these two values once the real Indexer password is set in [Step 11](#step-11--log-in-and-change-default-passwords).)

```bash
sudo systemctl daemon-reload
sudo systemctl enable filebeat
```

---

## Step 8 — Build the Wazuh Dashboard package

Same spirit as the Indexer — this uses Wazuh's official parameterized Docker build for the Dashboard.

```bash
cd ~
git clone -b v4.14.6 https://github.com/wazuh/wazuh-dashboard.git
cd wazuh-dashboard/dev-tools/build-packages/
```

Check the Dockerfile's expected build arguments before running (these have shifted between releases, and the values below are current for `4.14.6` at the time of writing — confirm against the actual `.nvmrc` and `package.json` in the checked-out tree if anything fails):
```bash
cat ../../.nvmrc
```

Build the Docker image:
```bash
sudo docker build \
  --build-arg NODE_VERSION=$(cat ../../.nvmrc) \
  --build-arg WAZUH_DASHBOARDS_BRANCH=v4.14.6 \
  --build-arg WAZUH_DASHBOARDS_PLUGINS=v4.14.6 \
  --build-arg WAZUH_SECURITY_DASHBOARDS_PLUGIN_BRANCH=v4.14.6 \
  --build-arg OPENSEARCH_DASHBOARDS_VERSION=2.19.5 \
  -t wzd:v4.14.6 \
  -f wazuh-dashboard.Dockerfile .
```
This is the heaviest build in this document — a full Node.js/Webpack build of an OpenSearch Dashboards fork plus several Wazuh plugin repositories, all inside Docker. Expect it to take a substantial amount of time (well over an hour is plausible) and to use significant CPU/RAM during the build — this is normal, not a sign something is wrong, unless it stalls completely or the container gets OOM-killed (check `sudo docker stats` in a second terminal if you're unsure it's still progressing).

Once the image build completes, run a container from it and copy the finished package out:
```bash
sudo docker create --name wazuh-dashboard-package wzd:v4.14.6
mkdir -p ~/wazuh-dashboard-packages
sudo docker cp wazuh-dashboard-package:/home/node/packages/. ~/wazuh-dashboard-packages/
sudo docker rm wazuh-dashboard-package
```

Confirm the package exists:
```bash
ls ~/wazuh-dashboard-packages/*.rpm
```

---

## Step 9 — Install and configure Wazuh Dashboard

```bash
sudo rpm -ivh ~/wazuh-dashboard-packages/wazuh-dashboard-4.14.6-*.rpm
```

Copy certificates:
```bash
sudo mkdir -p /etc/wazuh-dashboard/certs
sudo cp ~/wazuh-certificates/node-1.pem /etc/wazuh-dashboard/certs/dashboard.pem
sudo cp ~/wazuh-certificates/node-1-key.pem /etc/wazuh-dashboard/certs/dashboard-key.pem
sudo cp ~/wazuh-certificates/root-ca.pem /etc/wazuh-dashboard/certs/
sudo chmod 500 /etc/wazuh-dashboard/certs
sudo chmod 400 /etc/wazuh-dashboard/certs/*
sudo chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
```

Confirm `/etc/wazuh-dashboard/opensearch_dashboards.yml` points at the local Indexer:
```bash
grep "opensearch.hosts" /etc/wazuh-dashboard/opensearch_dashboards.yml
```
Expect `https://192.168.10.40:9200` or `https://localhost:9200`.

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-dashboard
```

---

## Step 10 — Start everything in order

```bash
sudo systemctl start wazuh-indexer   # already started in Step 6, confirm it's still up
sudo systemctl start wazuh-manager
sudo systemctl start filebeat
sudo systemctl start wazuh-dashboard
```

Confirm Filebeat can actually reach the Indexer (a very common point of failure — TLS cert mismatches or wrong credentials surface here):
```bash
sudo filebeat test output
```
Expect `talk to server... OK`.

---

## Step 11 — Log in and change default passwords

1. Open a browser to:
```
https://192.168.10.40
```
Accept the self-signed certificate warning.
2. Log in with the default credentials: `admin` / `admin`.
3. Immediately change the Indexer's internal passwords:
```bash
sudo bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -a
```
Copy the `admin` password immediately into [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `MON01_10.40` group, entry "Wazuh Dashboard admin".
4. Update Filebeat's keystore with the new `admin` password:
```bash
echo 'New-Password-From-Script-Output' | sudo filebeat keystore add password --stdin --force
sudo systemctl restart filebeat
```
5. Log out and back into the Dashboard with the new password to confirm it took effect.

---

## Step 12 — Configure File Integrity Monitoring (FIM)

Wazuh Manager's FIM (via the `syscheck` module) watches specified directories for changes and reports them as alerts. Since MON01's own local filesystem changes aren't especially interesting to monitor (the real value comes once agents are deployed to WEB01, WINAPP01, and others in [`17`](./17-agent-deployment-all-vms.md)), this step configures FIM on the Manager's own local ruleset so it's ready to push out to agents later, and enables it for the Manager itself as a working local example.

```bash
sudo vi /var/ossec/etc/ossec.conf
```
```xml
<syscheck>
  <disabled>no</disabled>
  <frequency>43200</frequency>

  <directories check_all="yes" report_changes="yes">/etc</directories>
  <directories check_all="yes" report_changes="yes">/var/ossec/etc</directories>
  <directories check_all="yes" report_changes="yes">/opt/zabbix/etc</directories>

  <ignore>/etc/mtab</ignore>
  <ignore>/etc/hosts.deny</ignore>
  <ignore type="sregex">.log$|.swp$</ignore>
</syscheck>
```

```bash
sudo systemctl restart wazuh-manager
```

Trigger a test change:
```bash
sudo touch /etc/test-fim-file.txt
sudo rm /etc/test-fim-file.txt
```
Check **Wazuh Dashboard → Integrity Monitoring** for the change event after the next `syscheck` scan.

---

## Step 13 — Suricata integration (forward pointer)

[`08-web01-suricata-nids.md`](./08-web01-suricata-nids.md) noted that Wazuh could optionally correlate Suricata's network-layer alerts with Wazuh's own host-layer FIM alerts. Wazuh Manager already ships built-in decoders and rules for Suricata's EVE JSON format — but the actual data path requires the **Wazuh Agent to be installed on WEB01** first, configured with a `<localfile>` block reading `/var/log/suricata/eve.json` with `log_format json`. Since WEB01 doesn't have a Wazuh Agent yet, this integration is completed in [`17-agent-deployment-all-vms.md`](./17-agent-deployment-all-vms.md) rather than here.

---

## Step 14 — Firewall

```bash
sudo firewall-cmd --permanent --add-port=9200/tcp
sudo firewall-cmd --permanent --add-port=1514/tcp
sudo firewall-cmd --permanent --add-port=1515/tcp
sudo firewall-cmd --permanent --add-port=55000/tcp
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```
`9200` — Indexer API. `1514` — agent event data. `1515` — agent enrollment. `55000` — Wazuh API. `https` (443) — Dashboard web UI.

---

## Step 15 — Final verification checklist

1. **All four services running:**
```bash
sudo systemctl status wazuh-indexer wazuh-manager filebeat wazuh-dashboard
```

2. **Filebeat successfully shipping to the Indexer:**
```bash
sudo filebeat test output
```

3. **Dashboard reachable and logging in with the new password:**
```bash
curl -k -I https://192.168.10.40
```

4. **FIM is active and generates alerts** (repeat the test from [Step 12](#step-12--configure-file-integrity-monitoring-fim)).

5. **Indexer cluster health is green or yellow** (yellow is expected and fine for a single-node deployment):
```bash
curl -k -u admin:'Your-New-Admin-Password' https://192.168.10.40:9200/_cluster/health?pretty
```

6. **Both built packages match the expected version** (confirms the from-source build actually produced 4.14.6, not some other checked-out state):
```bash
rpm -q wazuh-indexer wazuh-dashboard wazuh-manager
```

7. **Credentials are stored, not memorized** — confirm the Wazuh `admin` password is saved in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `MON01_10.40` group.

If all seven checks pass, Wazuh 4.14.6 — built entirely from source — is ready on MON01, running alongside Zabbix from [`12`](./12-mon01-zabbix-server-configuration.md).

---

## Next step

Continue to [`14-mon01-prometheus-grafana-monitoring.md`](./14-mon01-prometheus-grafana-monitoring.md) to add Prometheus and Grafana to this same VM, as a separate metrics-monitoring ecosystem alongside Zabbix.
