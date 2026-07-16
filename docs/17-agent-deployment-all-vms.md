# 17 — Agent Deployment to All VMs

This document is different in character from every build document before it — there's no new service being built from source here. Instead, it rolls out **Zabbix Agent 2**, **Wazuh Agent**, and **node_exporter**/**windows_exporter** to every VM built so far, and completes two integrations that earlier documents deliberately left as forward pointers:

1. [`08-web01-suricata-nids.md`](./08-web01-suricata-nids.md) and [`13`](./13-mon01-wazuh-manager-configuration.md#step-13--suricata-integration-forward-pointer) — WEB01's Suricata alerts reaching Wazuh Manager via a Wazuh Agent.
2. [`15`](./15-log01-elasticsearch-logstash-kibana.md#step-14--configure-logstash) — WEB01's Suricata alerts also reaching LOG01's Logstash pipeline via Filebeat.

**Zabbix Agent 2 is built from source** on both Linux and Windows VMs — same `8.0.0beta2` tarball as MON01's Zabbix Server in [`12`](./12-mon01-zabbix-server-configuration.md), keeping the agent-to-server protocol version matched across every host, consistent with this lab's general preference for building things from source where practical. The Windows build follows [Zabbix's own official documentation](https://www.zabbix.com/documentation/8.0/en/manual/installation/install/win_agent2) closely, since it's a genuinely different toolchain (MSYS2/MinGW/vcpkg) from the `./configure && make` pattern used everywhere else in this lab. Wazuh Agent and the Prometheus exporters, by contrast, are installed via official packages/binaries — Wazuh Agent's source build shares the same multi-repo complexity already worked through once for the Manager/Indexer in [`13`](./13-mon01-wazuh-manager-configuration.md), with little additional value in repeating that effort for a comparatively thin agent; the exporters are simple, self-contained static Go binaries with no meaningful "build" step to learn from.

## VMs covered

| VM | OS | Zabbix Agent 2 | Wazuh Agent | Exporter |
|---|---|---|---|---|
| WEB01 | Rocky Linux 10 | Agent 2 | Wazuh Agent | node_exporter |
| WINAPP01 | Windows Server 2025 | Agent (Windows) | Wazuh Agent (Windows) | windows_exporter |
| DC01 | Windows Server 2025 | Agent (Windows) | Wazuh Agent (Windows) | windows_exporter |
| CLIENT01 | Windows 11 | Agent (Windows) | Wazuh Agent (Windows) | windows_exporter |
| LOG01 | Rocky Linux 10 | Agent 2 | Wazuh Agent | node_exporter |
| LOG02 | Rocky Linux 10 | Agent 2 | Wazuh Agent | node_exporter |

MON01 already monitors itself (Zabbix Agent 2 from [`12`](./12-mon01-zabbix-server-configuration.md#step-19--add-mon01-as-the-first-monitored-host), node_exporter from [`14`](./14-mon01-prometheus-grafana-monitoring.md#step-5--install-node_exporter-on-mon01)) — no changes needed there. OPS01 doesn't exist yet ([`19`](./19-ops01-ansible-keycloak.md)) and is covered when it's built.

---

## Table of contents

- [Step 1 — Zabbix Agent 2 from source on Linux VMs](#step-1--zabbix-agent-2-from-source-on-linux-vms)
- [Step 2 — Zabbix Agent 2 from source on Windows VMs](#step-2--zabbix-agent-2-from-source-on-windows-vms)
- [Step 3 — Confirm Zabbix host groups populated correctly](#step-3--confirm-zabbix-host-groups-populated-correctly)
- [Step 4 — Wazuh Agent on Linux VMs](#step-4--wazuh-agent-on-linux-vms)
- [Step 5 — Wazuh Agent on Windows VMs](#step-5--wazuh-agent-on-windows-vms)
- [Step 6 — Complete the Suricata-to-Wazuh integration](#step-6--complete-the-suricata-to-wazuh-integration)
- [Step 7 — Complete the Suricata-to-Logstash integration](#step-7--complete-the-suricata-to-logstash-integration)
- [Step 8 — Winlogbeat on Windows VMs](#step-8--winlogbeat-on-windows-vms)
- [Step 9 — Filebeat on WINAPP01 for IIS logs](#step-9--filebeat-on-winapp01-for-iis-logs)
- [Step 10 — Metricbeat and Packetbeat on WEB01](#step-10--metricbeat-and-packetbeat-on-web01)
- [Step 11 — node_exporter on Linux VMs](#step-11--node_exporter-on-linux-vms)
- [Step 12 — windows_exporter on Windows VMs](#step-12--windows_exporter-on-windows-vms)
- [Step 13 — Update Prometheus scrape targets](#step-13--update-prometheus-scrape-targets)
- [Step 14 — Create Kibana Data Views](#step-14--create-kibana-data-views)
- [Step 15 — Final verification checklist](#step-15--final-verification-checklist)
- [Next step](#next-step)

---

## Step 1 — Zabbix Agent 2 from source on Linux VMs

Same source tarball as MON01's Zabbix Server build in [`12`](./12-mon01-zabbix-server-configuration.md#step-11--download-and-configure-zabbix-source) — matching version keeps the agent-to-server protocol in sync, and it's the same pre-release branch this lab has already committed to for Zabbix. Unlike MON01, these VMs only need the agent, not the full server — a lighter `configure` with `--enable-agent2` alone, skipping the server-specific database/SNMP flags entirely.

Run on **WEB01**, **LOG01**, and **LOG02**:

```bash
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y pkgconfig openssl openssl-devel pcre2-devel golang git libstdc++-static

sudo groupadd --system zabbix
sudo useradd --system -g zabbix -d /usr/lib/zabbix -s /sbin/nologin -c "Zabbix Monitoring System" zabbix
```
Same dependency list as MON01's build (minus MariaDB/SNMP-specific packages the agent doesn't need) — `libstdc++-static` is included here too, for the same reason documented in [`12`](./12-mon01-zabbix-server-configuration.md#step-9--install-zabbix-build-dependencies): a missing static libstdc++ causes a linking failure partway through the build that has nothing obviously to do with the actual error message.

```bash
cd ~
wget https://cdn.zabbix.com/zabbix/sources/development/8.0/zabbix-8.0.0beta2.tar.gz
tar -xzvf zabbix-8.0.0beta2.tar.gz
mv zabbix-8.0.0beta2 zabbix-8.0.0
cd zabbix-8.0.0

./configure \
  --prefix=/opt/zabbix \
  --enable-agent2 \
  --with-openssl \
  --with-libpcre2

make -j$(nproc)
sudo make install
```

Confirm the binary built:
```bash
/opt/zabbix/sbin/zabbix_agent2 --version
```

Configure it to point at MON01:
```bash
sudo vi /opt/zabbix/etc/zabbix_agent2.conf
```
```ini
Server=192.168.10.40
ServerActive=192.168.10.40
Hostname=<WEB01_10.21, LOG01_10.50, or LOG02_10.51 — whichever this VM actually is, matching this lab's `HOSTNAME_lastTwoOctets` convention used consistently across every monitoring tool (Zabbix, Wazuh, Prometheus, Elasticsearch, OpenSearch) rather than a bare hostname>
```

Create a systemd unit — a source build has no packaged one, same as every other from-source service in this lab:
```bash
sudo tee /etc/systemd/system/zabbix-agent2.service << 'EOF'
[Unit]
Description=Zabbix Agent 2
After=network.target

[Service]
Type=simple
User=zabbix
Group=zabbix
ExecStart=/opt/zabbix/sbin/zabbix_agent2 -c /opt/zabbix/etc/zabbix_agent2.conf --foreground
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now zabbix-agent2

sudo firewall-cmd --permanent --add-port=10050/tcp
sudo firewall-cmd --reload
```

Clean up the source tree once confirmed working, same disk-hygiene lesson as every other from-source build in this lab:
```bash
rm -rf ~/zabbix-8.0.0 ~/zabbix-8.0.0beta2.tar.gz
```

---

## Step 2 — Zabbix Agent 2 from source on Windows VMs

Same source, same reasoning as [Step 1](#step-1--zabbix-agent-2-from-source-on-linux-vms) — building on Windows is a genuinely different process from the Linux `./configure && make` pattern used everywhere else in this lab, since Windows has no equivalent toolchain by default. This follows Zabbix's own official documentation for [building Agent 2 on Windows](https://www.zabbix.com/documentation/8.0/en/manual/installation/install/win_agent2), using its **vcpkg** method — the simpler of the two official approaches, since it manages the OpenSSL/PCRE2 dependency builds automatically rather than requiring each to be manually compiled from source first.

Run on **WINAPP01**, **DC01**, and **CLIENT01**:

1. Install **Build Tools for Visual Studio 2022** (select the **Desktop development with C++** workload during installation — this includes the `vcpkg` package manager).
2. Install **Go** from [go.dev/dl](https://go.dev/dl/) (MSI installer), setting the install folder to `C:\Zabbix\Go`.
3. Download a MinGW distribution using the Microsoft Visual C runtime (e.g. `x86_64-15.1.0-release-win32-seh-msvcrt-rt_v12-rev0.7z` for 64-bit) from [the mingw-builds-binaries releases](https://github.com/niXman/mingw-builds-binaries/releases), and extract it to `C:\Zabbix\mingw64`.

Initialize `vcpkg` and install the C library dependencies (Command Prompt, run as Administrator):
```
cd C:\Zabbix
set PATH=%PATH%;"C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\vcpkg"

vcpkg new --application
vcpkg add port pcre2
vcpkg add port libiconv
vcpkg add port openssl

set PATH=C:\Zabbix\mingw64\bin;%PATH%
vcpkg install --triplet x64-mingw-static --x-install-root=x64
```

Download the same source tarball used for the Linux agent build in [Step 1](#step-1--zabbix-agent-2-from-source-on-linux-vms) — matching version across every agent in this lab, not just the Linux ones:
```
:: Extract zabbix-8.0.0beta2.tar.gz (from https://cdn.zabbix.com/zabbix/sources/development/8.0/zabbix-8.0.0beta2.tar.gz)
:: to C:\Zabbix\zabbix-8.0.0 — same renamed-directory convention as Step 1, for consistency
```

Navigate to `C:\Zabbix\zabbix-8.0.0\build\mingw` and create `build.bat`:
```
:: Add MinGW and Go to PATH for this session:
set PATH=C:\Zabbix\mingw64\bin;%PATH%
set PATH=C:\Zabbix\Go\bin;%PATH%

:: vcpkg installation path:
set vcpkg="C:\Zabbix\x64\x64-mingw-static"

:: Linker flag needed for the Crypt32 library:
SET CGO_LDFLAGS="-lCrypt32"

mingw32-make GOFLAGS="-buildvcs=false" ARCH=AMD64 PCRE2=%vcpkg% OPENSSL=%vcpkg% all
```
> **Don't wrap `%vcpkg%` in an extra pair of quotes** (`PCRE2="%vcpkg%"`) — the `vcpkg` variable's own `set` line already includes the quotes as part of its value, so referencing it as `%vcpkg%` alone already expands correctly to `"C:\Zabbix\x64\x64-mingw-static"`. Adding another layer produces a malformed doubled-quote value that breaks the build — confirmed by actually hitting this while working through the build live. This command must also be run **while standing in `build\mingw` specifically**, not the Zabbix source root — the Makefile's relative paths only resolve correctly from that exact directory.

Compile:
```
build.bat
```
The binary lands at `C:\Zabbix\zabbix-8.0.0\bin\win64\zabbix_agent2.exe`, with default config templates at `C:\Zabbix\zabbix-8.0.0\src\go\conf`.

Deploy to a dedicated runtime folder and configure it to point at MON01:
```
mkdir C:\Zabbix\agent2
copy C:\Zabbix\zabbix-8.0.0\bin\win64\zabbix_agent2.exe C:\Zabbix\agent2\
copy C:\Zabbix\zabbix-8.0.0\src\go\conf\zabbix_agent2.win.conf C:\Zabbix\agent2\
xcopy /E /I C:\Zabbix\zabbix-8.0.0\src\go\conf\zabbix_agent2.d C:\Zabbix\agent2\zabbix_agent2.d\
```

Edit `C:\Zabbix\agent2\zabbix_agent2.win.conf`:
```ini
Server=192.168.10.40
ServerActive=192.168.10.40
Hostname=<WINAPP01_10.15, DC01_10.10, or CLIENT01 — whichever this VM actually is; CLIENT01 uses DHCP with no fixed IP, so it stays as plain `CLIENT01` without the `_lastTwoOctets` suffix the other VMs use>
```

Register it as a Windows service (a from-source build has no installer to do this automatically, unlike the MSI package):
```powershell
C:\Zabbix\agent2\zabbix_agent2.exe --config C:\Zabbix\agent2\zabbix_agent2.win.conf --install
Start-Service "Zabbix Agent 2"
```

Open the firewall:
```powershell
New-NetFirewallRule -DisplayName "Zabbix Agent 2" -Direction Inbound -Protocol TCP -LocalPort 10050 -Action Allow
```

Confirm the service is running:
```powershell
Get-Service "Zabbix Agent 2"
```

---

## Step 3 — Confirm Zabbix host groups populated correctly

On **MON01**, in the Zabbix Frontend (`https://zabbix.corp-lab.com.vn:8443`):

1. **Data collection → Hosts → Create host** for each of the six VMs in the table above. **Host name** must exactly match each agent's own `Hostname=` config value — `WEB01_10.21`, `WINAPP01_10.15`, `DC01_10.10`, `LOG01_10.50`, `LOG02_10.51`, and `CLIENT01` (no suffix — CLIENT01 uses DHCP with no fixed IP, so the `_lastTwoOctets` convention doesn't cleanly apply there; confirm its actual current lease with `ipconfig /all` if in doubt). A mismatch here is the single most common reason an agent shows as unreachable despite the service itself running fine — Zabbix matches purely on this name string, not IP address.
2. **Host groups**: assign `Servers` for WEB01/WINAPP01/DC01/LOG01/LOG02, `Workstations` for CLIENT01 — matching the same grouping already established for WSUS client-side targeting in [`10`](./10-gpo-wsus-client-policy.md).
3. **Interfaces**: **Agent**, IP address matching each VM's static IP, port `10050`.
4. **Templates**: link **Linux by Zabbix agent** for the Linux VMs, **Windows by Zabbix agent** for the Windows VMs.
5. **Add** for each host.

Confirm all six show a green "ZBX" icon (not red) within a couple of minutes.

---

## Step 4 — Wazuh Agent on Linux VMs

Same source and build pattern as MON01's Wazuh Manager in [`13`](./13-mon01-wazuh-manager-configuration.md#step-2--build-wazuh-manager-from-source) — same repo, same version (`v4.14.6`), just `TARGET=agent` instead of `TARGET=server`.

Run on **WEB01**, **LOG01**, and **LOG02**:

```bash
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y cmake gcc gcc-c++ make automake autoconf libtool \
    openssl openssl-devel policycoreutils-python-utils procps-ng git curl wget \
    libstdc++-static

cd ~
git clone --branch v4.14.6 --depth 1 https://github.com/wazuh/wazuh.git wazuh-agent-src
cd wazuh-agent-src/src

make deps TARGET=agent
make -j$(nproc) TARGET=agent
```

Run the installer from the top of the extracted source tree (not `src/`):
```bash
cd ~/wazuh-agent-src
sudo ./install
```
This launches the same interactive wizard as the Manager build, with different choices this time:
1. Language: `en`.
2. Confirm you've read the notice → **Enter**.
3. **2. Install Wazuh agent** (not manager, this time).
4. Wazuh server IP/address: `192.168.10.40`.
5. Accept the default installation prefix (`/var/ossec` — same 4.x path convention as the Manager in [`13`](./13-mon01-wazuh-manager-configuration.md)).
6. Set the **agent name**: `<WEB01_10.21, LOG01_10.50, or LOG02_10.51 — whichever this VM actually is>`, matching this lab's `HOSTNAME_lastTwoOctets` convention used consistently across every monitoring tool.
7. Accept defaults for remaining prompts → **Enter** through them.
8. Wait for compilation and installation to complete.

Confirm it built and installed correctly:
```bash
sudo /var/ossec/bin/wazuh-control status
```
(Expect it to report as not running yet — this just confirms the binaries exist.)

```bash
sudo systemctl enable --now wazuh-agent
sudo firewall-cmd --permanent --add-port=1514/tcp
sudo firewall-cmd --reload
```

Clean up the source tree once confirmed working, same disk-hygiene lesson as every other from-source build in this lab:
```bash
rm -rf ~/wazuh-agent-src
```

---

## Step 5 — Wazuh Agent on Windows VMs

Run on **WINAPP01**, **DC01**, and **CLIENT01**:

1. Download the Windows agent MSI from `https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.6-1.msi`.
2. Install, setting `WAZUH_AGENT_NAME` explicitly — same reasoning as [Step 4](#step-4--wazuh-agent-on-linux-vms), otherwise Wazuh defaults to the bare OS hostname instead of this lab's `HOSTNAME_lastTwoOctets` convention:
```powershell
msiexec /i wazuh-agent-4.14.6-1.msi /q WAZUH_MANAGER="192.168.10.40" WAZUH_AGENT_NAME="WINAPP01_10.15"
```
(Substitute `DC01_10.10` or `CLIENT01` — no suffix, since it has no fixed IP — for the other two VMs.)
3. Start the service:
```powershell
NET START WazuhSvc
```
4. Open the firewall:
```powershell
New-NetFirewallRule -DisplayName "Wazuh Agent" -Direction Outbound -Protocol TCP -RemotePort 1514,1515 -Action Allow
```

---

## Step 6 — Complete the Suricata-to-Wazuh integration

This is the first forward pointer from [`08`](./08-web01-suricata-nids.md) and [`13`](./13-mon01-wazuh-manager-configuration.md#step-13--suricata-integration-forward-pointer) — now that WEB01 has a Wazuh Agent (from [Step 4](#step-4--wazuh-agent-on-linux-vms)), point it at Suricata's EVE JSON output:

On **WEB01**:
```bash
sudo vi /var/ossec/etc/ossec.conf
```
Add inside the `<ossec_config>` block:
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

```bash
sudo systemctl restart wazuh-agent
```

Trigger Suricata traffic (same test approach as [`08`'s own verification](./08-web01-suricata-nids.md)) and confirm an alert appears in **Wazuh Dashboard → Threat Hunting**, filtered to WEB01, showing a Suricata-decoded event rather than a generic unparsed log line — confirming Wazuh's built-in Suricata decoder actually matched the format, not just that data arrived.

---

## Step 7 — Complete the Suricata-to-Logstash integration

The second forward pointer, from [`15`](./15-log01-elasticsearch-logstash-kibana.md#step-14--configure-logstash) — WEB01 already has Filebeat installed for shipping Wazuh alerts (from earlier in this lab's history on WEB01, if applicable) or needs one now specifically for this. Point a Filebeat instance at LOG01's Logstash Beats input:

**Use Filebeat 9.4.3 here, not the 7.10.2 pinned for MON01's own Filebeat in [`13`](./13-mon01-wazuh-manager-configuration.md#step-7--install-and-configure-filebeat).** That version was pinned specifically to work around a Bulk API incompatibility between Filebeat and OpenSearch/Wazuh Indexer — a problem that doesn't apply here, since this Filebeat instance ships to Logstash → genuine Elasticsearch on LOG01, not OpenSearch. The "requires the default distribution" check that forced the OSS repository workaround for MON01 also doesn't trigger here, since Elasticsearch **is** the default distribution — a plain, current Filebeat works normally against it. This matches the same reasoning already applied to Winlogbeat in [Step 8](#step-8--winlogbeat-on-windows-vms).

On **WEB01**, if Filebeat isn't already installed for another purpose:
```bash
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
sudo tee /etc/yum.repos.d/elastic.repo << 'EOF'
[elastic-9.x]
name=Elastic repository for 9.x packages
baseurl=https://artifacts.elastic.co/packages/9.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

sudo dnf install -y filebeat-9.4.3
```

Configure a dedicated input/output specifically for Suricata → Logstash, separate from any Wazuh-alert-shipping Filebeat config that might already exist on this host:
```bash
sudo tee /etc/filebeat/filebeat.yml << 'EOF'
filebeat.inputs:
  - type: filestream
    id: suricata-eve
    paths:
      - /var/log/suricata/eve.json
    parsers:
      - ndjson:
          keys_under_root: true
          add_error_key: true
    fields:
      log_type: suricata
    fields_under_root: true

output.logstash:
  hosts: ["192.168.10.50:5044"]
EOF

sudo systemctl enable --now filebeat
```

Verify data is actually flowing — check on **LOG01**:
```bash
curl -k -u elastic:'Password-From-13' "https://192.168.10.50:9200/suricata-*/_count?pretty"
```
Expect a non-zero count once Suricata has logged any traffic.

---

## Step 8 — Winlogbeat on Windows VMs

Windows Event Log data (from WINAPP01, DC01, and CLIENT01) ships to the same Logstash pipeline on LOG01 as WEB01's Suricata data in [Step 7](#step-7--complete-the-suricata-to-logstash-integration) — same port, same Beats input, routed to a separate `winlogbeat-*` index by the `tags`-based conditional already built into that pipeline in [`15`'s Step 14](./15-log01-elasticsearch-logstash-kibana.md#step-14--configure-logstash).

**Use Winlogbeat 9.4.3** — same version as WEB01's Filebeat in [Step 7](#step-7--complete-the-suricata-to-logstash-integration), both matched to LOG01's actual Elasticsearch/Logstash version per Elastic's own compatibility guidance. Only MON01's own Filebeat in [`13`](./13-mon01-wazuh-manager-configuration.md#step-7--install-and-configure-filebeat) stays pinned to `7.10.2` — that one specifically talks to OpenSearch/Wazuh Indexer, which rejects newer Beats' Bulk API format, a problem unique to that one integration.

Run on **WINAPP01**, **DC01**, and **CLIENT01**:

1. Download Winlogbeat 9.4.3 from `https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-9.4.3-windows-x86_64.zip`.
2. Extract to `C:\Program Files\Winlogbeat`.
3. Configure:
```powershell
notepad "C:\Program Files\Winlogbeat\winlogbeat.yml"
```
```yaml
winlogbeat.event_logs:
  - name: Application
    fields:
      log_type: winlogbeat
    fields_under_root: true
  - name: System
    fields:
      log_type: winlogbeat
    fields_under_root: true
  - name: Security
    fields:
      log_type: winlogbeat
    fields_under_root: true

output.logstash:
  hosts: ["192.168.10.50:5044"]
```
4. Install as a service and start:
```powershell
cd "C:\Program Files\Winlogbeat"
.\install-service-winlogbeat.ps1
Start-Service winlogbeat
```
5. Open the firewall for outbound traffic to LOG01 (most Windows firewall profiles allow outbound by default — this rule makes the intent explicit rather than relying on that default):
```powershell
New-NetFirewallRule -DisplayName "Winlogbeat to LOG01" -Direction Outbound -Protocol TCP -RemotePort 5044 -RemoteAddress 192.168.10.50 -Action Allow
```

Verify data is flowing — check on **LOG01**:
```bash
curl -k -u elastic:'Password-From-13' "https://192.168.10.50:9200/winlogbeat-*/_count?pretty"
```
Expect a non-zero count, growing over time as Windows Event Log entries accumulate on each of the three VMs.

---

## Step 9 — Filebeat on WINAPP01 for IIS logs

**WINAPP01 only** — DC01 and CLIENT01 don't run IIS ([`09`](./09-winapp01-iis-sql-wsus.md) installed it exclusively on WINAPP01), so this step doesn't apply to them. Winlogbeat in [Step 8](#step-8--winlogbeat-on-windows-vms) covers Windows Event Log on all three Windows VMs, but IIS's access log is a separate plain-text file on disk, not part of Event Log — a genuinely different data source needing its own Beat.

Same version reasoning as [Step 7](#step-7--complete-the-suricata-to-logstash-integration) — Filebeat `9.4.3`, matching LOG01's actual Elasticsearch/Logstash version, no OSS-repo workaround needed since the destination is genuine Elasticsearch, not OpenSearch.

On **WINAPP01**:
1. Download Filebeat 9.4.3 for Windows from `https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-9.4.3-windows-x86_64.zip`.
2. Extract to `C:\Program Files\Filebeat`.
3. Configure:
```powershell
notepad "C:\Program Files\Filebeat\filebeat.yml"
```
```yaml
filebeat.inputs:
  - type: filestream
    id: iis-w3svc1
    paths:
      - C:\inetpub\logs\LogFiles\W3SVC1\*.log
    fields:
      log_type: iis
    fields_under_root: true

output.logstash:
  hosts: ["192.168.10.50:5044"]
```
IIS's default log location (`C:\inetpub\logs\LogFiles\W3SVC1\`) — confirm this matches the actual path if IIS logging was ever reconfigured to somewhere else since [`09`](./09-winapp01-iis-sql-wsus.md), since that document didn't customize it from the IIS default.

4. Install as a service and start:
```powershell
cd "C:\Program Files\Filebeat"
.\install-service-filebeat.ps1
Start-Service filebeat
```
5. Open the firewall:
```powershell
New-NetFirewallRule -DisplayName "Filebeat to LOG01" -Direction Outbound -Protocol TCP -RemotePort 5044 -RemoteAddress 192.168.10.50 -Action Allow
```

Verify data is flowing — check on **LOG01**:
```bash
curl -k -u elastic:'Password-From-13' "https://192.168.10.50:9200/iis-*/_count?pretty"
```
Expect a non-zero count once IIS has served any request — browsing to `http://web01.corp-lab.com.vn` or any WINAPP01-hosted site generates a log line immediately if none exists yet.

Add a matching **Kibana Data View** in [Step 14](#step-14--create-kibana-data-views) (`iis-*`, same as `suricata-*` and `winlogbeat-*`) once this is confirmed flowing.

---

## Step 10 — Metricbeat and Packetbeat on WEB01

Two more Beats, purely to practice getting different data types into Elasticsearch — same `9.4.3` version and same Logstash pipeline on LOG01 as everything else in this document, just two new source types. **Metricbeat** collects OS/service-level metrics (CPU, memory, disk); **Packetbeat** captures network traffic at the protocol level (HTTP requests, MySQL queries) — a genuinely different vantage point from Suricata's signature-based intrusion detection in [`08`](./08-web01-suricata-nids.md), since Packetbeat shows *what actually happened* at the protocol layer rather than *whether something matched a known-bad pattern*.

> **Overlap with Zabbix/Prometheus is expected and fine here** — Metricbeat's system metrics genuinely duplicate what Zabbix Agent 2 ([Step 1](#step-1--zabbix-agent-2-from-source-on-linux-vms)) and node_exporter ([Step 11](#step-11--node_exporter-on-linux-vms)) already collect. The point of adding it isn't to replace either — it's to practice shipping another data type through this same Beats → Logstash → Elasticsearch pipeline, which is a different skill from configuring Zabbix or Prometheus itself.

Run on **WEB01**:

```bash
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
sudo tee /etc/yum.repos.d/elastic.repo << 'EOF'
[elastic-9.x]
name=Elastic repository for 9.x packages
baseurl=https://artifacts.elastic.co/packages/9.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

sudo dnf install -y metricbeat-9.4.3 packetbeat-9.4.3
```

Configure Metricbeat — the `system` module (CPU/memory/disk/network) is enabled by default, just needs pointing at Logstash instead of its default direct-to-Elasticsearch output:
```bash
sudo tee /etc/metricbeat/metricbeat.yml << 'EOF'
metricbeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

fields:
  log_type: metricbeat
fields_under_root: true

output.logstash:
  hosts: ["192.168.10.50:5044"]
EOF

sudo metricbeat modules enable system
sudo systemctl enable --now metricbeat
```

Configure Packetbeat to watch WEB01's actual traffic — HTTP (Apache, from [`07`](./07-web01-lamp-nginx-loadbalancer.md)) and MySQL (MariaDB, same document) on their standard ports:
```bash
sudo tee /etc/packetbeat/packetbeat.yml << 'EOF'
packetbeat.interfaces.device: any

packetbeat.protocols:
  - type: http
    ports: [80, 8080, 8000, 5000, 8002]
  - type: mysql
    ports: [3306]

fields:
  log_type: packetbeat
fields_under_root: true

output.logstash:
  hosts: ["192.168.10.50:5044"]
EOF

sudo systemctl enable --now packetbeat
```

Verify data is flowing — check on **LOG01**:
```bash
curl -k -u elastic:'Password-From-13' "https://192.168.10.50:9200/metricbeat-*/_count?pretty"
curl -k -u elastic:'Password-From-13' "https://192.168.10.50:9200/packetbeat-*/_count?pretty"
```
Expect non-zero counts for both — Metricbeat generates data continuously on its own polling interval; Packetbeat needs actual traffic to WEB01 (an HTTP request or MySQL query) to have something to capture.

Add matching **Kibana Data Views** in [Step 14](#step-14--create-kibana-data-views) (`metricbeat-*`, `packetbeat-*`) once confirmed flowing.

---

## Step 11 — node_exporter on Linux VMs

Run on **WEB01**, **LOG01**, and **LOG02** — same binary, same systemd unit pattern as [`14`'s Step 5](./14-mon01-prometheus-grafana-monitoring.md#step-5--install-node_exporter-on-mon01):

```bash
cd ~
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
tar -xzf node_exporter-1.9.1.linux-amd64.tar.gz
sudo cp node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter-1.9.1.linux-amd64*

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

Open the firewall — **only to MON01's IP specifically**, not the whole network, since this exposes system metrics:
```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.40" port port="9100" protocol="tcp" accept'
sudo firewall-cmd --reload
```

---

## Step 12 — windows_exporter on Windows VMs

Run on **WINAPP01**, **DC01**, and **CLIENT01**:

1. Download the latest `windows_exporter` MSI from `https://github.com/prometheus-community/windows_exporter/releases/latest`.
2. Install:
```powershell
msiexec /i windows_exporter-amd64.msi ENABLED_COLLECTORS="cpu,cs,logical_disk,net,os,service,system,memory"
```
3. Confirm the service:
```powershell
Get-Service windows_exporter
```
4. Open the firewall, restricted to MON01's IP:
```powershell
New-NetFirewallRule -DisplayName "windows_exporter (MON01 only)" -Direction Inbound -Protocol TCP -LocalPort 9182 -RemoteAddress 192.168.10.40 -Action Allow
```
(`windows_exporter`'s default port is `9182`, not `9100` — different from the Linux `node_exporter` default, worth double-checking when writing the Prometheus scrape config in the next step so it isn't copy-pasted incorrectly.)

---

## Step 13 — Update Prometheus scrape targets

On **MON01**:
```bash
sudo vi /etc/prometheus/prometheus.yml
```
Extend the `node_exporter` job (created in [`14`'s Step 3](./14-mon01-prometheus-grafana-monitoring.md#step-3--configure-prometheus) with only MON01 itself as a target) to include every VM from [Step 11](#step-11--node_exporter-on-linux-vms), and add a new job for the Windows VMs from [Step 12](#step-12--windows_exporter-on-windows-vms):
```yaml
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance: 'mon01_10.40'
      - targets: ['192.168.10.21:9100']
        labels:
          instance: 'web01_10.21'
      - targets: ['192.168.10.50:9100']
        labels:
          instance: 'log01_10.50'
      - targets: ['192.168.10.51:9100']
        labels:
          instance: 'log02_10.51'

  - job_name: 'windows_exporter'
    static_configs:
      - targets: ['192.168.10.15:9182']
        labels:
          instance: 'winapp01_10.15'
      - targets: ['192.168.10.10:9182']
        labels:
          instance: 'dc01_10.10'
      - targets: ['192.168.10.100:9182']
        labels:
          instance: 'client01'
```
Every `instance` label follows this lab's `HOSTNAME_lastTwoOctets` convention (matching the VMware Library names used throughout, and the same rename already applied to MON01's own `instance` label in [`14`'s Step 3](./14-mon01-prometheus-grafana-monitoring.md#step-3--configure-prometheus)) — except **CLIENT01**, which stays as a plain hostname with no suffix, since it has no fixed IP to derive one from.

> CLIENT01's IP above (`192.168.10.100`) is the first address in the DHCP pool from [`02`](./02-network-architecture-planning.md) — CLIENT01 doesn't have a static IP, so confirm its actual current lease (`ipconfig /all` on CLIENT01) before trusting this value, since DHCP can hand out a different address within the pool.

```bash
sudo systemctl restart prometheus
```

Verify every target shows `"health": "up"`:
```bash
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -A2 '"health"'
```

---

## Step 14 — Create Kibana Data Views

Data flowing into Elasticsearch (confirmed in [Step 7](#step-7--complete-the-suricata-to-logstash-integration), [Step 8](#step-8--winlogbeat-on-windows-vms), [Step 9](#step-9--filebeat-on-winapp01-for-iis-logs), and [Step 10](#step-10--metricbeat-and-packetbeat-on-web01)) isn't automatically visible in Kibana — a **Data View** (Kibana's current term for what was called an "index pattern" in older versions) has to be created first, telling Kibana which indices to query and which field holds the timestamp.

On **LOG01**'s Kibana (`https://kibana.corp-lab.com.vn:5601` — or LOG01's IP directly if [`06`'s CNAME](./06-dc01-active-directory-dns-dhcp.md#step-5--configure-dns) hasn't been created yet):

1. ☰ menu → **Stack Management → Data Views → Create data view**.
2. **Name**: `Suricata`. **Index pattern**: `suricata-*`. **Timestamp field**: `@timestamp`. **Save data view to Kibana**.
3. Repeat: **Name**: `Winlogbeat`. **Index pattern**: `winlogbeat-*`. **Timestamp field**: `@timestamp`.
4. Repeat: **Name**: `IIS`. **Index pattern**: `iis-*`. **Timestamp field**: `@timestamp`.
5. Repeat: **Name**: `Metricbeat`. **Index pattern**: `metricbeat-*`. **Timestamp field**: `@timestamp`.
6. Repeat: **Name**: `Packetbeat`. **Index pattern**: `packetbeat-*`. **Timestamp field**: `@timestamp`.

Verify all five are actually browsable:

7. ☰ menu → **Discover** → select the `Suricata` data view from the dropdown (top left) — expect to see WEB01's Suricata events, most recent first.
8. Switch the dropdown to `Winlogbeat` — expect to see Windows Event Log entries from WINAPP01, DC01, and CLIENT01.
9. Switch the dropdown to `IIS` — expect to see WINAPP01's IIS access log entries.
10. Switch the dropdown to `Metricbeat` — expect to see WEB01's system metrics.
11. Switch the dropdown to `Packetbeat` — expect to see captured HTTP/MySQL transactions from WEB01.

> If the Data View creation screen shows no matching indices for any pattern, that means no index exists yet at the Elasticsearch level — this is a data-pipeline problem (Filebeat/Winlogbeat/Metricbeat/Packetbeat → Logstash → Elasticsearch), not a Kibana problem. Confirm directly against Elasticsearch before troubleshooting Kibana further:
> ```bash
> curl -k -u elastic:'Password-From-13' "https://192.168.10.50:9200/_cat/indices?v" | grep -E "suricata|winlogbeat|iis|metricbeat|packetbeat"
> ```

---

## Step 15 — Final verification checklist

1. **Every VM shows a healthy Zabbix Agent 2** (repeat the check from [Step 3](#step-3--confirm-zabbix-host-groups-populated-correctly)).

2. **Every VM shows as connected in Wazuh**:
```bash
# On MON01
sudo /var/ossec/bin/agent_control -l
```
Expect all six agents listed as `Active`.

3. **Suricata-to-Wazuh integration produces decoded alerts** (repeat the check from [Step 6](#step-6--complete-the-suricata-to-wazuh-integration)).

4. **Suricata-to-Logstash integration produces indexed documents** (repeat the check from [Step 7](#step-7--complete-the-suricata-to-logstash-integration)).

5. **Winlogbeat-to-Logstash integration produces indexed documents** (repeat the check from [Step 8](#step-8--winlogbeat-on-windows-vms)).

6. **IIS-to-Logstash integration produces indexed documents** (repeat the check from [Step 9](#step-9--filebeat-on-winapp01-for-iis-logs)).

7. **Metricbeat/Packetbeat-to-Logstash integration produces indexed documents** (repeat the check from [Step 10](#step-10--metricbeat-and-packetbeat-on-web01)).

8. **Every Prometheus target is healthy** (repeat the check from [Step 13](#step-13--update-prometheus-scrape-targets)).

9. **Kibana Data Views are browsable in Discover** (repeat the check from [Step 14](#step-14--create-kibana-data-views)).

10. **Firewall rules on the Linux exporters are restricted to MON01's IP, not open to the whole network** — spot-check one:
```bash
# On WEB01
sudo firewall-cmd --list-rich-rules
```

If all ten checks pass, every VM built so far reports into both monitoring ecosystems (Zabbix and Wazuh), the Prometheus/Grafana metrics stack, and the Elastic Stack via five different Beats (Filebeat, Winlogbeat, Metricbeat, Packetbeat) — the forward pointers left earlier in this lab (Suricata-to-Wazuh, Suricata-to-Logstash, IIS-to-Logstash) are now genuinely working end-to-end rather than just configured-and-untested.

---

## Next step

Continue to [`18-secops-firewall-selinux-hardening.md`](./18-secops-firewall-selinux-hardening.md) for security hardening across the fleet.
