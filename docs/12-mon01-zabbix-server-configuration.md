# 12 — MON01: Zabbix Server Configuration

This document clones the Rocky Linux 10 [Golden Baseline](./05-golden-baseline-rocky-linux-10.md) into **MON01** and builds the first half of this lab's monitoring stack: Zabbix Server, its MariaDB backend, and the web frontend — followed by templates, triggers, actions, and a dashboard using MON01 itself as the first monitored host. Wazuh ([`13`](./13-mon01-wazuh-manager-configuration.md)) and Prometheus/Grafana ([`14`](./14-mon01-prometheus-grafana-monitoring.md)) join this same VM in the next two documents.

| Component | Version used | Source |
|---|---|---|
| Zabbix | 8.0 (pre-release — latest alpha/beta build) | https://repo.zabbix.com/zabbix/8.0/ |
| MariaDB | 12.3 LTS | https://mariadb.org/download/ |

> **This uses Zabbix 8.0's pre-release packages (alpha/beta builds), not a stable release.** Zabbix's official repository ships 8.0 packages under an `unstable` repo section that's disabled by default — installing them requires explicitly enabling it, covered in [Step 9](#step-9--install-zabbix-server-frontend-and-agent). Pre-release software can have unfinished features, schema changes between builds, and rougher edges than a stable release; this is a deliberate choice for this lab to get hands-on with upcoming Zabbix 8.0 features early, not a recommendation for production use. If you'd rather build on the stable, long-term-supported branch instead, use **Zabbix 7.0 LTS** (supported through June 2029) by substituting `7.0` for `8.0` in the repository URL in Step 9, and skip enabling the unstable repo section.

---

## Table of contents

- [VM specification](#vm-specification)
- [Step 1 — Clone the golden baseline and provision disks](#step-1--clone-the-golden-baseline-and-provision-disks)
- [Step 2 — Set hostname and static IP](#step-2--set-hostname-and-static-ip)
- [Step 3 — Grow the root filesystem](#step-3--grow-the-root-filesystem)
- [Step 4 — Create swap](#step-4--create-swap)
- [Step 5 — Domain-join](#step-5--domain-join)
- [Step 6 — Add DNS records on DC01](#step-6--add-dns-records-on-dc01)
- [Step 7 — Install MariaDB](#step-7--install-mariadb)
- [Step 8 — Create the Zabbix database](#step-8--create-the-zabbix-database)
- [Step 9 — Install Zabbix server, frontend, and agent](#step-9--install-zabbix-server-frontend-and-agent)
- [Step 10 — Import the initial schema](#step-10--import-the-initial-schema)
- [Step 11 — Configure the Zabbix server](#step-11--configure-the-zabbix-server)
- [Step 12 — Configure PHP for the frontend](#step-12--configure-php-for-the-frontend)
- [Step 13 — SELinux and firewall](#step-13--selinux-and-firewall)
- [Step 14 — Start services and complete the frontend wizard](#step-14--start-services-and-complete-the-frontend-wizard)
- [Step 15 — Change the default Admin password](#step-15--change-the-default-admin-password)
- [Step 16 — Add MON01 as the first monitored host](#step-16--add-mon01-as-the-first-monitored-host)
- [Step 17 — Create triggers of different types](#step-17--create-triggers-of-different-types)
- [Step 18 — Build a basic dashboard](#step-18--build-a-basic-dashboard)
- [Step 19 — Final verification checklist](#step-19--final-verification-checklist)
- [Next step](#next-step)

---

## VM specification

| Setting | Value |
|---|---|
| VM name (VMware Library) | `MON01_10.40` |
| OS hostname | `MON01` |
| vCPU | 4 |
| RAM | 10–12 GB |
| Disk 1 (OS + swap) | 60 GB (40 GB OS + 20 GB swap) |
| Disk 2 (data) | 20 GB — Zabbix/Wazuh/Prometheus data |
| NIC 1 (NAT) | DHCP |
| NIC 2 (Host-only) | Static — `192.168.10.40` |

---

## Step 1 — Clone the golden baseline and provision disks

Follow [`05-golden-baseline-rocky-linux-10.md`'s cloning instructions](./05-golden-baseline-rocky-linux-10.md#cloning-this-baseline-later):

1. Right-click `GoldenBaseline-Rocky10` → **Manage → Clone…** → **Create a full clone**.
2. Name the clone `MON01_10.40`.
3. Adjust vCPU/RAM in **VM → Settings** per the [VM specification](#vm-specification) above.
4. With the VM still powered off, expand the OS disk to **60 GB**: **VM → Settings → Hard Disk → Utilities → Expand…**.
5. Add the second disk while still powered off: **VM → Settings → Add… → Hard Disk → Create a new virtual disk**, size **20 GB**.
6. Power on the VM and reconnect via PuTTY using its NAT-assigned IP (`ip a`), same approach as [`05`'s Step 2](./05-golden-baseline-rocky-linux-10.md#step-2--enable-ssh-and-switch-to-putty).

---

## Step 2 — Set hostname and static IP

```bash
sudo hostnamectl set-hostname MON01
```

Configure NIC 2 (Host-only) with a static IP:
```bash
sudo nmcli con mod "Wired connection 1" ipv4.addresses 192.168.10.40/24
sudo nmcli con mod "Wired connection 1" ipv4.dns 192.168.10.10
sudo nmcli con mod "Wired connection 1" ipv4.method manual
sudo nmcli con up "Wired connection 1"
```

Stop NIC 1 (NAT) from polluting DNS resolution — required before domain-join in Step 5 will work (see the full explanation in [`02`](./02-network-architecture-planning.md#rocky-linux-nic-2--host-only)):
```bash
nmcli con show
sudo nmcli con modify "ens160" ipv4.ignore-auto-dns yes
sudo nmcli con up "ens160"
cat /etc/resolv.conf
```

---

## Step 3 — Grow the root filesystem

Install `growpart`, identify the real device/partition/VG names, then grow — **never assume `/dev/sda` or a volume group named `rl`**, per the lesson learned building WEB01 (see [`05`'s expansion procedure](./05-golden-baseline-rocky-linux-10.md#cloning-this-baseline-later) for the full explanation of why):

```bash
sudo dnf install -y cloud-utils-growpart

lsblk
sudo vgs
sudo lvs
```

Using the actual names confirmed above (3 for UEFI layouts, 2 for BIOS):
```bash
# Example for a SATA/SCSI disk (/dev/sda)
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3

# Example for an NVMe disk (/dev/nvme0n1)
sudo growpart /dev/nvme0n1 3
sudo pvresize /dev/nvme0n1p3

# Replace vg_name-lv_name with the actual /dev/mapper/ path from `lvs`
sudo lvextend -l +100%FREE /dev/mapper/vg_name-lv_name
sudo xfs_growfs /
```

---

## Step 4 — Create swap

20 GB (2× the 10 GB midpoint of this VM's 10–12 GB RAM range, rounded to the nearest 10 GB per this guide's convention):

```bash
sudo fallocate -l 20G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

MON01 will run Wazuh Indexer ([`13`](./13-mon01-wazuh-manager-configuration.md)), a JVM-based service — tune `vm.swappiness` down now to avoid JVM garbage-collection stalls later, per the [README's note on this](../README.md#virtual-machine-inventory):
```bash
echo 'vm.swappiness=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## Step 5 — Domain-join

```bash
sudo dnf install -y realmd sssd-common sssd-ad adcli krb5-workstation samba-common-tools oddjob oddjob-mkhomedir

sudo realm discover corp-lab.com.vn
sudo realm join -U administrator corp-lab.com.vn
```
Enter the `CORP-LAB\Administrator` password from [KeePass](./03-remote-access-tooling-setup.md#keepass) when prompted.

Verify:
```bash
realm list
```

---

## Step 6 — Add DNS records on DC01

MON01 needs the same manual forward `A` and reverse `PTR` records every Linux VM in this lab requires, per [`06`'s DNS section](./06-dc01-active-directory-dns-dhcp.md#step-5--configure-dns) — `realmd`/`sssd` domain-join doesn't register DNS automatically the way Windows does.

On **DC01**:
```powershell
Add-DnsServerResourceRecordA -ZoneName "corp-lab.com.vn" -Name "mon01" -IPv4Address "192.168.10.40"
Add-DnsServerResourceRecordPtr -ZoneName "10.168.192.in-addr.arpa" -Name "40" -PtrDomainName "mon01.corp-lab.com.vn."
```

---

## Step 7 — Install MariaDB

```bash
curl -LsS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | sudo bash -s -- --mariadb-server-version="mariadb-12.3"

sudo dnf install -y MariaDB-server MariaDB-client

sudo systemctl enable --now mariadb
sudo mysql_secure_installation
```
Answer **Y** to every prompt, setting a strong root password — store it in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `MON01_10.40` group.

---

## Step 8 — Create the Zabbix database

```bash
sudo mysql -u root -p
```
```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER zabbix@localhost IDENTIFIED BY 'Your-Strong-Password-Here';
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
SET GLOBAL log_bin_trust_function_creators = 1;
FLUSH PRIVILEGES;
EXIT;
```
Store the `zabbix` database user's password in KeePass alongside the MariaDB root password.

> `log_bin_trust_function_creators` is required for importing Zabbix's schema on a server with binary logging enabled — it's set globally here rather than added to `my.cnf`, since it only matters during the one-time import in the next step.

---

## Step 9 — Install Zabbix server, frontend, and agent

```bash
sudo rpm -Uvh https://repo.zabbix.com/zabbix/8.0/release/rocky/10/noarch/zabbix-release-latest.el10.noarch.rpm
sudo dnf clean all
```

Enable the `unstable` repo section to get pre-release (alpha/beta) packages instead of waiting for a stable 8.0 release:
```bash
sudo sed -i '/\[zabbix-unstable\]/,/^\[/ s/enabled=0/enabled=1/' /etc/yum.repos.d/zabbix.repo
```
Confirm it took effect:
```bash
grep -A5 "\[zabbix-unstable\]" /etc/yum.repos.d/zabbix.repo
```
Expect `enabled=1` under that section.

```bash
sudo dnf install -y zabbix-server-mysql zabbix-web-mysql zabbix-apache-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent2
```

Confirm which pre-release build was actually installed:
```bash
rpm -q zabbix-server-mysql
```

---

## Step 10 — Import the initial schema

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | sudo mysql --default-character-set=utf8mb4 -u zabbix -p zabbix
```
Enter the `zabbix` database user's password when prompted. This takes a minute or two — it's populating the full schema and default templates.

Revert the temporary binary logging setting from Step 8 (no longer needed after import):
```bash
sudo mysql -u root -p -e "SET GLOBAL log_bin_trust_function_creators = 0;"
```

---

## Step 11 — Configure the Zabbix server

```bash
sudo vi /etc/zabbix/zabbix_server.conf
```
Set:
```ini
DBPassword=Your-Strong-Password-Here
```
(the `zabbix` database user's password from Step 8)

---

## Step 12 — Configure PHP for the frontend

```bash
sudo vi /etc/php-fpm.d/zabbix.conf
```
Find and set the timezone:
```ini
php_value[date.timezone] = Asia/Ho_Chi_Minh
```

---

## Step 13 — SELinux and firewall

```bash
sudo setsebool -P httpd_can_connect_zabbix on
sudo setsebool -P httpd_can_network_connect_db on

sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-port=10051/tcp
sudo firewall-cmd --permanent --add-port=10050/tcp
sudo firewall-cmd --reload
```
`10051/tcp` is the Zabbix trapper port (server receiving active-check data from agents); `10050/tcp` is the agent's own listening port (server polling agents directly, needed on MON01 too since it monitors itself in Step 16).

---

## Step 14 — Start services and complete the frontend wizard

```bash
sudo systemctl enable --now zabbix-server zabbix-agent2 httpd php-fpm
```

From the host machine (or any VM with network access to MON01), browse to:
```
http://192.168.10.40
```

Work through the setup wizard:
1. **Welcome** → **Next step**.
2. **Check of pre-requisites** — everything should show **OK**. If PHP settings show as failing, revisit Step 12 and confirm `php-fpm` was restarted after the edit (`sudo systemctl restart php-fpm`).
3. **Configure DB connection**: Database type `MySQL`, Host `localhost`, Port `0` (default), Database name `zabbix`, User `zabbix`, Password (from Step 8).
4. **Zabbix server details**: Host `localhost`, Port `10051`, Name `MON01` (or leave blank).
5. **Pre-installation summary** → **Next step**.
6. **Install** → **Finish**.

You'll land on the Zabbix login page.

---

## Step 15 — Change the default Admin password

1. Log in with the default credentials: username `Admin`, password `zabbix`.
2. Immediately change it: click the user icon (top right) → **Profile** → **Change password**.
3. Store the new password in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `MON01_10.40` group, entry "Zabbix Frontend admin".

---

## Step 16 — Add MON01 as the first monitored host

Full agent rollout to every other VM in this lab happens in [`17-agent-deployment-all-vms.md`](./17-agent-deployment-all-vms.md) — this step just gets one working example end-to-end using the agent already installed on MON01 itself in Step 9.

1. **Data collection → Hosts → Create host**.
2. **Host name**: `MON01`.
3. **Host groups**: create a new group `Monitoring Servers`.
4. **Interfaces**: click **Add** next to Agent → IP address `127.0.0.1` (or `192.168.10.40`), port `10050`.
5. **Templates** tab → **Add** → search for and select **Linux by Zabbix agent**.
6. **Add**.

Confirm the agent is reachable — after a minute or two, **Data collection → Hosts** should show a green "ZBX" icon next to `MON01`, not red.

---

## Step 17 — Create triggers of different types

The **Linux by Zabbix agent** template already ships with a comprehensive set of triggers (high CPU, low disk space, etc.). Rather than duplicate those, this step builds a small set of custom triggers that each demonstrate a distinct trigger mechanism Zabbix supports, using MON01 itself as the test host.

**1. Simple threshold trigger** (single condition, `avg` over a time window):

1. **Data collection → Hosts** → click **Triggers** next to `MON01` → **Create trigger**.
2. **Name**: `MON01: High memory utilization`.
3. **Expression**: **Add** → item `Memory utilization`, function `avg`, time period `5m`, operator `>`, value `90` → **Insert**.
4. **Severity**: `Warning` → **Add**.

**2. Multi-condition trigger** (combining two conditions with a logical operator — fires only when *both* are true, reducing false positives compared to either condition alone):

5. **Create trigger** → **Name**: `MON01: High CPU sustained with load`.
6. **Expression**: build two conditions joined by `and`:
```
avg(/MON01/system.cpu.util,5m)>80 and avg(/MON01/system.cpu.load,5m)>4
```
(Type the expression directly in the **Expression** field rather than using the item picker for the second half — both item keys must already exist on the host via the linked template.)
7. **Severity**: `Average` → **Add**.

**3. `nodata()` trigger** (fires when a host *stops reporting* entirely — different failure mode than a threshold being crossed, and one of the most operationally important trigger types, since a silent host produces no threshold breaches at all):

8. **Create trigger** → **Name**: `MON01: Agent not reporting`.
9. **Expression**:
```
nodata(/MON01/agent.ping,10m)=1
```
10. **Severity**: `High` → **Add**.

**4. Trigger dependency** (suppress a lower-priority alert when a higher-priority one covering the same root cause is already firing — prevents alert floods where one real problem generates many redundant notifications):

11. Open the **`MON01: High CPU sustained with load`** trigger created above → **Dependencies** tab → **Add** → select **`MON01: Agent not reporting`** → **Update**.
This means: if the host is already flagged as not reporting at all, the (now-meaningless, since there's no fresh data) CPU trigger won't separately notify on top of it.

---

## Step 18 — Build a basic dashboard

1. **Dashboards → All dashboards → Create dashboard**.
2. **Name**: `Infrastructure Overview`.
3. **Add widget** → type **Graph (classic)** → select host `MON01`, item `CPU utilization`.
4. **Add** another widget → type **Problems** → leave default filters (shows any active problems/triggers across all hosts).
5. **Add** another widget → type **Host availability** → shows a quick up/down summary, useful once more hosts are added in [`17`](./17-agent-deployment-all-vms.md).
6. **Apply** / **Save changes**.

---

## Step 19 — Final verification checklist

1. **Zabbix server running:**
```bash
sudo systemctl status zabbix-server
```

2. **Frontend reachable:**
```bash
curl -I http://192.168.10.40
```

3. **Database connection healthy** — check `/var/log/zabbix/zabbix_server.log` for connection errors:
```bash
sudo tail -20 /var/log/zabbix/zabbix_server.log
```

4. **MON01 host shows green (reachable) in Data collection → Hosts.**

5. **Triggers were created** (repeat the checks from [Step 17](#step-17--create-triggers-of-different-types)).

6. **Domain-join succeeded:**
```bash
realm list
```

7. **DNS records resolve:**
```bash
nslookup mon01.corp-lab.com.vn
nslookup 192.168.10.40
```

8. **Credentials are stored, not memorized** — confirm the local sudo user password, MariaDB root password, `zabbix` DB user password, and Zabbix Frontend Admin password are all saved in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `MON01_10.40` group.

If all eight checks pass, Zabbix is ready on MON01.

---

## Next step

Continue to [`13-mon01-wazuh-manager-configuration.md`](./13-mon01-wazuh-manager-configuration.md) to add Wazuh Manager, Indexer, and Dashboard to this same VM.
