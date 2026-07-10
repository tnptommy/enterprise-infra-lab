# 13 — MON01: Wazuh Manager Configuration

This document adds **Wazuh** to MON01 alongside the Zabbix stack from [`12`](./12-mon01-zabbix-server-configuration.md) — Wazuh Indexer (an OpenSearch fork), Wazuh Manager (the analysis engine agents report to), and Wazuh Dashboard (the web UI), **all built from source**. Zabbix and Wazuh solve different problems side by side rather than competing: Zabbix answers "is this service up and performing within normal bounds," Wazuh answers "did something suspicious happen on this host" (file integrity changes, log-based intrusion detection, vulnerability data).

No new VM, disk, or network configuration is needed here — this continues directly on MON01 as already built in [`12`](./12-mon01-zabbix-server-configuration.md).

| Component | Version used | Source |
|---|---|---|
| Wazuh Manager | 5.0.0-beta3 (pre-release, built from source) | https://github.com/wazuh/wazuh |
| Wazuh Indexer | 5.0.0-beta3 (pre-release, built from source) | https://github.com/wazuh/wazuh-indexer |
| Wazuh Dashboard | 5.0.0-beta3 (pre-release, built from source) | https://github.com/wazuh/wazuh-dashboard |

> **This is genuinely bleeding-edge software — read this before starting.** Wazuh 5.0.0 is currently only at **beta3** (released July 2, 2026), not a stable release, and its from-source build documentation is still evolving alongside the beta itself — **this isn't a hypothetical concern: the config file was renamed from `ossec.conf` to `wazuh-manager.conf`, and the control script from `wazuh-control` to `wazuh-manager-control`, between beta1 and beta3** ([source: official release notes](https://github.com/wazuh/wazuh/releases/tag/v5.0.0-beta3), PRs [#36999](https://github.com/wazuh/wazuh/pull/36999) and [#37059](https://github.com/wazuh/wazuh/pull/37059)). Expect more of this kind of change before GA. Two things make 5.0 worth the risk for a learning-focused lab like this one:
>
> 1. **Filebeat is removed entirely in 5.0** — earlier Wazuh versions required Filebeat as a separate component shipping Manager alerts to the Indexer over TLS. 5.0 replaces this with a native **`indexer-connector`** built into the Manager itself, talking to the Indexer directly, now authenticating as a dedicated `wazuh-server` user rather than `admin` ([PR #37061](https://github.com/wazuh/wazuh/pull/37061)), with default credentials auto-configured during install ([PR #37192](https://github.com/wazuh/wazuh/pull/37192)) — simplifying the credential-wiring steps below compared to earlier betas.
> 2. Building **all three components from source is realistic for the Manager** (C/Python, a well-documented `make`-based build) but **genuinely heavy for the Indexer and Dashboard** — both are large Gradle (Java) and Yarn/Webpack (Node.js) builds respectively, each capable of taking well over an hour and consuming significant RAM during compilation, distinct from their much lighter runtime footprint afterward. Budget real time for this document, expect at least one build failure to troubleshoot, and treat every command below as "best understanding at the time of writing" rather than guaranteed-correct for whatever beta build you actually download — cross-check anything that doesn't behave as expected against the current release notes at https://github.com/wazuh/wazuh/releases.
>
> If this turns out to be more troubleshooting than learning, [`12`'s alternative note](./12-mon01-zabbix-server-configuration.md) about falling back to a stable package-based install applies equally well here — substitute Wazuh 4.14.5 (the current stable release) and the official `dnf install wazuh-manager wazuh-indexer wazuh-dashboard` packages, which still require Filebeat as a fourth component.

---

## Table of contents

- [Step 1 — Install common build dependencies](#step-1--install-common-build-dependencies)
- [Step 2 — Build Wazuh Manager from source](#step-2--build-wazuh-manager-from-source)
- [Step 3 — Generate certificates](#step-3--generate-certificates)
- [Step 4 — Build Wazuh Indexer from source](#step-4--build-wazuh-indexer-from-source)
- [Step 5 — Initialize Indexer security](#step-5--initialize-indexer-security)
- [Step 6 — Configure the Manager's indexer-connector](#step-6--configure-the-managers-indexer-connector)
- [Step 7 — Build Wazuh Dashboard from source](#step-7--build-wazuh-dashboard-from-source)
- [Step 8 — Start everything in order](#step-8--start-everything-in-order)
- [Step 9 — Log in and change default passwords](#step-9--log-in-and-change-default-passwords)
- [Step 10 — Configure File Integrity Monitoring (FIM)](#step-10--configure-file-integrity-monitoring-fim)
- [Step 11 — Suricata integration (forward pointer)](#step-11--suricata-integration-forward-pointer)
- [Step 12 — Firewall](#step-12--firewall)
- [Step 13 — Final verification checklist](#step-13--final-verification-checklist)
- [Next step](#next-step)

---

## Step 1 — Install common build dependencies

```bash
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y cmake gcc gcc-c++ make automake autoconf libtool \
    openssl openssl-devel policycoreutils-python-utils procps-ng git curl wget \
    java-21-openjdk-devel libstdc++-static
```
> `openssl-devel` provides headers/libraries for *building* software against OpenSSL — it does **not** include the `openssl` command-line binary itself. [Step 3](#step-3--generate-certificates)'s `wazuh-certs-tool.sh` shells out to the `openssl` CLI directly, so the plain `openssl` package is included here too, alongside `libstdc++-static` (needed specifically for linking the Wazuh Engine component (`wazuh-engine`) during the Manager build in [Step 2](#step-2--build-wazuh-manager-from-source) — without it, `make` fails partway through with `cannot find -lstdc++`, since Rocky Linux only ships the shared library (`.so`) by default, not the static archive (`.a`) some of Wazuh's build targets link against).

`java-21-openjdk-devel` is for the Indexer/Dashboard builds in later steps ([Step 4](#step-4--build-wazuh-indexer-from-source), [Step 7](#step-7--build-wazuh-dashboard-from-source)) — OpenSearch 3.x (the base Wazuh Indexer 5.0 forks) requires a modern JDK to build. Confirm Rocky 10's system `gcc`/`cmake` versions are new enough before proceeding — the Wazuh source build documentation for older EL7/EL8 systems includes steps to compile GCC 9.4.0 and CMake 3.18.3 from source because those systems shipped compilers too old to build Wazuh; Rocky Linux 10 ships GCC 14.x and CMake well past the minimum, so that extra step isn't needed here:
```bash
gcc --version
cmake --version
```

Node.js and Yarn are needed specifically for the Dashboard build — install these now too. **Rocky Linux 10 dropped `dnf module` streams entirely** (unlike Rocky 9/8, which used `dnf module install nodejs:X`) — install via the NodeSource repository instead, which also pins an exact major version rather than leaving it to whatever AppStream's plain `nodejs` package happens to default to:

```bash
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
sudo dnf install -y nodejs
```
NodeSource's package bundles `npm` directly — no separate `npm` package to install, and no `sudo npm install -g yarn` needed for npm itself. Confirm the version:
```bash
node --version
npm --version
```

Install Yarn (used by the Dashboard's build tooling in [Step 7](#step-7--build-wazuh-dashboard-from-source)):
```bash
sudo npm install -g yarn
```

---

## Step 2 — Build Wazuh Manager from source

```bash
cd ~
git clone --branch v5.0.0-beta3 --depth 1 https://github.com/wazuh/wazuh.git wazuh-manager-src
cd wazuh-manager-src/src

make deps TARGET=server
make -j$(nproc) TARGET=server
```
`make deps` downloads a portable CPython build alongside the rest of the Manager's bundled dependencies — this can take a while on first run.

Run the installer from the top of the extracted source tree (not `src/`):
```bash
cd ~/wazuh-manager-src
sudo ./install
```
This launches an interactive wizard:
1. Language: `en`.
2. Confirm you've read the notice → **Enter**.
3. **1. Install Wazuh manager** (not agent).
4. Accept the default installation prefix (`/var/wazuh-manager` in 5.0, replacing the `/var/ossec` path used through 4.x) unless you have a reason to change it.
5. Decline email notifications setup for this lab (or configure it if you'd like — not required).
6. Accept default install of remaining optional modules (vulnerability detector, etc.) — **Enter** through prompts.
7. Wait for compilation and installation to complete.

Confirm the binary built and installed correctly:
```bash
sudo /var/wazuh-manager/bin/wazuh-manager-control status
```
(Expect it to report processes as **not running** at this point — nothing has been started yet, this just confirms the binaries exist and the control script works.)

---

## Step 3 — Generate certificates

Wazuh's certificate tool is a shell script wrapper around `openssl`, not a compiled binary — there's nothing to "build from source" here beyond what's already on the system:

```bash
cd ~
curl -sO https://packages.wazuh.com/4.14/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.14/config.yml
```
> Note this URL still references the `4.14` path — at the time of writing, Wazuh hadn't yet published a `5.0` equivalent of this specific helper script, and the certificate format/tooling for a single-node OpenSearch-based deployment hasn't changed between 4.x and 5.0-beta1. If a `5.0`-specific version exists by the time you're following this, prefer that one instead.

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

## Step 4 — Build Wazuh Indexer from source

This is the heaviest build in this document — budget real time for it.

```bash
cd ~
git clone --branch v5.0.0-beta3 --depth 1 https://github.com/wazuh/wazuh-indexer.git
cd wazuh-indexer

./gradlew :distribution:archives:linux-tar:assemble
```
This can take well over an hour on first run — Gradle downloads a large dependency tree and compiles the full OpenSearch-based codebase. If it fails partway through with an out-of-memory error, increase Gradle's heap before retrying:
```bash
export GRADLE_OPTS="-Xmx4g"
./gradlew :distribution:archives:linux-tar:assemble
```

Once complete, the built distribution tarball lands under `distribution/archives/linux-tar/build/distributions/`:
```bash
ls distribution/archives/linux-tar/build/distributions/
```

Extract it to a permanent location:
```bash
sudo mkdir -p /opt/wazuh-indexer
sudo tar -xzf distribution/archives/linux-tar/build/distributions/wazuh-indexer-*.tar.gz -C /opt/wazuh-indexer --strip-components=1

sudo groupadd --system wazuh-indexer
sudo useradd --system -g wazuh-indexer -d /opt/wazuh-indexer -s /sbin/nologin wazuh-indexer
sudo chown -R wazuh-indexer:wazuh-indexer /opt/wazuh-indexer
```

Copy certificates:
```bash
sudo mkdir -p /opt/wazuh-indexer/certs
sudo cp ~/wazuh-certificates/node-1.pem /opt/wazuh-indexer/certs/indexer.pem
sudo cp ~/wazuh-certificates/node-1-key.pem /opt/wazuh-indexer/certs/indexer-key.pem
sudo cp ~/wazuh-certificates/admin.pem /opt/wazuh-indexer/certs/
sudo cp ~/wazuh-certificates/admin-key.pem /opt/wazuh-indexer/certs/
sudo cp ~/wazuh-certificates/root-ca.pem /opt/wazuh-indexer/certs/
sudo chmod 500 /opt/wazuh-indexer/certs
sudo chmod 400 /opt/wazuh-indexer/certs/*
sudo chown -R wazuh-indexer:wazuh-indexer /opt/wazuh-indexer/certs
```

Configure `opensearch.yml`:
```bash
sudo vi /opt/wazuh-indexer/config/opensearch.yml
```
Set (or confirm) single-node discovery, and point the security plugin at the certs just copied:
```yaml
network.host: 192.168.10.40
discovery.type: single-node

plugins.security.ssl.transport.pemcert_filepath: certs/indexer.pem
plugins.security.ssl.transport.pemkey_filepath: certs/indexer-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: certs/root-ca.pem
plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: certs/indexer.pem
plugins.security.ssl.http.pemkey_filepath: certs/indexer-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: certs/root-ca.pem
```

Set the JVM heap (roughly half this VM's RAM allocated to the Indexer, leaving room for Zabbix, Manager, Dashboard, and the OS):
```bash
sudo vi /opt/wazuh-indexer/config/jvm.options
```
```
-Xms4g
-Xmx4g
```

Create a systemd unit (a source build has no packaged one):
```bash
sudo tee /etc/systemd/system/wazuh-indexer.service << 'EOF'
[Unit]
Description=Wazuh Indexer
After=network.target

[Service]
Type=simple
User=wazuh-indexer
Group=wazuh-indexer
LimitNOFILE=65535
LimitMEMLOCK=infinity
ExecStart=/opt/wazuh-indexer/bin/opensearch
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable wazuh-indexer
```

---

## Step 5 — Initialize Indexer security

```bash
sudo systemctl start wazuh-indexer
```
Wait 30–60 seconds for it to finish starting, then confirm it's listening:
```bash
sudo ss -tlnp | grep 9200
```

Run the security initialization script bundled in the built distribution:
```bash
sudo /opt/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
  -cd /opt/wazuh-indexer/opensearch-security/ \
  -icl -nhnv \
  -cacert /opt/wazuh-indexer/certs/root-ca.pem \
  -cert /opt/wazuh-indexer/certs/admin.pem \
  -key /opt/wazuh-indexer/certs/admin-key.pem
```

Verify:
```bash
curl -k -u admin:admin https://192.168.10.40:9200
```
Expect a JSON response describing the cluster.

---

## Step 6 — Configure the Manager's indexer-connector

This is the specific piece of 5.0 replacing Filebeat — the Manager talks to the Indexer natively instead of through a separate shipping agent. As of beta3, this connection authenticates as a dedicated **`wazuh-server`** user (not `admin`), with default credentials auto-configured during installation rather than requiring manual setup — confirm the block below matches what the installer actually generated before assuming you need to create it from scratch.

```bash
sudo vi /var/wazuh-manager/etc/wazuh-manager.conf
```
Confirm (or add) an `<indexer>` block pointing at the local Indexer:
```xml
<indexer>
  <enabled>yes</enabled>
  <hosts>
    <host>https://192.168.10.40:9200</host>
  </hosts>
  <username>wazuh-server</username>
  <ssl>
    <certificate_authorities>
      <ca>/opt/wazuh-indexer/certs/root-ca.pem</ca>
    </certificate_authorities>
    <certificate>/opt/wazuh-indexer/certs/admin.pem</certificate>
    <key>/opt/wazuh-indexer/certs/admin-key.pem</key>
  </ssl>
</indexer>
```

> **This exact block's tag names and structure are this document's best understanding of 5.0-beta3's `indexer-connector` configuration surface at the time of writing** — it changed once already between beta1 and beta3 (config file and control script both renamed, per this document's opening warning), and may change again before GA. Double-check this against the current beta's actual shipped documentation/changelog at https://github.com/wazuh/wazuh/releases if the Manager fails to connect and the log doesn't obviously explain why.

Create a systemd unit for the Manager (a source build has no packaged one):
```bash
sudo tee /etc/systemd/system/wazuh-manager.service << 'EOF'
[Unit]
Description=Wazuh Manager
After=network.target wazuh-indexer.service

[Service]
Type=forking
ExecStart=/var/wazuh-manager/bin/wazuh-manager-control start
ExecStop=/var/wazuh-manager/bin/wazuh-manager-control stop
PIDFile=/var/wazuh-manager/var/run/wazuh-manager-execd.pid
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable wazuh-manager
```

---

## Step 7 — Build Wazuh Dashboard from source

```bash
cd ~
git clone --branch v5.0.0-beta3 --depth 1 https://github.com/wazuh/wazuh-dashboard.git
cd wazuh-dashboard

yarn osd bootstrap
yarn build --linux --skip-os-packages
```
Like the Indexer, this is a large build (Node.js/Webpack) — expect it to take a substantial amount of time and RAM.

The build output lands under `target/`:
```bash
ls target/*.tar.gz
```

Extract it:
```bash
sudo mkdir -p /opt/wazuh-dashboard
sudo tar -xzf target/wazuh-dashboard-*.tar.gz -C /opt/wazuh-dashboard --strip-components=1

sudo groupadd --system wazuh-dashboard
sudo useradd --system -g wazuh-dashboard -d /opt/wazuh-dashboard -s /sbin/nologin wazuh-dashboard
sudo chown -R wazuh-dashboard:wazuh-dashboard /opt/wazuh-dashboard
```

Copy certificates:
```bash
sudo mkdir -p /opt/wazuh-dashboard/certs
sudo cp ~/wazuh-certificates/node-1.pem /opt/wazuh-dashboard/certs/dashboard.pem
sudo cp ~/wazuh-certificates/node-1-key.pem /opt/wazuh-dashboard/certs/dashboard-key.pem
sudo cp ~/wazuh-certificates/root-ca.pem /opt/wazuh-dashboard/certs/
sudo chmod 500 /opt/wazuh-dashboard/certs
sudo chmod 400 /opt/wazuh-dashboard/certs/*
sudo chown -R wazuh-dashboard:wazuh-dashboard /opt/wazuh-dashboard/certs
```

Configure `opensearch_dashboards.yml`:
```bash
sudo vi /opt/wazuh-dashboard/config/opensearch_dashboards.yml
```
```yaml
server.host: 0.0.0.0
server.ssl.enabled: true
server.ssl.certificate: /opt/wazuh-dashboard/certs/dashboard.pem
server.ssl.key: /opt/wazuh-dashboard/certs/dashboard-key.pem

opensearch.hosts: ["https://192.168.10.40:9200"]
opensearch.ssl.certificateAuthorities: ["/opt/wazuh-dashboard/certs/root-ca.pem"]
opensearch.username: "kibanaserver"
opensearch.password: "kibanaserver"
```

Create a systemd unit:
```bash
sudo tee /etc/systemd/system/wazuh-dashboard.service << 'EOF'
[Unit]
Description=Wazuh Dashboard
After=network.target wazuh-indexer.service

[Service]
Type=simple
User=wazuh-dashboard
Group=wazuh-dashboard
ExecStart=/opt/wazuh-dashboard/bin/opensearch-dashboards
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable wazuh-dashboard
```

---

## Step 8 — Start everything in order

```bash
sudo systemctl start wazuh-indexer   # already started in Step 5, confirm it's still up
sudo systemctl start wazuh-manager
sudo systemctl start wazuh-dashboard
```

Confirm the Manager successfully connected to the Indexer via the new `indexer-connector` — check the Manager's own log rather than a separate Filebeat log (which no longer exists in this architecture):
```bash
sudo tail -50 /var/wazuh-manager/logs/wazuh-manager.log | grep -i indexer
```
Look for a successful connection message; if you instead see repeated connection errors, revisit [Step 6](#step-6--configure-the-managers-indexer-connector)'s configuration block against the current beta's documentation.

---

## Step 9 — Log in and change default passwords

1. Open a browser to:
```
https://192.168.10.40
```
Accept the self-signed certificate warning.
2. Log in with the default credentials: `admin` / `admin`.
3. Change the Indexer's internal passwords using the security admin tool from the built distribution:
```bash
sudo /opt/wazuh-indexer/plugins/opensearch-security/tools/hash.sh -p 'Your-New-Strong-Password'
```
Use the resulting hash to update the `admin` user's password in `/opt/wazuh-indexer/opensearch-security/internal_users.yml`, then re-run the `securityadmin.sh` command from [Step 5](#step-5--initialize-indexer-security) to apply it.
4. Store the new `admin` password in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `MON01_10.40` group, entry "Wazuh Dashboard admin".
5. Log out and back into the Dashboard with the new password to confirm it took effect.

---

## Step 10 — Configure File Integrity Monitoring (FIM)

Wazuh Manager's FIM (via the `syscheck` module) watches specified directories for changes and reports them as alerts — this configuration surface is unchanged from 4.x in 5.0. Since MON01's own local filesystem changes aren't especially interesting to monitor (the real value comes once agents are deployed to WEB01, WINAPP01, and others in [`17`](./17-agent-deployment-all-vms.md)), this step configures FIM on the Manager's own local ruleset so it's ready to push out to agents later, and enables it for the Manager itself as a working local example.

```bash
sudo vi /var/wazuh-manager/etc/wazuh-manager.conf
```
Find the `<syscheck>` block and confirm/add directories relevant to this lab's own hardening priorities:
```xml
<syscheck>
  <disabled>no</disabled>
  <frequency>43200</frequency>

  <directories check_all="yes" report_changes="yes">/etc</directories>
  <directories check_all="yes" report_changes="yes">/var/wazuh-manager/etc</directories>
  <directories check_all="yes" report_changes="yes">/opt/zabbix/etc</directories>

  <ignore>/etc/mtab</ignore>
  <ignore>/etc/hosts.deny</ignore>
  <ignore type="sregex">.log$|.swp$</ignore>
</syscheck>
```
`report_changes="yes"` makes Wazuh keep a diff of what actually changed, not just that a file changed.

Restart the Manager to apply:
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

## Step 11 — Suricata integration (forward pointer)

[`08-web01-suricata-nids.md`](./08-web01-suricata-nids.md) noted that Wazuh could optionally correlate Suricata's network-layer alerts with Wazuh's own host-layer FIM alerts. Wazuh Manager already ships built-in decoders and rules for Suricata's EVE JSON format — but the actual data path requires the **Wazuh Agent to be installed on WEB01** first, configured with a `<localfile>` block reading `/var/log/suricata/eve.json` with `log_format json`, so it can forward those entries to this Manager.

Since WEB01 doesn't have a Wazuh Agent yet, this integration is completed in [`17-agent-deployment-all-vms.md`](./17-agent-deployment-all-vms.md) rather than here — nothing further to do on MON01 itself for this.

---

## Step 12 — Firewall

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

## Step 13 — Final verification checklist

1. **All three services running:**
```bash
sudo systemctl status wazuh-indexer wazuh-manager wazuh-dashboard
```

2. **Manager successfully connected to Indexer** (repeat the log check from [Step 8](#step-8--start-everything-in-order)).

3. **Dashboard reachable and logging in with the new password:**
```bash
curl -k -I https://192.168.10.40
```

4. **FIM is active and generates alerts** (repeat the test from [Step 10](#step-10--configure-file-integrity-monitoring-fim)).

5. **Indexer cluster health is green or yellow** (yellow is expected and fine for a single-node deployment):
```bash
curl -k -u admin:'Your-New-Admin-Password' https://192.168.10.40:9200/_cluster/health?pretty
```

6. **Credentials are stored, not memorized** — confirm the Wazuh `admin` password is saved in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `MON01_10.40` group.

If all six checks pass, Wazuh 5.0.0-beta3 is running on MON01, alongside Zabbix from [`12`](./12-mon01-zabbix-server-configuration.md).

---

## Next step

Continue to [`14-mon01-prometheus-grafana-monitoring.md`](./14-mon01-prometheus-grafana-monitoring.md) to add Prometheus and Grafana to this same VM, as a separate metrics-monitoring ecosystem alongside Zabbix.
