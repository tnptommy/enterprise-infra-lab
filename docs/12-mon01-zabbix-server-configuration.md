# 12 — MON01: Zabbix Server Configuration

This document clones the Rocky Linux 10 [Golden Baseline](./05-golden-baseline-rocky-linux-10.md) into **MON01** and builds the first half of this lab's monitoring stack: Zabbix Server (built from source), its MariaDB backend, and the web frontend — followed by templates, triggers, and a dashboard using MON01 itself as the first monitored host. Wazuh ([`13`](./13-mon01-wazuh-manager-configuration.md)) and Prometheus/Grafana ([`14`](./14-mon01-prometheus-grafana-monitoring.md)) join this same VM in the next two documents.

| Component | Version used | Source |
|---|---|---|
| Zabbix | 8.0.0beta2 (pre-release, built from source) | https://www.zabbix.com/download_sources |
| MariaDB | 12.3.2 (built from source) | https://mariadb.org/download/ |
| Apache HTTP Server | 2.4.68 (built from source) | https://httpd.apache.org/download.cgi |
| PHP | 8.5.8 (built from source) | https://www.php.net/downloads |

> **Everything on this VM — MariaDB, Apache, PHP, and Zabbix itself — is built from source**, consistent with this guide's general approach (see [`07`](./07-web01-lamp-nginx-loadbalancer.md) for the same pattern applied to WEB01's LAMP stack). MariaDB and Apache/PHP here reuse the exact same versions and build steps already established in `07`, adapted to a single Apache instance (MON01 has no load balancer in front of it, so Apache listens directly on port 80) with PHP linked explicitly against the from-source MariaDB client library rather than a system one. Zabbix itself uses the latest pre-release source tarball (`8.0.0beta2`) rather than a stable release — pre-release software can have unfinished features and rougher edges; this is a deliberate choice to get hands-on with upcoming Zabbix 8.0 features early, not a recommendation for production use. If you'd rather build the stable, long-term-supported branch instead, substitute the **Zabbix 7.0 LTS** source tarball URL (supported through June 2029) from the same download page in [Step 11](#step-11--download-and-configure-zabbix-source).

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
- [Step 8 — Build MariaDB from source](#step-8--build-mariadb-from-source)
- [Step 9 — Create the Zabbix database](#step-9--create-the-zabbix-database)
- [Step 10 — Install Zabbix build dependencies](#step-10--install-zabbix-build-dependencies)
- [Step 11 — Download and configure Zabbix source](#step-11--download-and-configure-zabbix-source)
- [Step 12 — Compile and install Zabbix server and agent 2](#step-12--compile-and-install-zabbix-server-and-agent-2)
- [Step 13 — Import the initial schema](#step-13--import-the-initial-schema)
- [Step 14 — Configure the Zabbix server and agent](#step-14--configure-the-zabbix-server-and-agent)
- [Step 15 — Create systemd services](#step-15--create-systemd-services)
- [Step 16 — Build Apache and PHP from source, deploy the frontend](#step-16--build-apache-and-php-from-source-deploy-the-frontend)
- [Step 17 — SELinux and firewall](#step-17--selinux-and-firewall)
- [Step 18 — Start services and complete the frontend wizard](#step-18--start-services-and-complete-the-frontend-wizard)
- [Step 19 — Enable HTTPS-only access](#step-19--enable-https-only-access)
- [Step 20 — Change the default Admin password](#step-20--change-the-default-admin-password)
- [Step 21 — Add MON01 as the first monitored host](#step-21--add-mon01-as-the-first-monitored-host)
- [Step 22 — Create triggers of different types](#step-22--create-triggers-of-different-types)
- [Step 23 — Build a basic dashboard](#step-23--build-a-basic-dashboard)
- [Step 24 — Final verification checklist](#step-24--final-verification-checklist)
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

## Step 5 — Partition and mount the data disk

The 20 GB second disk added in [Step 1](#step-1--clone-the-golden-baseline-and-provision-disks) exists but hasn't been used yet — mount it now, before installing anything, so MariaDB and Wazuh Indexer (added in [`13`](./13-mon01-wazuh-manager-configuration.md)) can be pointed at it from the start rather than needing a live data migration later. This matters more than it might seem: without this, every data-heavy service defaults to writing into the 60 GB OS disk, which fills up far faster than expected once Zabbix's database and Wazuh's indices start accumulating real data — a problem discovered the hard way while building this exact VM, where the OS disk hit 93% used partway through the Wazuh Indexer build in [`13`](./13-mon01-wazuh-manager-configuration.md).

Identify the disk — on an NVMe-based VM (per the [golden baseline's device-naming note](./05-golden-baseline-rocky-linux-10.md#cloning-this-baseline-later)) the second disk typically appears as `/dev/nvme0n2`, distinct from the OS disk's `/dev/nvme0n1`:
```bash
lsblk
```

Partition and format it:
```bash
sudo parted /dev/nvme0n2 --script mklabel gpt mkpart primary 0% 100%
lsblk
```
Confirm the new partition (`/dev/nvme0n2p1` for NVMe, or `/dev/sdb1` on a SATA/SCSI-based VM) shows close to the full 20 GB before formatting — a partition showing only a few dozen MB means `parted`'s `mkpart` range wasn't applied correctly; re-run `parted /dev/nvme0n2 --script rm 1` and retry the `mkpart` command above rather than formatting a wrongly-sized partition.

```bash
sudo mkfs.xfs /dev/nvme0n2p1
sudo mkdir -p /mnt/data
sudo mount /dev/nvme0n2p1 /mnt/data
```

Add it to `/etc/fstab` for it to persist across reboots — get the UUID first, and **do not wrap it in quotes** in the fstab line, which is a syntax `mount -a` silently mishandles:
```bash
sudo blkid /dev/nvme0n2p1
```
```bash
echo 'UUID=<uuid-from-above> /mnt/data xfs defaults 0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
```

Test the fstab entry actually works before trusting it to survive a real reboot:
```bash
sudo umount /mnt/data
sudo mount -a
df -h /mnt/data
```
Expect `/mnt/data` to remount cleanly, showing close to 20 GB available.

---

## Step 6 — Domain-join

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

## Step 7 — Add DNS records on DC01

MON01 needs the same manual forward `A` and reverse `PTR` records every Linux VM in this lab requires, per [`06`'s DNS section](./06-dc01-active-directory-dns-dhcp.md#step-5--configure-dns) — `realmd`/`sssd` domain-join doesn't register DNS automatically the way Windows does.

On **DC01**:
```powershell
Add-DnsServerResourceRecordA -ZoneName "corp-lab.com.vn" -Name "mon01" -IPv4Address "192.168.10.40"
Add-DnsServerResourceRecordPtr -ZoneName "10.168.192.in-addr.arpa" -Name "40" -PtrDomainName "mon01.corp-lab.com.vn."
```

---

## Step 8 — Build MariaDB from source

Same version and build process as [`07`'s MariaDB build](./07-web01-lamp-nginx-loadbalancer.md#step-8--build-mariadb-from-source), with one difference: the data directory points at the data disk mounted in [Step 5](#step-5--partition-and-mount-the-data-disk) (`/mnt/data/mariadb`) rather than living on the OS disk under `/opt/mariadb/data` — MariaDB's actual data grows continuously and is exactly the kind of thing that disk was set aside for:

```bash
cd ~
wget https://downloads.mariadb.org/rest-api/mariadb/12.3.2/mariadb-12.3.2.tar.gz
tar -xzvf mariadb-12.3.2.tar.gz
cd mariadb-12.3.2

sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y cmake ncurses-devel bison libxml2-devel gnutls-devel \
    openssl-devel libaio-devel

mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/opt/mariadb -DMYSQL_DATADIR=/mnt/data/mariadb
make -j$(nproc)
sudo make install

sudo useradd -r -s /sbin/nologin mysql
sudo mkdir -p /mnt/data/mariadb
sudo chown -R mysql:mysql /opt/mariadb /mnt/data/mariadb
sudo /opt/mariadb/scripts/mariadb-install-db --user=mysql --datadir=/mnt/data/mariadb --basedir=/opt/mariadb
```

Create a systemd unit:
```bash
sudo tee /etc/systemd/system/mariadb.service << 'EOF'
[Unit]
Description=MariaDB database server
After=network.target

[Service]
User=mysql
Group=mysql
ExecStart=/opt/mariadb/bin/mariadbd --datadir=/mnt/data/mariadb --basedir=/opt/mariadb
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now mariadb
```

Secure the installation and set the root password:
```bash
sudo /opt/mariadb/bin/mariadb-secure-installation
```
Answer **Y** to every prompt, setting a strong root password — store it in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `MON01_10.40` group.

---

## Step 9 — Create the Zabbix database

```bash
sudo /opt/mariadb/bin/mariadb -u root -p
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

> `log_bin_trust_function_creators` is required for importing Zabbix's schema on a server with binary logging enabled — it's set globally here rather than added to `my.cnf`, since it only matters during the one-time import in Step 12.

---

## Step 10 — Install Zabbix build dependencies

```bash
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y pkgconfig mariadb-devel openssl-devel pcre2-devel \
    libxml2-devel libcurl-devel libevent-devel net-snmp-devel \
    golang git
```
`golang` is required specifically for **Zabbix Agent 2**, which is written in Go rather than C — everything else here is for the C-based `zabbix_server` build.

Create the dedicated `zabbix` system user Zabbix's daemons drop privileges to after starting as root (standard practice, per Zabbix's own installation documentation — never run monitoring daemons as root or another privileged account long-term):
```bash
sudo groupadd --system zabbix
sudo useradd --system -g zabbix -d /usr/lib/zabbix -s /sbin/nologin -c "Zabbix Monitoring System" zabbix
```

---

## Step 11 — Download and configure Zabbix source

```bash
cd ~
wget https://cdn.zabbix.com/zabbix/sources/development/8.0/zabbix-8.0.0beta2.tar.gz
tar -xzvf zabbix-8.0.0beta2.tar.gz
mv zabbix-8.0.0beta2 zabbix-8.0.0
cd zabbix-8.0.0

./configure \
  --prefix=/opt/zabbix \
  --enable-server \
  --enable-agent2 \
  --with-mysql \
  --with-openssl \
  --with-libpcre2 \
  --with-libevent \
  --with-net-snmp \
  --enable-ipv6
```

**Explanation of the key flags:**
- `--enable-server` → builds `zabbix_server`, the core monitoring daemon.
- `--enable-agent2` → builds `zabbix_agent2` (Go-based) alongside it, so MON01 can monitor itself in [Step 21](#step-21--add-mon01-as-the-first-monitored-host) without a separate build pass.
- `--with-mysql` → links against MariaDB's client libraries (MariaDB is wire-compatible with MySQL, so Zabbix's `--with-mysql` flag works against it correctly).
- `--with-libpcre2` → regex support for item/trigger matching.
- `--with-net-snmp` → optional SNMP monitoring support, useful later for network devices.

> `--with-unixodbc` (optional ODBC database monitoring support) is intentionally skipped — `unixODBC-devel` isn't yet available in Rocky Linux 10's repositories (including EPEL/CRB) as of this writing. This only disables the optional ODBC item type; every other Zabbix monitoring feature used in this lab is unaffected.

---

## Step 12 — Compile and install Zabbix server and agent 2

```bash
make -j$(nproc)
sudo make install
```

This installs binaries to `/opt/zabbix/sbin/` (`zabbix_server`) and `/opt/zabbix/sbin/` (`zabbix_agent2`), config files to `/opt/zabbix/etc/`, and the frontend's PHP source to the extracted `ui/` directory (copied separately in [Step 16](#step-16--build-apache-and-php-from-source-deploy-the-frontend) — `make install` doesn't handle the frontend, since it's plain PHP with no compilation step).

Confirm both binaries built successfully:
```bash
/opt/zabbix/sbin/zabbix_server --version
/opt/zabbix/sbin/zabbix_agent2 --version
```

---

## Step 13 — Import the initial schema

Source builds ship the schema as individual SQL files rather than the single combined `server.sql.gz` the package-based install uses:

```bash
cd ~/zabbix-8.0.0/database/mysql
sudo /opt/mariadb/bin/mariadb --default-character-set=utf8mb4 -u zabbix -p zabbix < schema.sql
sudo /opt/mariadb/bin/mariadb --default-character-set=utf8mb4 -u zabbix -p zabbix < images.sql
sudo /opt/mariadb/bin/mariadb --default-character-set=utf8mb4 -u zabbix -p zabbix < data.sql
```
Enter the `zabbix` database user's password each time. `schema.sql` takes the longest (creating all tables); `images.sql` and `data.sql` are quick (default icons and templates).

Revert the temporary binary logging setting from Step 8 (no longer needed after import):
```bash
sudo /opt/mariadb/bin/mariadb -u root -p -e "SET GLOBAL log_bin_trust_function_creators = 0;"
```

---

## Step 14 — Configure the Zabbix server and agent

```bash
sudo vi /opt/zabbix/etc/zabbix_server.conf
```
Set:
```ini
DBPassword=Your-Strong-Password-Here
```
(the `zabbix` database user's password from Step 8)

```bash
sudo vi /opt/zabbix/etc/zabbix_agent2.conf
```
Confirm/set:
```ini
Server=127.0.0.1
ServerActive=127.0.0.1
Hostname=MON01
```

Fix ownership so the `zabbix` user (created in Step 9) can read its own configs and write logs/PID files:
```bash
sudo mkdir -p /var/log/zabbix /var/run/zabbix
sudo chown -R zabbix:zabbix /var/log/zabbix /var/run/zabbix /opt/zabbix/etc
```

---

## Step 15 — Create systemd services

Source installs don't ship systemd unit files the way packages do — create them manually:

```bash
sudo tee /etc/systemd/system/zabbix-server.service << 'EOF'
[Unit]
Description=Zabbix Server
After=syslog.target network.target mariadb.service

[Service]
Type=forking
User=zabbix
Group=zabbix
ExecStart=/opt/zabbix/sbin/zabbix_server -c /opt/zabbix/etc/zabbix_server.conf
ExecStop=/bin/kill -SIGTERM $MAINPID
PIDFile=/var/run/zabbix/zabbix_server.pid
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

sudo tee /etc/systemd/system/zabbix-agent2.service << 'EOF'
[Unit]
Description=Zabbix Agent 2
After=syslog.target network.target

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
```

---

## Step 16 — Build Apache and PHP from source, deploy the frontend

**Apache** — same version as [`07`'s Apache build](./07-web01-lamp-nginx-loadbalancer.md#step-9--build-apache-instance-1--php), single instance here (no load balancer in front of MON01, so it listens directly on port 80):
```bash
cd ~
wget https://dlcdn.apache.org/httpd/httpd-2.4.68.tar.gz
wget https://dlcdn.apache.org/apr/apr-1.7.6.tar.gz
wget https://dlcdn.apache.org/apr/apr-util-1.6.3.tar.gz

tar -xzvf httpd-2.4.68.tar.gz
tar -xzvf apr-1.7.6.tar.gz
tar -xzvf apr-util-1.6.3.tar.gz

mv apr-1.7.6 httpd-2.4.68/srclib/apr
mv apr-util-1.6.3 httpd-2.4.68/srclib/apr-util

sudo dnf install -y pcre2-devel openssl-devel expat-devel

cd httpd-2.4.68
./configure \
  --prefix=/opt/apache \
  --with-mpm=prefork \
  --enable-so \
  --enable-unique-id \
  --enable-rewrite \
  --enable-headers \
  --enable-setenvif \
  --enable-logio \
  --enable-expires \
  --enable-ssl \
  --with-ssl \
  --enable-deflate \
  --enable-cache \
  --enable-file-cache \
  --enable-shared \
  --disable-autoindex \
  --disable-asis \
  --disable-cgi \
  --disable-cgid \
  --disable-negotiation \
  --disable-userdir \
  --with-included-apr

make -j$(nproc)
sudo make install
```

**PHP** — same version and dependency list as [`07`'s PHP build](./07-web01-lamp-nginx-loadbalancer.md#step-9--build-apache-instance-1--php):
```bash
cd ~
wget https://www.php.net/distributions/php-8.5.8.tar.gz
tar -xzvf php-8.5.8.tar.gz
cd php-8.5.8

sudo dnf install -y libxml2-devel sqlite-devel oniguruma-devel libcurl-devel \
    libzip-devel gmp-devel libicu-devel freetype-devel libjpeg-turbo-devel \
    libpng-devel libwebp-devel gd-devel gdbm-devel libtidy-devel

./configure \
  --with-apxs2=/opt/apache/bin/apxs \
  --prefix=/opt/apache/php \
  --with-config-file-path=/opt/apache/php \
  --disable-cgi \
  --with-zlib \
  --with-gdbm \
  --enable-soap \
  --with-pear \
  --with-libxml \
  --enable-mbstring \
  --enable-gd \
  --with-freetype \
  --enable-shared \
  --with-jpeg \
  --enable-sockets \
  --with-bz2 \
  --enable-xml \
  --with-gettext \
  --with-gmp \
  --with-iconv \
  --with-pdo-mysql \
  --with-tidy \
  --enable-bcmath \
  --enable-intl \
  --with-libdir=lib64 \
  --with-openssl \
  --with-curl \
  --enable-cli \
  --with-mysqli \
  --with-zip \
  --with-bcmath \
  --with-mysqli=/opt/mariadb/bin/mariadb_config \
  --with-pdo-mysql=/opt/mariadb

make -j$(nproc)
sudo make install
sudo cp php.ini-production /opt/apache/php/php.ini
echo "date.timezone = Asia/Ho_Chi_Minh" | sudo tee -a /opt/apache/php/php.ini
```

**Tune PHP settings the Zabbix frontend prerequisites check expects** — the defaults from `php.ini-production` are too low for a few values Zabbix's setup wizard checks in [Step 18](#step-18--start-services-and-complete-the-frontend-wizard); adjust them now rather than discovering the failures mid-wizard:

```bash
sudo vi /opt/apache/php/php.ini
```
Search for each setting with `/` in `vi` (e.g. `/post_max_size`) and change it to the value below:

| Setting | Default | Change to |
|---|---|---|
| `post_max_size` | `8M` | `16M` |
| `max_execution_time` | `30` | `300` |
| `max_input_time` | `60` | `300` |

Or apply all three non-interactively instead of editing by hand:
```bash
sudo sed -i 's/^post_max_size = .*/post_max_size = 16M/' /opt/apache/php/php.ini
sudo sed -i 's/^max_execution_time = .*/max_execution_time = 300/' /opt/apache/php/php.ini
sudo sed -i 's/^max_input_time = .*/max_input_time = 300/' /opt/apache/php/php.ini

grep -E "^post_max_size|^max_execution_time|^max_input_time" /opt/apache/php/php.ini
```

> `post_max_size` needs to be **larger** than PHP's `upload_max_filesize` (default `2M`, well under `16M`) for consistency — Zabbix's own prerequisites check doesn't test `upload_max_filesize` directly, but a mismatched pair can still cause confusing upload failures later (e.g. importing large templates), so this is worth keeping in mind even though it isn't one of the three settings the wizard flags.

> `--with-mysqli`/`--with-pdo-mysql` point explicitly at `/opt/mariadb` here (unlike WEB01's PHP build, which linked against the system MariaDB client libraries) — since MariaDB itself was also built from source into `/opt/mariadb` in Step 7, PHP needs to be told exactly where to find its client library and `mariadb_config` rather than relying on the system's default library paths, which have nothing installed at this location.

Configure Apache to hand `.php` files to PHP, and deploy the Zabbix frontend as the site's document root:
```bash
sudo tee /opt/apache/conf/extra/httpd-php.conf << 'EOF'
AddHandler application/x-httpd-php .php
DirectoryIndex index.php index.html
EOF
echo 'Include conf/extra/httpd-php.conf' | sudo tee -a /opt/apache/conf/httpd.conf

sudo rm -rf /opt/apache/htdocs
sudo mkdir -p /opt/apache/htdocs
sudo cp -r ~/zabbix-8.0.0/ui/* /opt/apache/htdocs/
sudo mkdir -p /opt/apache/htdocs/conf
sudo chmod 750 /opt/apache/htdocs/conf
```

Create a systemd unit and start:
```bash
sudo tee /etc/systemd/system/apache.service << 'EOF'
[Unit]
Description=Apache HTTP Server (Zabbix frontend)
After=network.target mariadb.service

[Service]
Type=forking
ExecStart=/opt/apache/bin/apachectl start
ExecStop=/opt/apache/bin/apachectl stop
ExecReload=/opt/apache/bin/apachectl graceful
PIDFile=/opt/apache/logs/httpd.pid

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
```

---

## Step 17 — SELinux and firewall

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-port=10051/tcp
sudo firewall-cmd --permanent --add-port=10050/tcp
sudo firewall-cmd --reload
```
`10051/tcp` is the Zabbix trapper port (server receiving active-check data from agents); `10050/tcp` is the agent's own listening port (server polling agents directly, needed on MON01 too since it monitors itself in Step 19).

> **SELinux with an entirely from-source stack:** since MariaDB, Apache, PHP, and Zabbix are all built into custom `/opt/...` paths here rather than installed as packages, none of them run under the SELinux domains (`httpd_t`, `mysqld_t`, a dedicated `zabbix_t`) that the usual `setsebool httpd_can_connect_zabbix`/`httpd_can_network_connect_db` booleans assume — those booleans only affect processes already labeled into those specific domains, which a custom systemd unit executing a binary from `/opt/apache/bin/httpd` typically isn't. In practice this often means the custom binaries run in a more permissive default domain and things "just work" without touching SELinux at all — but if anything unexpectedly fails to start or connect under enforcing mode, check for actual denials before assuming a config mistake:
> ```bash
> sudo ausearch -m avc -ts recent | grep -E "zabbix|httpd|mysqld"
> ```
> Address any real denials with `audit2allow`, or temporarily test with `sudo setenforce 0` to confirm SELinux is actually the blocker before spending time writing a custom policy — then re-enable enforcing mode (`sudo setenforce 1`) and fix the specific denial rather than leaving the system permissive long-term.

---

## Step 18 — Start services and complete the frontend wizard

```bash
sudo systemctl enable --now zabbix-server zabbix-agent2 apache
```

From the host machine (or any VM with network access to MON01), browse to:
```
http://192.168.10.40
```

Work through the setup wizard:
1. **Welcome** → **Next step**.
2. **Check of pre-requisites** — everything should show **OK**. If PHP settings show as failing, revisit Step 15 and confirm `apache` was restarted after any PHP edits (`sudo systemctl restart apache`).
3. **Configure DB connection**: Database type `MySQL`, Host `localhost`, Port `0` (default), Database name `zabbix`, User `zabbix`, Password (from Step 8).
4. **Zabbix server details**: Host `localhost`, Port `10051`, Name `MON01` (or leave blank).
5. **Pre-installation summary** → **Next step**.
6. **Install** → **Finish**.

You'll land on the Zabbix login page.

---

## Step 19 — Enable HTTPS-only access

**Only needed once Wazuh joins this VM in [`13`](./13-mon01-wazuh-manager-configuration.md) — skip this step if that hasn't happened yet.** Wazuh Dashboard listens on port 443 by default, and two separate services can't both bind port 443 on the same IP without a reverse proxy distinguishing them by hostname (SNI) — which this lab doesn't set up. Rather than fight over the standard HTTPS port, give Zabbix its own HTTPS port instead — and since there's no good reason to leave a security-relevant dashboard reachable over unencrypted HTTP anyway, this drops the plain-HTTP listener entirely rather than running both side by side.

Generate a self-signed certificate:
```bash
sudo mkdir -p /opt/apache/conf/ssl
sudo openssl req -x509 -nodes -days 825 \
  -newkey rsa:2048 \
  -keyout /opt/apache/conf/ssl/zabbix.key \
  -out /opt/apache/conf/ssl/zabbix.crt \
  -subj "/C=VN/O=CorpLab/CN=zabbix.corp-lab.com.vn"
```

Enable Apache's SSL module — it was already compiled in via `--enable-ssl --with-ssl` when Apache was built in [Step 16](#step-16--build-apache-and-php-from-source-deploy-the-frontend), but ships commented out by default:
```bash
grep "LoadModule ssl_module" /opt/apache/conf/httpd.conf
```
If that line starts with `#`, uncomment it:
```bash
sudo sed -i 's/#LoadModule ssl_module modules\/mod_ssl.so/LoadModule ssl_module modules\/mod_ssl.so/' /opt/apache/conf/httpd.conf
```

Add an SSL vhost on port 8443:
```bash
sudo tee -a /opt/apache/conf/httpd.conf << 'EOF'
Listen 8443
<VirtualHost *:8443>
    SSLEngine on
    SSLCertificateFile /opt/apache/conf/ssl/zabbix.crt
    SSLCertificateKeyFile /opt/apache/conf/ssl/zabbix.key
    DocumentRoot /opt/apache/htdocs
    <Directory /opt/apache/htdocs>
        AllowOverride None
        Require all granted
        DirectoryIndex index.php
    </Directory>
</VirtualHost>
EOF
```

Disable the plain-HTTP listener from [Step 18](#step-18--start-services-and-complete-the-frontend-wizard) — comment out its `Listen` directive rather than deleting it, so it's easy to find and re-enable later if there's ever a reason to:
```bash
grep -n "^Listen 80$" /opt/apache/conf/httpd.conf
```
Comment out whatever line number that returned (adjust the line number below to match):
```bash
sudo sed -i '52s/^Listen 80$/#Listen 80/' /opt/apache/conf/httpd.conf
```

Test the config before restarting — catches syntax mistakes without taking the frontend down if something's wrong:
```bash
/opt/apache/bin/apachectl configtest
```
Expect `Syntax OK` (a warning about not being able to determine the server's fully qualified domain name is harmless and can be ignored). Then:
```bash
sudo systemctl restart apache
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --permanent --remove-service=http 2>/dev/null
sudo firewall-cmd --reload
```

Verify HTTPS works and HTTP is genuinely gone:
```bash
curl -k -I https://192.168.10.40:8443
curl -I http://192.168.10.40:80
```
Expect `HTTP/1.1 200 OK` from the first command, and a connection failure/refused from the second — confirming port 80 no longer answers at all, not just that it redirects.

From here, Zabbix Frontend is reachable **only** at `https://zabbix.corp-lab.com.vn:8443` (the non-standard port needs to be explicit — the CNAME itself doesn't carry port information), and Wazuh Dashboard stays on the standard `https://wazuh.corp-lab.com.vn` with no port needed, once [`13`](./13-mon01-wazuh-manager-configuration.md) is built. Neither service answers plain HTTP on either VM.

---

## Step 20 — Change the default Admin password

1. Log in with the default credentials: username `Admin`, password `zabbix`.
2. Immediately change it: click the user icon (top right) → **Profile** → **Change password**.
3. Store the new password in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `MON01_10.40` group, entry "Zabbix Frontend admin".

---

## Step 21 — Add MON01 as the first monitored host

Full agent rollout to every other VM in this lab happens in [`17-agent-deployment-all-vms.md`](./17-agent-deployment-all-vms.md) — this step just gets one working example end-to-end using the agent already built on MON01 itself in Step 11.

1. **Data collection → Hosts → Create host**.
2. **Host name**: `MON01`.
3. **Host groups**: create a new group `Monitoring Servers`.
4. **Interfaces**: click **Add** next to Agent → IP address `127.0.0.1` (or `192.168.10.40`), port `10050`.
5. **Templates** tab → **Add** → search for and select **Linux by Zabbix agent**.
6. **Add**.

Confirm the agent is reachable — after a minute or two, **Data collection → Hosts** should show a green "ZBX" icon next to `MON01`, not red.

---

## Step 22 — Create triggers of different types

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

## Step 23 — Build a basic dashboard

1. **Dashboards → All dashboards → Create dashboard**.
2. **Name**: `Infrastructure Overview`.
3. **Add widget** → type **Graph (classic)** → select host `MON01`, item `CPU utilization`.
4. **Add** another widget → type **Problems** → leave default filters (shows any active problems/triggers across all hosts).
5. **Add** another widget → type **Host availability** → shows a quick up/down summary, useful once more hosts are added in [`17`](./17-agent-deployment-all-vms.md).
6. **Apply** / **Save changes**.

---

## Step 24 — Final verification checklist

1. **Zabbix server running:**
```bash
sudo systemctl status zabbix-server zabbix-agent2
```

2. **Frontend reachable:**
```bash
curl -I http://192.168.10.40
```

3. **Database connection healthy** — check the server log for connection errors:
```bash
sudo tail -20 /var/log/zabbix/zabbix_server.log
```

4. **MON01 host shows green (reachable) in Data collection → Hosts.**

5. **Triggers were created** (repeat the checks from [Step 22](#step-22--create-triggers-of-different-types)).

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
