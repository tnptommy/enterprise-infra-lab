# 17 — Agent Deployment to All VMs

This document is different in character from every build document before it — there's no new service being built from source here. Instead, it rolls out **Zabbix Agent 2**, **Wazuh Agent**, and **node_exporter**/**windows_exporter** to every VM built so far, and completes two integrations that earlier documents deliberately left as forward pointers:

1. [`08-web01-suricata-nids.md`](./08-web01-suricata-nids.md) and [`13`](./13-mon01-wazuh-manager-configuration.md#step-13--suricata-integration-forward-pointer) — WEB01's Suricata alerts reaching Wazuh Manager via a Wazuh Agent.
2. [`15`](./15-log01-elasticsearch-logstash-kibana.md#step-14--configure-logstash) — WEB01's Suricata alerts also reaching LOG01's Logstash pipeline via Filebeat.

Agents themselves are installed via official packages, not built from source — nobody builds a monitoring agent from source in practice, and unlike the server-side components this lab has spent considerable effort building, an agent is a thin, mostly-static piece of software with little to learn from compiling it yourself.

## VMs covered

| VM | OS | Zabbix Agent | Wazuh Agent | Exporter |
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

- [Step 1 — Zabbix Agent 2 on Linux VMs](#step-1--zabbix-agent-2-on-linux-vms)
- [Step 2 — Zabbix Agent on Windows VMs](#step-2--zabbix-agent-on-windows-vms)
- [Step 3 — Confirm Zabbix host groups populated correctly](#step-3--confirm-zabbix-host-groups-populated-correctly)
- [Step 4 — Wazuh Agent on Linux VMs](#step-4--wazuh-agent-on-linux-vms)
- [Step 5 — Wazuh Agent on Windows VMs](#step-5--wazuh-agent-on-windows-vms)
- [Step 6 — Complete the Suricata-to-Wazuh integration](#step-6--complete-the-suricata-to-wazuh-integration)
- [Step 7 — Complete the Suricata-to-Logstash integration](#step-7--complete-the-suricata-to-logstash-integration)
- [Step 8 — node_exporter on Linux VMs](#step-8--node_exporter-on-linux-vms)
- [Step 9 — windows_exporter on Windows VMs](#step-9--windows_exporter-on-windows-vms)
- [Step 10 — Update Prometheus scrape targets](#step-10--update-prometheus-scrape-targets)
- [Step 11 — Final verification checklist](#step-11--final-verification-checklist)
- [Next step](#next-step)

---

## Step 1 — Zabbix Agent 2 on Linux VMs

Run on **WEB01**, **LOG01**, and **LOG02**:

```bash
sudo rpm --import https://repo.zabbix.com/keys/RPM-GPG-KEY-ZABBIX-8B0D4C4D
sudo dnf install -y https://repo.zabbix.com/zabbix/8.0/release/rocky/10/noarch/zabbix-release-latest.el10.noarch.rpm
sudo dnf clean all

sudo sed -i '/\[zabbix-unstable\]/,/^\[/ s/enabled=0/enabled=1/' /etc/yum.repos.d/zabbix.repo

sudo dnf install -y zabbix-agent2
```
Same repo and unstable-section-enable pattern as MON01's own Zabbix build in [`12`](./12-mon01-zabbix-server-configuration.md#step-9--install-zabbix-build-dependencies) — kept on the same pre-release 8.0 branch for protocol compatibility with the Zabbix Server this agent reports to.

Configure it to point at MON01:
```bash
sudo vi /etc/zabbix/zabbix_agent2.conf
```
```ini
Server=192.168.10.40
ServerActive=192.168.10.40
Hostname=<WEB01, LOG01, or LOG02 — whichever this VM actually is>
```

```bash
sudo systemctl enable --now zabbix-agent2
sudo firewall-cmd --permanent --add-port=10050/tcp
sudo firewall-cmd --reload
```

---

## Step 2 — Zabbix Agent on Windows VMs

Run on **WINAPP01**, **DC01**, and **CLIENT01**:

1. Download the Windows agent MSI from `https://cdn.zabbix.com/zabbix/binaries/stable/8.0/8.0.0/zabbix_agent2-8.0.0-windows-amd64-openssl.msi` (adjust the version path if a newer 8.0.x patch exists by the time you're reading this — check `https://cdn.zabbix.com/zabbix/binaries/stable/8.0/` directly rather than assuming this exact patch number stays current).
2. Run the installer:
```powershell
msiexec /i zabbix_agent2-8.0.0-windows-amd64-openssl.msi /qn SERVER=192.168.10.40 SERVERACTIVE=192.168.10.40 HOSTNAME=%COMPUTERNAME%
```
3. Open the firewall:
```powershell
New-NetFirewallRule -DisplayName "Zabbix Agent" -Direction Inbound -Protocol TCP -LocalPort 10050 -Action Allow
```
4. Confirm the service is running:
```powershell
Get-Service "Zabbix Agent 2"
```

---

## Step 3 — Confirm Zabbix host groups populated correctly

On **MON01**, in the Zabbix Frontend (`https://zabbix.corp-lab.com.vn:8443`):

1. **Data collection → Hosts → Create host** for each of the six VMs in the table above.
2. **Host groups**: assign `Servers` for WEB01/WINAPP01/DC01/LOG01/LOG02, `Workstations` for CLIENT01 — matching the same grouping already established for WSUS client-side targeting in [`10`](./10-gpo-wsus-client-policy.md).
3. **Interfaces**: **Agent**, IP address matching each VM's static IP, port `10050`.
4. **Templates**: link **Linux by Zabbix agent** for the Linux VMs, **Windows by Zabbix agent** for the Windows VMs.
5. **Add** for each host.

Confirm all six show a green "ZBX" icon (not red) within a couple of minutes.

---

## Step 4 — Wazuh Agent on Linux VMs

Run on **WEB01**, **LOG01**, and **LOG02**:

```bash
sudo rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH
sudo dnf install -y https://packages.wazuh.com/4.x/yum/wazuh-agent-4.14.6-1.x86_64.rpm
```

Configure it to point at MON01:
```bash
sudo sed -i "s/MANAGER_IP/192.168.10.40/" /var/ossec/etc/ossec.conf
```
Or, if that placeholder isn't present in this package's default config, edit directly:
```bash
sudo vi /var/ossec/etc/ossec.conf
```
Confirm the `<client><server><address>` block reads `192.168.10.40`.

```bash
sudo systemctl enable --now wazuh-agent
sudo firewall-cmd --permanent --add-port=1514/tcp
sudo firewall-cmd --reload
```

---

## Step 5 — Wazuh Agent on Windows VMs

Run on **WINAPP01**, **DC01**, and **CLIENT01**:

1. Download the Windows agent MSI from `https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.6-1.msi`.
2. Install:
```powershell
msiexec /i wazuh-agent-4.14.6-1.msi /q WAZUH_MANAGER="192.168.10.40"
```
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

On **WEB01**, if Filebeat isn't already installed for another purpose:
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
Same OSS repository as [`13`'s Filebeat install](./13-mon01-wazuh-manager-configuration.md#step-7--install-and-configure-filebeat) — the reasoning there (avoiding the non-OSS distribution check) applies here too, even though this Filebeat instance talks to Logstash rather than directly to an Indexer.

Configure a dedicated input/output specifically for Suricata → Logstash, separate from any Wazuh-alert-shipping Filebeat config that might already exist on this host:
```bash
sudo tee /etc/filebeat/filebeat.yml << 'EOF'
filebeat.inputs:
  - type: log
    paths:
      - /var/log/suricata/eve.json
    json.keys_under_root: true
    json.add_error_key: true

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

## Step 8 — node_exporter on Linux VMs

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

## Step 9 — windows_exporter on Windows VMs

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

## Step 10 — Update Prometheus scrape targets

On **MON01**:
```bash
sudo vi /etc/prometheus/prometheus.yml
```
Extend the `node_exporter` job (created in [`14`'s Step 3](./14-mon01-prometheus-grafana-monitoring.md#step-3--configure-prometheus) with only MON01 itself as a target) to include every VM from [Step 8](#step-8--node_exporter-on-linux-vms), and add a new job for the Windows VMs from [Step 9](#step-9--windows_exporter-on-windows-vms):
```yaml
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance: 'mon01'
      - targets: ['192.168.10.21:9100']
        labels:
          instance: 'web01'
      - targets: ['192.168.10.50:9100']
        labels:
          instance: 'log01'
      - targets: ['192.168.10.51:9100']
        labels:
          instance: 'log02'

  - job_name: 'windows_exporter'
    static_configs:
      - targets: ['192.168.10.15:9182']
        labels:
          instance: 'winapp01'
      - targets: ['192.168.10.10:9182']
        labels:
          instance: 'dc01'
      - targets: ['192.168.10.100:9182']
        labels:
          instance: 'client01'
```
> CLIENT01's IP above (`192.168.10.100`) is the first address in the DHCP pool from [`02`](./02-network-architecture-planning.md) — CLIENT01 doesn't have a static IP, so confirm its actual current lease (`ipconfig /all` on CLIENT01) before trusting this value, since DHCP can hand out a different address within the pool.

```bash
sudo systemctl restart prometheus
```

Verify every target shows `"health": "up"`:
```bash
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -A2 '"health"'
```

---

## Step 11 — Final verification checklist

1. **Every VM shows a healthy Zabbix Agent** (repeat the check from [Step 3](#step-3--confirm-zabbix-host-groups-populated-correctly)).

2. **Every VM shows as connected in Wazuh**:
```bash
# On MON01
sudo /var/ossec/bin/agent_control -l
```
Expect all six agents listed as `Active`.

3. **Suricata-to-Wazuh integration produces decoded alerts** (repeat the check from [Step 6](#step-6--complete-the-suricata-to-wazuh-integration)).

4. **Suricata-to-Logstash integration produces indexed documents** (repeat the check from [Step 7](#step-7--complete-the-suricata-to-logstash-integration)).

5. **Every Prometheus target is healthy** (repeat the check from [Step 10](#step-10--update-prometheus-scrape-targets)).

6. **Firewall rules on the Linux exporters are restricted to MON01's IP, not open to the whole network** — spot-check one:
```bash
# On WEB01
sudo firewall-cmd --list-rich-rules
```

If all six checks pass, every VM built so far reports into both monitoring ecosystems (Zabbix and Wazuh) and the Prometheus/Grafana metrics stack, and the two integrations left as forward pointers earlier in this lab are now genuinely working end-to-end rather than just configured-and-untested.

---

## Next step

Continue to [`18-secops-firewall-selinux-hardening.md`](./18-secops-firewall-selinux-hardening.md) for security hardening across the fleet.
