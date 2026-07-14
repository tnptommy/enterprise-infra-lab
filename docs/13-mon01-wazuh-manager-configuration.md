# 13 — MON01: Wazuh Manager Configuration

This document adds **Wazuh 4.14.6** to MON01 alongside the Zabbix stack from [`12`](./12-mon01-zabbix-server-configuration.md) — Wazuh Indexer (an OpenSearch fork), Wazuh Manager (the analysis engine agents report to), Filebeat (ships Manager alerts to the Indexer), and Wazuh Dashboard (the web UI). Every component is **built from source**, using Wazuh's own officially documented packaging pipeline rather than downloading pre-built packages.

No new VM or network configuration is needed here — this continues directly on MON01 as already built in [`12`](./12-mon01-zabbix-server-configuration.md), including the mounted data disk from [`12`'s Step 5](./12-mon01-zabbix-server-configuration.md#step-5--partition-and-mount-the-data-disk) (`/mnt/data`) — confirm that's mounted (`df -h /mnt/data`) before starting, since [Step 5](#step-5--install-and-configure-wazuh-indexer) below points the Indexer's data path there directly. A disk-space consideration beyond that, confirmed by actually running this build rather than estimated in advance: the Indexer build itself (Gradle cache + source tree) used roughly **9 GB** on the OS disk regardless of where the Indexer's own data ends up living — and pushing MON01's 60 GB OS disk past 90% used during the build triggers a real, confusing failure later, unrelated to the data-disk question above. OpenSearch's disk watermark silently blocks creating its own security index, surfacing in [Step 6](#step-6--initialize-indexer-security) as an error that looks unrelated to disk space at all. [Step 5](#step-5--install-and-configure-wazuh-indexer) includes a build-artifact cleanup step for this; the Dashboard build in [Step 8](#step-8--build-the-wazuh-dashboard-package) will add its own significant Docker/Node scratch space on top. Confirm at least 20 GB free on the OS disk before starting, and don't skip the cleanup step once the Indexer package is built.

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
- [Step 15 — Logrotate](#step-15--logrotate)
- [Step 16 — Final verification checklist](#step-16--final-verification-checklist)
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
sudo ./install
```
This launches an interactive wizard:
1. Language: `en`.
2. Confirm you've read the notice → **Enter**.
3. **1. Install Wazuh manager** (not agent).
4. Accept the default installation prefix (`/var/ossec` in 4.x).
5. Decline email notifications setup for this lab.
6. Accept default install of remaining optional modules → **Enter** through prompts.
7. Wait for compilation and installation to complete.

Confirm it built and installed correctly:
```bash
sudo /var/ossec/bin/wazuh-control status
```
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

The assemble stage downloads plugins (like `alerting`) from Maven Central using `mvn` — install Maven now, before it's needed partway through:
```bash
sudo dnf install -y maven
```

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

> **Known issue in this tag's spec file:** `rpmbuild` may fail with `error: line 97: second %install`. This isn't a real second `%install` section — it's a comment (`# The %install section gets executed under a dash shell,`) that literally contains the substring `%install`, which some `rpmbuild` versions treat as a new section directive even inside a comment. **Patch the original template**, not just the copy `assemble.sh` extracts into `artifacts/tmp/` — otherwise the next run re-extracts the unfixed version and hits the same error again:
> ```bash
> sed -i 's/# The %install section/# The %%install section/' \
>   ~/wazuh-indexer/distribution/packages/src/rpm/wazuh-indexer.rpm.spec
> ```
> Matching on the comment's actual text (rather than a line number) means this fix applies correctly regardless of which copy of the file you're looking at.

> **If assemble.sh is re-run after a previous failed attempt**, it can fail with `plugin directory [...] already exists` — stale state left behind by the earlier partial run. Clean it up before retrying (safe to do now that the template itself is patched above, so re-extraction won't reintroduce the spec bug):
> ```bash
> rm -rf ~/wazuh-indexer/artifacts/tmp
> bash packaging_scripts/assemble.sh -a x64 -d rpm -r 1
> ```

Confirm the final package exists:
```bash
ls artifacts/dist/*.rpm
```
Expect a file named like `wazuh-indexer_4.14.6-0_x86_64_<commit-hash>.rpm` (note underscores, and a trailing git commit hash — this is `baptizer.sh`'s actual naming convention) — clearly larger and differently named than `wazuh-indexer-min_4.14.6-0_x86_64_<commit-hash>.rpm`, the intermediate, plugin-less package from the build stage alone (roughly 260 MB vs. over 800 MB once plugins are bundled in).

> **This document's exact commands reflect what was confirmed by reading `packaging_scripts/README.md` directly in the checked-out `v4.14.6` tag — always prefer that file over this document or Wazuh's web documentation if they disagree**, since both had already drifted from this tag's actual script layout and flag casing (`-r` not `-R`, `packaging_scripts/` not `build-scripts/`, no `ci.sh`) by the time this was written.

---

## Step 5 — Install and configure Wazuh Indexer

Install whatever `.rpm` the assemble stage actually produced — the filename includes a git commit hash that varies per checkout, so use a glob rather than assuming an exact name:
```bash
ls ~/wazuh-indexer/artifacts/dist/*.rpm
sudo rpm -ivh ~/wazuh-indexer/artifacts/dist/wazuh-indexer_4.14.6-*_x86_64_*.rpm
```
This glob matches the full package (e.g. `wazuh-indexer_4.14.6-0_x86_64_3eb218fe481.rpm`, roughly 800+ MB) but not the `-min` intermediate (roughly 260 MB) — confirm only one file matches before running `rpm -ivh` if in doubt:
```bash
ls ~/wazuh-indexer/artifacts/dist/wazuh-indexer_4.14.6-*_x86_64_*.rpm
```

**Clean up build artifacts now, before they cause a real problem later.** The Gradle build cache (`~/.gradle`, ~3.5 GB), the checked-out `wazuh-indexer` source tree (~3.4 GB), and the earlier `wazuh-manager-src` tree (~1.9 GB) together add up to roughly 9 GB — easily enough to push a 60 GB OS disk past 90% used. This matters more than it might seem: OpenSearch enforces a **disk watermark** that silently blocks creating new indices once usage crosses a threshold (90% by default), which surfaces later in [Step 6](#step-6--initialize-indexer-security) as a confusing `cluster create-index blocked (api)` error that has nothing to do with certificates or configuration — it's purely about running low on disk space at the exact moment the security index needs to be created. Avoid hitting that entirely by cleaning up now, once the `.rpm` above is safely built:
```bash
df -h /
```
If usage is climbing toward 90%, back up just the built package first (small, worth keeping in case you need to reinstall on another VM later), then remove the much larger source/cache directories — the Manager and Indexer are both already installed at this point and don't need their source trees anymore:
```bash
mkdir -p ~/wazuh-rpm-backup
cp ~/wazuh-indexer/artifacts/dist/*.rpm ~/wazuh-rpm-backup/

rm -rf ~/.gradle
rm -rf ~/wazuh-indexer
rm -rf ~/wazuh-manager-src

df -h /
```
Aim to get back under 80% used — comfortable headroom before [Step 8](#step-8--build-the-wazuh-dashboard-package)'s Docker-based Dashboard build, which will consume its own significant scratch space.

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

**Point the Indexer's data path at the data disk mounted in [`12`'s Step 5](./12-mon01-zabbix-server-configuration.md#step-5--partition-and-mount-the-data-disk) (`/mnt/data`), instead of the OS disk default.** OpenSearch indices grow continuously and are exactly the kind of thing that disk was set aside for — this avoids the disk-full crisis that hit the OS disk while actually building this VM (documented above in this same step):
```bash
sudo mkdir -p /mnt/data/wazuh-indexer
sudo chown -R wazuh-indexer:wazuh-indexer /mnt/data/wazuh-indexer
sudo sed -i 's|^path.data:.*|path.data: /mnt/data/wazuh-indexer|' /etc/wazuh-indexer/opensearch.yml
grep "^path.data" /etc/wazuh-indexer/opensearch.yml
```
Expect `path.data: /mnt/data/wazuh-indexer`.

Confirm `opensearch.yml` reflects single-node bootstrap — the package-generated config for this version doesn't use a `discovery.type: single-node` key (a common pattern in some OpenSearch setups); instead it bootstraps by listing the node itself in `cluster.initial_master_nodes`:
```bash
grep -A2 "cluster.initial_master_nodes" /etc/wazuh-indexer/opensearch.yml
```
Expect `node.name` (a few lines above) and the single entry under `cluster.initial_master_nodes` to match — both should read `"node-1"`, matching MON01 as the only node in this deployment.

**Rename `node.name` to match this lab's `HOSTNAME_lastTwoOctets` convention** (`MON01_10.40`, matching the VMware Library name used throughout this lab) rather than leaving the generic `node-1` default — `cluster.initial_master_nodes` must be updated to the exact same value, or the single-node cluster won't bootstrap correctly on the next restart:
```bash
sudo sed -i 's/node\.name: .*/node.name: "MON01_10.40"/' /etc/wazuh-indexer/opensearch.yml
sudo sed -i 's/- "node-1"/- "MON01_10.40"/' /etc/wazuh-indexer/opensearch.yml
grep -A3 "cluster.initial_master_nodes\|^node.name" /etc/wazuh-indexer/opensearch.yml
```
Using `sed 's/key: .*/key: value/'` here (replacing everything after the colon) rather than a literal find-and-replace on the old value specifically — the latter is not idempotent and silently produces something like `MON01_10.40_10.40` if the substitution ever runs twice against an already-updated file, a mistake made live while working through this exact change.

The certificate files themselves (`node-1.pem`/`node-1-key.pem`, copied in this same step) don't need renaming to match — the certificate's identity is tied to its Distinguished Name (`plugins.security.nodes_dn` in this same config), not to `node.name`, so this rename is purely cosmetic/organizational and doesn't require regenerating certificates.

**Add `compatibility.override_main_response_version: true`.** This isn't optional if Filebeat (a genuine Elastic Beats client, not an OpenSearch-native tool) will ever successfully ship data to this Indexer — without it, two separate incompatibilities surface between Filebeat 7.10.2 and OpenSearch 2.19.5 (the base this Indexer build is on):
1. Filebeat's non-OSS distribution check fails outright (`Filebeat requires the default distribution of Elasticsearch`) — addressed separately in [Step 7](#step-7--install-and-configure-filebeat) by using the OSS repository instead.
2. Even with OSS Filebeat, its Bulk API client still includes a legacy `_type` field in request metadata (a concept Elasticsearch deprecated in 7.x and OpenSearch removed entirely in 2.0+) — OpenSearch rejects these requests outright with `illegal_argument_exception: ... unknown parameter [_type]`, silently dropping every event Filebeat tries to ship, with no index ever appearing on the Indexer side despite Filebeat's own logs showing no obvious error unless checked closely.

This setting makes the Indexer report a version string that satisfies both compatibility checks:
```bash
echo "compatibility.override_main_response_version: true" | sudo tee -a /etc/wazuh-indexer/opensearch.yml
grep "override_main_response_version" /etc/wazuh-indexer/opensearch.yml
```

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

> **If this fails with `index_create_block_exception... blocked by: [FORBIDDEN/10/cluster create-index blocked (api)]`**, it's the disk watermark issue flagged in [Step 5](#step-5--install-and-configure-wazuh-indexer) — OpenSearch refuses to create the `.opendistro_security` index because disk usage crossed its safety threshold (default 90%) at some point, most likely during the build in [Step 4](#step-4--build-the-wazuh-indexer-package). Confirm and fix:
> ```bash
> df -h /
> ```
> If usage is still high, free up space (the cleanup commands in Step 5 are the first thing to check — confirm they were actually run). Once usage is comfortably under 90%, **the block does not always clear itself immediately** even though the underlying disk pressure is gone — retrying `indexer-security-init.sh` a couple of times a minute or so apart is often enough once there's real headroom again. If it's still stuck, force the block to clear explicitly:
> ```bash
> curl -k -u admin:admin -X PUT "https://127.0.0.1:9200/_cluster/settings" \
>   -H 'Content-Type: application/json' \
>   -d '{"transient": {"cluster.blocks.create_index": null}, "persistent": {"cluster.blocks.create_index": null}}'
> ```
> Then re-run `indexer-security-init.sh` again.

Verify:
```bash
curl -k -u admin:admin https://192.168.10.40:9200
```
Expect a JSON response describing the cluster (the default `admin:admin` credential is changed in [Step 11](#step-11--log-in-and-change-default-passwords), not before).

---

## Step 7 — Install and configure Filebeat

Filebeat ships Wazuh Manager's alerts to the Indexer — the piece that actually needs the TLS certificates in the 4.x architecture.

**Use the OSS repository, not the default Elastic repository.** Wazuh Indexer is an OpenSearch fork, not genuine Elasticsearch — the default (non-OSS) Filebeat build actively checks that whatever it's talking to is a "default distribution" Elasticsearch (via the `/_license` endpoint and response metadata), and rejects the connection otherwise with `Filebeat requires the default distribution of Elasticsearch`, even though the actual data connection (TLS, auth) works perfectly fine. This is Wazuh's own officially documented pairing, not a workaround — Filebeat OSS drops that check entirely:
```bash
sudo tee /etc/yum.repos.d/elastic-oss.repo << 'EOF'
[elastic-oss-7.x]
name=Elastic OSS repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/oss-7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

sudo dnf install -y filebeat-7.10.2
```
> The package is still named plain `filebeat` even in this OSS repo (not `filebeat-oss`) — what makes it the OSS build is which repository it came from, not the package name. If you already added the regular (non-OSS) `elastic-7.x` repo and installed from there, `dnf` may resolve to that version instead; if `filebeat test output` later fails with the distribution error described above, confirm with `dnf info filebeat` which repo it actually installed from, remove it, and reinstall explicitly from `elastic-oss-7.x`.

Download Wazuh's Filebeat template and module (module version `0.5`, matching the `4.14.6` compatibility matrix — earlier 4.x releases used `0.4`, which won't have current field mappings):
```bash
sudo curl -so /etc/filebeat/filebeat.yml https://packages.wazuh.com/4.14/tpl/wazuh/filebeat/filebeat.yml
sudo curl -so /etc/filebeat/wazuh-template.json https://raw.githubusercontent.com/wazuh/wazuh/v4.14.6/extensions/elasticsearch/7.x/wazuh-template.json
sudo chmod go+r /etc/filebeat/wazuh-template.json

sudo mkdir -p /usr/share/filebeat/module
sudo curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.5.tar.gz | sudo tar -xvz -C /usr/share/filebeat/module
```

Confirm `/etc/filebeat/filebeat.yml`'s `hosts` entry points at MON01 itself — **use the real IP (`192.168.10.40`), not `127.0.0.1`**, even though both reach the same machine: the certificate generated in [Step 3](#step-3--generate-certificates) is issued for `192.168.10.40` specifically (matching `config.yml`'s node IP), so TLS hostname verification fails against `127.0.0.1` with `x509: certificate is valid for 192.168.10.40, not 127.0.0.1`, even though the connection itself succeeds:
```bash
sudo vi /etc/filebeat/filebeat.yml
```
```yaml
output.elasticsearch:
  hosts: ["192.168.10.40:9200"]
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

This is by far the most involved build in this document. Unlike the Indexer's `build.sh`/`assemble.sh` pair, the Dashboard's `build-packages.sh` (found by reading `dev-tools/build-packages/README.md` directly in the checked-out tree — the web documentation's single Docker command doesn't match this tag's actual layout, the same kind of drift already seen with the Indexer) **doesn't build anything itself** — it only packages three separately-built inputs, each requiring its own Node.js/Yarn build first:

1. The Dashboard base (from the `wazuh-dashboard` repo itself)
2. `wazuh-dashboard-plugins` (a separate repo, itself containing several sub-plugins: `main`, `wazuh-core`, `wazuh-check-updates`)
3. `wazuh-security-dashboards-plugin` (another separate repo)

Every step below reflects what was actually confirmed working, including several non-obvious requirements discovered only by hitting the failures directly — each is explained so you understand *why*, not just *what*.

### Set up a non-root build user

OpenSearch Dashboards' own build tooling refuses to run as root (`OpenSearch Dashboards should not be run as root. Use --allow-root to continue.`) — passing `--allow-root` through `yarn`'s multiple layers of wrapped subcommands isn't reliable, so create a dedicated user instead and do every `yarn`/`nvm` step as that user:

```bash
useradd -m -s /bin/bash builder
mv ~/wazuh-dashboard /home/builder/
chown -R builder:builder /home/builder/wazuh-dashboard
su - builder
```

Everything from here through the end of Build 3 runs as `builder`, not root.

### Install NVM

Different sub-projects can require different Node versions, so use `nvm` (Node Version Manager) rather than a single system-wide Node install:
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm --version
```

### Build 1 — the Dashboard base

```bash
cd ~/wazuh-dashboard
nvm install $(cat .nvmrc)
nvm use $(cat .nvmrc)
yarn osd bootstrap
```
If `yarn osd bootstrap` fails partway through with an SSL/TLS error (`sslv3 alert bad record mac`), it's a transient network interruption, not a real problem — delete `node_modules` and retry:
```bash
rm -rf node_modules
yarn osd bootstrap
```

```bash
yarn build-platform --linux --skip-os-packages --release
```
This is the heaviest of the three builds — expect a substantial amount of time (an hour or more is plausible) and significant CPU/RAM usage, which is normal. The result lands in `target/`, not the repo root:
```bash
find ~/wazuh-dashboard/target -iname "*.tar.gz"
```

**`build-packages.sh` expects `--base` to be a `.zip` file *containing* this `.tar.gz`, not the `.tar.gz` itself** — this is a real, non-obvious requirement discovered by hitting `unzip: cannot find zipfile directory` otherwise. Wrap it:
```bash
cd ~/wazuh-dashboard/target
zip base-wrapped.zip opensearch-dashboards-2.19.5-linux-x64.tar.gz
```

### Build 2 — the security plugin

```bash
mkdir -p ~/wazuh-dashboard/plugins
cd ~/wazuh-dashboard/plugins
git clone --depth 1 -b v4.14.6 https://github.com/wazuh/wazuh-security-dashboards-plugin.git
cd wazuh-security-dashboards-plugin
nvm use $(cat .nvmrc 2>/dev/null || cat ../../.nvmrc)
yarn
yarn build
find . -maxdepth 2 -iname "*.zip"
```

**Same wrapping requirement as the base package** — `--security` also needs a `.zip` containing the plugin `.zip`, not the plugin zip supplied directly (supplying it directly makes `build-packages.sh` extract its *contents* into the working directory instead of installing it as a plugin, which surfaces confusingly as `bin/opensearch-dashboards-plugin install opensearch-dashboards` failing with "No valid url specified" — a symptom that looks unrelated to its actual cause). Wrap it:
```bash
mkdir -p ~/wazuh-dashboard/security-package
cp ~/wazuh-dashboard/plugins/wazuh-security-dashboards-plugin/build/security-dashboards-*.zip ~/wazuh-dashboard/security-package/
cd ~/wazuh-dashboard/security-package
zip -j security-wrapped.zip security-dashboards-*.zip
```

### Build 3 — the dashboard plugins (main, wazuh-core, wazuh-check-updates)

```bash
cd ~
git clone --depth 1 -b v4.14.6 https://github.com/wazuh/wazuh-dashboard-plugins.git
cd wazuh-dashboard-plugins
nvm install $(cat .nvmrc)
nvm use $(cat .nvmrc)
cp -r plugins/* ~/wazuh-dashboard/plugins/
```

**Set `OPENSEARCH_DASHBOARDS_VERSION` before building — this is required, not optional.** Each plugin's `package.json` build script reads this environment variable (`yarn plugin-helpers build --opensearch-dashboards-version=$OPENSEARCH_DASHBOARDS_VERSION`) and stamps it into the plugin's manifest. Skip this and the manifest ships with an **empty** version string, which later fails with `Plugin wazuh [4.14.6] is incompatible with OpenSearch Dashboards [2.19.5]` — a real error that looks like a version mismatch but is actually a missing build input:
```bash
export OPENSEARCH_DASHBOARDS_VERSION=2.19.5

cd ~/wazuh-dashboard/plugins/main
yarn
yarn build

cd ~/wazuh-dashboard/plugins/wazuh-core
yarn
yarn build

cd ~/wazuh-dashboard/plugins/wazuh-check-updates
yarn
yarn build
```

Confirm each plugin's version stamped correctly (the filename itself is a quick tell — `wazuh-2.19.5.zip` with the version present, not a truncated `wazuh-.zip`):
```bash
find ~/wazuh-dashboard/plugins/main/build ~/wazuh-dashboard/plugins/wazuh-core/build ~/wazuh-dashboard/plugins/wazuh-check-updates/build -iname "*.zip"
```

Combine all three into a single `app.zip` — `build-packages.sh` expects exactly one `--app` input, containing all plugin zips flattened together (no subdirectories):
```bash
mkdir -p ~/wazuh-dashboard/app-package
cp ~/wazuh-dashboard/plugins/main/build/wazuh-*.zip ~/wazuh-dashboard/app-package/
cp ~/wazuh-dashboard/plugins/wazuh-core/build/wazuhCore-*.zip ~/wazuh-dashboard/app-package/
cp ~/wazuh-dashboard/plugins/wazuh-check-updates/build/wazuhCheckUpdates-*.zip ~/wazuh-dashboard/app-package/

cd ~/wazuh-dashboard/app-package
zip -j app.zip *.zip
```

### Final packaging step

`build-packages.sh` shells out to `docker build`/`docker run`, which needs root (or a user in the `docker` group, not set up in this guide) — switch back to root for this step:
```bash
exit
```

Git refuses to operate on a repository owned by a different user by default — allow it explicitly before running anything that touches the checked-out tree as root:
```bash
git config --global --add safe.directory /home/builder/wazuh-dashboard
```

```bash
cd /home/builder/wazuh-dashboard/dev-tools/build-packages/
rm -rf tmp output

bash build-packages.sh \
    --app "file:///home/builder/wazuh-dashboard/app-package/app.zip" \
    --base "file:///home/builder/wazuh-dashboard/target/base-wrapped.zip" \
    --security "file:///home/builder/wazuh-dashboard/security-package/security-wrapped.zip" \
    --rpm --debug
```

This installs each plugin one at a time inside a Docker container, then builds the final RPM in a second container — expect several more minutes, and don't be alarmed by `docker build`/`docker run` output interleaved with plugin installation logs.

Confirm the package exists:
```bash
find /home/builder/wazuh-dashboard/dev-tools/build-packages/output -iname "*.rpm"
```

**Clean up now, before moving on.** This build is the single heaviest disk consumer in this entire document — the `wazuh-dashboard` and `wazuh-dashboard-plugins` source trees (`node_modules` included) run to roughly **8 GB**, and the two Docker images built along the way (`dashboard-base-builder`, `dashboard-rpm-builder`) add another **~3 GB** of layer cache. Left in place, this is what pushes a 60 GB OS disk uncomfortably high — worth clearing immediately while the RPM you actually need is already safely built:
```bash
mkdir -p /mnt/data/backups
cp /home/builder/wazuh-dashboard/dev-tools/build-packages/output/*.rpm /mnt/data/backups/

rm -rf /home/builder/wazuh-dashboard
rm -rf /home/builder/wazuh-dashboard-plugins

sudo docker system prune -a -f

df -h /
```
The RPM itself is backed up to the data disk mounted in [`12`'s Step 5](./12-mon01-zabbix-server-configuration.md#step-5--partition-and-mount-the-data-disk) first, so nothing is lost — only the multi-gigabyte build scaffolding (source, `node_modules`, Docker layers) that has no further use once the package exists is removed.

---

## Step 9 — Install and configure Wazuh Dashboard

Install the `.rpm` backed up to the data disk in the cleanup step above (the original location inside `~/wazuh-dashboard` no longer exists — that's exactly what was just removed):
```bash
find /mnt/data/backups -iname "wazuh-dashboard*.rpm"
sudo rpm -ivh <path-from-the-find-command-above>
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

Wazuh Manager's FIM (via the `syscheck` module) watches specified directories for changes and reports them as alerts. Since MON01's own local filesystem changes aren't especially interesting to monitor (the real value comes once agents are deployed to WEB01, WINAPP01, and others in [`17`](./17-agent-deployment-all-vms.md)), this step configures FIM on the Manager's own local ruleset so it's ready to push out to agents later, and enables it for the Manager itself as a working local example — but configured the way a real deployment would be, not just switched on with defaults.

**Three things separate a genuinely useful FIM setup from a bare "enable it" config:**
1. **`whodata`, not just `check_all`** — plain FIM tells you *that* a file changed; `whodata` (backed by the Linux Audit subsystem) tells you *who* changed it and *what process* did it. For anything you'd actually investigate after an alert, this difference matters enormously.
2. **Tiered monitoring, not one flat list** — real-time + whodata for directories where the delay of a scheduled scan matters (system binaries, configs); scheduled scanning for everything else, since real-time watches have real CPU/inotify overhead and aren't worth paying everywhere.
3. **A real ignore list** — the default noise from log rotation, package manager caches, and temp files will drown out genuine alerts if left unfiltered.

### Install and configure auditd (required for `whodata` on Linux)

```bash
sudo dnf install -y audit
sudo systemctl enable --now auditd
```

### Configure syscheck

```bash
sudo vi /var/ossec/etc/ossec.conf
```
```xml
<syscheck>
  <disabled>no</disabled>
  <frequency>43200</frequency>
  <scan_on_start>yes</scan_on_start>
  <alert_new_files>yes</alert_new_files>
  <auto_ignore frequency="10" timeframe="3600">no</auto_ignore>

  <!-- Tier 1: real-time + whodata — system binaries and configs where knowing who/what made
       the change, immediately, is worth the overhead -->
  <directories realtime="yes" whodata="yes" report_changes="yes">/etc</directories>
  <directories realtime="yes" whodata="yes" report_changes="yes">/var/ossec/etc</directories>
  <directories realtime="yes" whodata="yes" report_changes="yes">/opt/zabbix/etc</directories>
  <directories realtime="yes" whodata="yes">/usr/bin,/usr/sbin</directories>
  <directories realtime="yes" whodata="yes">/bin,/sbin</directories>

  <!-- Tier 2: scheduled scan only — the Wazuh default; worth watching, not worth
       real-time overhead -->
  <directories>/boot</directories>
  <directories check_all="yes">/root</directories>

  <!-- Files/directories to ignore — the Wazuh default list -->
  <ignore>/etc/mtab</ignore>
  <ignore>/etc/hosts.deny</ignore>
  <ignore>/etc/mail/statistics</ignore>
  <ignore>/etc/random-seed</ignore>
  <ignore>/etc/random.seed</ignore>
  <ignore>/etc/adjtime</ignore>
  <ignore>/etc/httpd/logs</ignore>
  <ignore>/etc/utmpx</ignore>
  <ignore>/etc/wtmpx</ignore>
  <ignore>/etc/cups/certs</ignore>
  <ignore>/etc/dumpdates</ignore>
  <ignore>/etc/svc/volatile</ignore>

  <!-- File types to ignore, plus this lab's own additions for backup/temp file patterns -->
  <ignore type="sregex">.log$|.swp$|.tmp$|~$</ignore>
  <ignore type="sregex">/var/lib/rpm</ignore>
  <ignore type="sregex">/var/cache/dnf</ignore>

  <!-- Check the file, but never compute/store the diff — this lab's own addition,
       for anything that might contain secrets -->
  <nodiff>/etc/ssl/private.key</nodiff>
  <nodiff>/etc/wazuh-indexer/certs</nodiff>
  <nodiff>/etc/filebeat/certs</nodiff>

  <skip_nfs>yes</skip_nfs>
  <skip_dev>yes</skip_dev>
  <skip_proc>yes</skip_proc>
  <skip_sys>yes</skip_sys>

  <!-- Nice value for Syscheck process -->
  <process_priority>10</process_priority>

  <!-- Maximum output throughput -->
  <max_eps>50</max_eps>

  <!-- Database synchronization settings -->
  <synchronization>
    <enabled>yes</enabled>
    <interval>5m</interval>
    <max_eps>10</max_eps>
  </synchronization>

  <!-- Cap the diff size so a large config file rewrite doesn't generate a massive alert body —
       this lab's own addition -->
  <diff>
    <disk_quota>
      <enabled>yes</enabled>
      <limit>100</limit>
    </disk_quota>
    <file_size>
      <enabled>yes</enabled>
      <limit>10</limit>
    </file_size>
  </diff>
</syscheck>
```

> This starts from Wazuh's own default `syscheck` block (`skip_nfs`/`skip_dev`/`skip_proc`/`skip_sys` to avoid scanning virtual filesystems, `synchronization` for keeping the Manager's FIM database consistent, `max_eps` rate limiting, `auto_ignore`/`alert_new_files`) — none of that is worth discarding just to add `whodata`. Only the `directories` blocks (split into tiers) and the `diff`/extra `nodiff`/`ignore` entries are this lab's own additions layered on top.

> **Extending this to Windows agents later:** once WINAPP01 gets a Wazuh Agent in [`17`](./17-agent-deployment-all-vms.md), the same `whodata`/`realtime` pattern applies to Windows directories too (`C:\Windows\System32`, etc.), and FIM there can additionally watch **registry keys** (`<windows_registry>` entries) — a Windows-specific capability with no Linux equivalent, worth configuring specifically in that later document rather than assuming this Linux-focused config transfers as-is.

Restart to apply:
```bash
sudo systemctl restart wazuh-manager
```

### Verify

Trigger changes across both tiers to confirm each behaves as configured:
```bash
# Tier 1 (real-time + whodata) — should generate an alert within seconds, not
# waiting for the next scheduled scan
sudo touch /etc/test-fim-file.txt
sudo rm /etc/test-fim-file.txt

# Tier 2 (scheduled only) — won't alert until the next scan cycle; confirmed
# separately below rather than waited on live
sudo touch /boot/test-fim-file.txt
sudo rm /boot/test-fim-file.txt
```

Check **Wazuh Dashboard → Integrity Monitoring** — the `/etc` change should appear almost immediately (real-time), and should show a **user and process**, not just a file path, confirming `whodata` actually captured attribution rather than falling back to plain FIM. The `/boot` change won't show up until the next `frequency` cycle (12 hours) or a forced scan:
```bash
sudo /var/ossec/bin/agent_control -R -u 000
```

Confirm the underlying rule fired correctly from the command line, rather than relying on the dashboard alone:
```bash
sudo grep "Integrity checksum changed" /var/ossec/logs/alerts/alerts.log | tail -5
```

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

## Step 15 — Logrotate

**Filebeat already handles its own log rotation** — `keepfiles: 7` in `/etc/filebeat/filebeat.yml`'s `logging.files` section (set by Wazuh's own template downloaded in [Step 7](#step-7--install-and-configure-filebeat)) caps it at 7 rotated files with no extra configuration needed here.

**Wazuh Indexer is Java-based (log4j2), which typically has its own size-based rolling policy built in** — confirm that's actually true for this install rather than assuming it:
```bash
grep -A3 "RollingFile\|SizeBasedTriggeringPolicy" /etc/wazuh-indexer/log4j2.properties
```
If that shows a rolling policy already configured (the expected outcome for a standard Wazuh Indexer package), nothing further is needed here. If it comes back empty, that's worth investigating before assuming logs are safe long-term — but for the package-installed Indexer this document uses, the built-in policy should already be present.

**Wazuh Manager and Wazuh Dashboard both need an external `logrotate` rule** — neither rotates its own logs by default:

```bash
sudo tee /etc/logrotate.d/wazuh-manager << 'EOF'
/var/ossec/logs/ossec.log
/var/ossec/logs/alerts/alerts.log
/var/ossec/logs/alerts/alerts.json {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 wazuh wazuh
    sharedscripts
    postrotate
        systemctl reload wazuh-manager >/dev/null 2>&1 || true
    endscript
}
EOF

sudo tee /etc/logrotate.d/wazuh-dashboard << 'EOF'
/var/log/wazuh-dashboard/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    sharedscripts
    postrotate
        systemctl reload wazuh-dashboard >/dev/null 2>&1 || true
    endscript
}
EOF
```

Verify both rules are syntactically valid without waiting for the next scheduled run:
```bash
sudo logrotate -d /etc/logrotate.d/wazuh-manager
sudo logrotate -d /etc/logrotate.d/wazuh-dashboard
```
(`-d` is a dry run — it prints what logrotate *would* do without actually rotating anything yet.) If `/var/log/wazuh-dashboard` doesn't actually exist on this install (paths can vary slightly by package build), confirm the real path first:
```bash
sudo find / -maxdepth 4 -iname "*wazuh-dashboard*" -path "*log*" 2>/dev/null
```
and adjust the rule above to match before relying on it.

---

## Step 16 — Final verification checklist

1. **All four services running, and enabled to start on boot** — `systemctl enable --now` (used throughout this document) handles both in one command, but worth confirming both conditions explicitly rather than just that the process happens to be running right now:
```bash
sudo systemctl status wazuh-indexer wazuh-manager filebeat wazuh-dashboard
sudo systemctl is-enabled wazuh-indexer wazuh-manager filebeat wazuh-dashboard
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
