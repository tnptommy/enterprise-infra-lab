# 08 — WEB01: Suricata network intrusion detection

This document adds **Suricata** to WEB01 as a passive Network Intrusion Detection System (NIDS) — it observes traffic arriving on WEB01's internal network interface and raises alerts on suspicious patterns (port scans, known exploit signatures, protocol anomalies), without sitting inline or blocking anything. This fills the one gap the rest of this lab's security stack doesn't cover: [ModSecurity](./07-web01-lamp-nginx-loadbalancer.md) (not used in this build) and [Wazuh](./13-mon01-wazuh-manager-configuration.md) both operate at the application/host layer, while Suricata is the only component watching raw network traffic.

No new VM is needed — Suricata runs directly on WEB01, since that's the single point where the most traffic in this lab passes through (see the [README's rationale](../README.md) for placing it here rather than a dedicated VM).

| Component | Version used | Source |
|---|---|---|
| Suricata | 8.0.5 | https://www.suricata.io/download/ |

---

## Table of contents

- [Why build from source instead of a package repo](#why-build-from-source-instead-of-a-package-repo)
- [Step 1 — Install build dependencies](#step-1--install-build-dependencies)
- [Step 2 — Install Rust, Cargo, and cbindgen](#step-2--install-rust-cargo-and-cbindgen)
- [Step 3 — Build and install Suricata](#step-3--build-and-install-suricata)
- [Step 4 — Configure the monitored interface and home network](#step-4--configure-the-monitored-interface-and-home-network)
- [Step 5 — Add a test rule](#step-5--add-a-test-rule)
- [Step 6 — Create the systemd service](#step-6--create-the-systemd-service)
- [Step 7 — Fire the test rule and confirm an alert](#step-7--fire-the-test-rule-and-confirm-an-alert)
- [Step 8 — Update the Emerging Threats ruleset](#step-8--update-the-emerging-threats-ruleset)
- [Forward pointers: Filebeat and Wazuh integration](#forward-pointers-filebeat-and-wazuh-integration)
- [Step 9 — Final verification checklist](#step-9--final-verification-checklist)
- [Next step](#next-step)

---

## Why build from source instead of a package repo

Rocky Linux's own EPEL repository only carries a very old Suricata release (5.0.8 as of this writing) — several major versions behind. OISF (the organization behind Suricata) maintains COPR repositories, but they're versioned per-distribution-release and per-branch (`@oisf/suricata-7.0`, etc.) and support for the newest distributions like Rocky Linux 10 tends to lag behind the source release. Building 8.0.5 from source guarantees the exact version used throughout this guide, consistent with how Nginx, Apache, PHP, and MariaDB were already built in [`07`](./07-web01-lamp-nginx-loadbalancer.md).

---

## Step 1 — Install build dependencies

EPEL and CRB were already enabled on this VM's [golden baseline](./05-golden-baseline-rocky-linux-10.md#step-5--apply-package-baseline) — confirm they're still active, then install Suricata's build requirements:

```bash
sudo dnf repolist | grep -E "epel|crb"

sudo dnf install -y gcc gcc-c++ make automake autoconf libtool \
    jansson-devel libpcap-devel libyaml-devel pcre2-devel zlib-devel \
    file-devel nss-devel nspr-devel libcap-ng-devel lz4-devel \
    python3 python3-pyyaml python3-devel libmaxminddb-devel
```
> `libmaxminddb-devel` is required specifically because of the `--enable-geoip` flag used in [Step 3](#step-3--build-and-install-suricata) — without it, `./configure` fails with `libmaxminddb GeoIP2 library not found`.

---

## Step 2 — Install Rust, Cargo, and cbindgen

Suricata 7.x+ requires a Rust toolchain to build:

```bash
sudo dnf install -y rustc cargo

which cbindgen || cargo install --force cbindgen

echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/.cargo/bin:$PATH"
```

---

## Step 3 — Build and install Suricata

```bash
cd ~
wget https://www.openinfosecfoundation.org/download/suricata-8.0.5.tar.gz
tar -xzvf suricata-8.0.5.tar.gz
cd suricata-8.0.5

./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var \
    --enable-lua --enable-geoip

make -j$(nproc)
```

Rather than a plain `make install`, use `install-full` — this single step also generates a working `/etc/suricata/suricata.yaml`, creates the expected runtime directories, and downloads the free Emerging Threats Open ruleset automatically:

```bash
sudo make install-full
```

Confirm:
```bash
suricata -V
ls /var/lib/suricata/rules/
```

---

## Step 4 — Configure the monitored interface and home network

Identify the internal network interface (`Internal-LabNet` per the [naming convention established for WEB01](./07-web01-lamp-nginx-loadbalancer.md#step-2--set-hostname-and-static-ip)) — this is the interface carrying traffic between WEB01 and the rest of `192.168.10.0/24`, and the one worth watching for lab-internal attack simulation:

```bash
ip a
```
Note the device name (commonly `ens224` or similar — whatever corresponds to the `192.168.10.21` address).

Edit the configuration:
```bash
sudo vi /etc/suricata/suricata.yaml
```

1. Set the home/external network definitions (find the `vars: address-groups:` section):
```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.10.0/24]"
    EXTERNAL_NET: "!$HOME_NET"
```

2. Set the monitored interface (find the `af-packet:` section):
```yaml
af-packet:
  - interface: ens224
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes
```
Replace `ens224` with the actual device name confirmed above.

3. Confirm EVE JSON output is enabled (it is, by default, from `install-full`) — find the `outputs:` section and verify:
```yaml
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
```
This is the structured log format that Filebeat will later ship to ELK01 and/or Wazuh — see [Forward pointers](#forward-pointers-filebeat-and-wazuh-integration) below.

---

## Step 5 — Add a test rule

Create a simple custom rule that fires on any ICMP packet, so the pipeline can be verified end-to-end without waiting for a real attack signature to match:

```bash
sudo tee /var/lib/suricata/rules/local.rules << 'EOF'
alert icmp any any -> any any (msg:"LOCAL TEST - ICMP packet detected"; sid:1000001; rev:1;)
EOF
```

Register it alongside the Emerging Threats ruleset (find the `rule-files:` section in `suricata.yaml`):
```bash
sudo sed -i '/rule-files:/a\  - local.rules' /etc/suricata/suricata.yaml
```

---

## Step 6 — Create the systemd service

```bash
sudo tee /etc/systemd/system/suricata.service << 'EOF'
[Unit]
Description=Suricata Intrusion Detection Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/suricata -c /etc/suricata/suricata.yaml --af-packet
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
```

Test the configuration before starting the service:
```bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```
Expect no errors, ending in something like `Configuration provided was successfully loaded`.

Enable and start:
```bash
sudo systemctl enable --now suricata
sudo systemctl status suricata
```

---

## Step 7 — Fire the test rule and confirm an alert

From **another VM** on `192.168.10.0/24` (e.g. DC01, once built), ping WEB01:
```powershell
ping 192.168.10.21
```
Or from a Linux VM:
```bash
ping -c 4 192.168.10.21
```

Back on WEB01, confirm the test rule fired:
```bash
sudo tail -20 /var/log/suricata/fast.log
```
Expect a line containing `LOCAL TEST - ICMP packet detected`.

Also check the structured EVE JSON output:
```bash
sudo tail -5 /var/log/suricata/eve.json | grep alert
```

Once confirmed working, the test rule can be left in place (harmless — ICMP traffic is common and this rule only alerts, never blocks) or removed by reverting Step 5's changes.

---

## Step 8 — Update the Emerging Threats ruleset

`install-full` downloaded rules once, at install time. Keep them current going forward:

```bash
sudo suricata-update
sudo systemctl restart suricata
```

Consider scheduling this via cron alongside the other maintenance tasks covered in [`18-automation-backup-scheduling.md`](./18-automation-backup-scheduling.md).

---

## Forward pointers: Filebeat and Wazuh integration

Suricata is fully functional as of this document, but its alerts currently only live in local log files on WEB01. Two later documents build on this:

- [`16-agent-deployment-all-vms.md`](./16-agent-deployment-all-vms.md) installs Filebeat on WEB01 with its built-in Suricata module enabled, shipping `eve.json` to ELK01 for centralized search and dashboards in Kibana.
- [`13-mon01-wazuh-manager-configuration.md`](./13-mon01-wazuh-manager-configuration.md) optionally configures Wazuh Manager on MON01 to ingest the same `eve.json` file directly, correlating network-layer alerts from Suricata with the host-layer FIM alerts Wazuh already generates — giving a single place to see "something changed on disk" and "something suspicious crossed the network" for the same incident.

Nothing further needs to be done on WEB01 itself until those documents are reached.

---

## Step 9 — Final verification checklist

1. **Suricata service is running:**
```bash
sudo systemctl status suricata
```

2. **Configuration loads without error:**
```bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```

3. **Correct interface is being monitored:**
```bash
sudo ss -i | grep -A2 suricata
```

4. **Test rule fires on real traffic** (repeat [Step 7](#step-7--fire-the-test-rule-and-confirm-an-alert) if not already confirmed).

5. **EVE JSON is being written:**
```bash
ls -la /var/log/suricata/eve.json
```

6. **Ruleset is current:**
```bash
sudo suricata-update list-sources
```

If all six checks pass, Suricata is ready, and this rounds out WEB01's build.

---

## Next step

Continue to [`09-winapp01-iis-sql-wsus.md`](./09-winapp01-iis-sql-wsus.md) to build the Windows application server (IIS, SQL Server, WSUS).
