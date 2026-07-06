# 07 — WEB01: LAMP stack behind an Nginx load balancer

This document clones the Rocky Linux 10 [Golden Baseline](./05-golden-baseline-rocky-linux-10.md) into **WEB01** and builds it into a self-load-balancing web tier: two independent Apache + PHP instances on the same host, MariaDB, and Nginx in front acting as reverse proxy, SSL terminator, and load balancer between the two Apache instances (see [`README.md`](../README.md#virtual-machine-inventory) for why this design was chosen over a second dedicated web VM).

All software versions below were current as of this document's last revision — confirm you're not missing a newer patch release before downloading.

| Component | Version used | Source |
|---|---|---|
| MariaDB | 12.3.2 (LTS) | https://mariadb.org/download/ |
| Apache HTTP Server | 2.4.68 | https://httpd.apache.org/download.cgi |
| APR | 1.7.6 | https://apr.apache.org/download.cgi |
| APR-Util | 1.6.3 | https://apr.apache.org/download.cgi |
| PHP | 8.5.8 | https://www.php.net/downloads |
| phpMyAdmin | 5.2.3 | https://www.phpmyadmin.net/downloads/ |
| Nginx | 1.30.3 (stable) | https://nginx.org/en/download.html |
| OpenSSL | 3.6.3 | https://openssl.org/source/ |
| zlib | 1.3.2 | https://zlib.net/ |
| headers-more-nginx-module | 0.40 | https://github.com/openresty/headers-more-nginx-module/archive/refs/tags/v0.40.tar.gz |

---

## Table of contents

- [VM specification](#vm-specification)
- [Step 1 — Clone the golden baseline and provision disks](#step-1--clone-the-golden-baseline-and-provision-disks)
- [Step 2 — Set hostname and static IP](#step-2--set-hostname-and-static-ip)
- [Step 3 — Grow the root filesystem](#step-3--grow-the-root-filesystem)
- [Step 4 — Create swap](#step-4--create-swap)
- [Step 5 — LVM demo on the second disk](#step-5--lvm-demo-on-the-second-disk)
- [Step 6 — NFS server setup](#step-6--nfs-server-setup)
- [Step 7 — Domain-join to corp-lab.com.vn](#step-7--domain-join-to-corp-labcomvn)
- [Step 8 — Build MariaDB from source](#step-8--build-mariadb-from-source)
- [Step 9 — Build Apache instance 1 + PHP](#step-9--build-apache-instance-1--php)
- [Step 10 — Build Apache instance 2 + PHP](#step-10--build-apache-instance-2--php)
- [Step 11 — Build Nginx from source](#step-11--build-nginx-from-source)
- [Step 12 — Configure Nginx as reverse proxy and load balancer](#step-12--configure-nginx-as-reverse-proxy-and-load-balancer)
- [Step 13 — SSL/TLS on Nginx](#step-13--ssltls-on-nginx)
- [Step 14 — Install phpMyAdmin](#step-14--install-phpmyadmin)
- [Step 15 — LogRotate](#step-15--logrotate)
- [Step 16 — Final verification checklist](#step-16--final-verification-checklist)
- [Next step](#next-step)

---

## VM specification

| Setting | Value |
|---|---|
| VM name (VMware Library) | `WEB01_10.21` |
| OS hostname | `WEB01` |
| vCPU | 4–6 |
| RAM | 8 GB |
| Disk 1 (OS + swap) | 50 GB (30 GB OS + 20 GB swap) |
| Disk 2 (data) | 20 GB — LVM demo |
| NIC 1 (NAT) | DHCP |
| NIC 2 (Host-only) | Static — `192.168.10.21` |

---

## Step 1 — Clone the golden baseline and provision disks

Follow [`05-golden-baseline-rocky-linux-10.md`'s cloning instructions](./05-golden-baseline-rocky-linux-10.md#cloning-this-baseline-later):

1. Right-click `GoldenBaseline-Rocky10` → **Manage → Clone…** → **Create a full clone**.
> <img width="323" height="265" alt="image" src="https://github.com/user-attachments/assets/ca69123c-d2a2-4ed9-b277-038a0c3dc4d7" />
2. Name the clone `WEB01_10.21`.
> <img width="323" height="265" alt="image" src="https://github.com/user-attachments/assets/0f29714a-6a6b-4f76-839d-40bec4941801" />
3. Adjust vCPU/RAM in **VM → Settings** per the [VM specification](#vm-specification) above.
4. With the VM still powered off, expand the OS disk to **50 GB**: **VM → Settings → Hard Disk → Utilities → Expand…**.
5. Add the second disk while still powered off: **VM → Settings → Add… → Hard Disk → Create a new virtual disk**, size **20 GB**.
> <img width="566" height="533" alt="image" src="https://github.com/user-attachments/assets/9918acb1-56b7-42cc-8f5c-42996c7e7183" />
6. Power on the VM and reconnect via PuTTY using its NAT-assigned IP (`ip a`), same approach as [`05`'s Step 2](./05-golden-baseline-rocky-linux-10.md#step-2--enable-ssh-and-switch-to-putty) — PuTTY will prompt to accept a new host key fingerprint since this is a fresh clone, which is expected.

---

## Step 2 — Set hostname and static IP

```bash
sudo hostnamectl set-hostname WEB01
```

Configure NIC 2 (Host-only) with a static IP using `nmtui`:

```bash
sudo nmtui
```
Select the connection tied to the Host-only NIC → **Edit** → IPv4 → **Manual** → address `192.168.10.21/24`, no gateway (per the [dual-interface design](./02-network-architecture-planning.md#design-overview)), DNS `192.168.10.10`.
> <img width="479" height="263" alt="image" src="https://github.com/user-attachments/assets/8e0f94c1-0efd-43be-a736-398886360071" />

Or via `nmcli`:
```bash
sudo nmcli con mod "Wired connection 1" ipv4.addresses 192.168.10.21/24
sudo nmcli con mod "Wired connection 1" ipv4.dns 192.168.10.10
sudo nmcli con mod "Wired connection 1" ipv4.method manual
sudo nmcli con up "Wired connection 1"
```

**Stop NIC 1 (NAT) from polluting DNS resolution** — required before domain-join in Step 7 will work. See the full explanation in [`02-network-architecture-planning.md`](./02-network-architecture-planning.md#rocky-linux-nic-2--host-only):
 
```bash
nmcli con show
# identify NIC 1's connection name (commonly matches the device name, e.g. "ens160")
 
sudo nmcli con modify "ens160" ipv4.ignore-auto-dns yes
sudo nmcli con up "ens160"
 
cat /etc/resolv.conf
# confirm only 192.168.10.10 remains
```
> <img width="239" height="32" alt="image" src="https://github.com/user-attachments/assets/5f142fc3-e536-4a44-bf6f-48eb5bac3761" />

Reconnect PuTTY using the new static IP from here on.

---

## Step 3 — Grow the root filesystem
 
The golden baseline's OS disk was 20 GB; it was expanded to 50 GB in Step 1. Grow the LVM volume and filesystem to use the new space, per [`05`'s expansion procedure](./05-golden-baseline-rocky-linux-10.md#cloning-this-baseline-later).
 
Install `growpart` first — not present by default on Rocky Linux:
```bash
sudo dnf install -y cloud-utils-growpart
```
 
Identify the actual disk device, partition number, and LVM volume group/logical volume names — **do not assume `/dev/sda` or a volume group named `rl`**. Depending on which virtual disk controller was selected when the VM was created, VMware Workstation may present the disk as SATA/SCSI (`/dev/sda`, partitions `/dev/sda1`/`2`/`3`) or NVMe (`/dev/nvme0n1`, partitions `/dev/nvme0n1p1`/`p2`/`p3` — note the `p` before the partition number):
 
```bash
lsblk
sudo vgs
sudo lvs
```
> <img width="489" height="155" alt="image" src="https://github.com/user-attachments/assets/95d4eb26-8081-40ba-8cd1-8d301d5eb1ac" />
 
Then grow the partition, physical volume, logical volume, and filesystem using the **actual names** confirmed above (3 for UEFI layouts, 2 for BIOS):
 
```bash
# Example for a SATA/SCSI disk (/dev/sda)
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3
 
# Example for an NVMe disk (/dev/nvme0n1)
sudo growpart /dev/nvme0n1 3
sudo pvresize /dev/nvme0n1p3
 
# Replace vg_name-lv_name with the actual /dev/mapper/ path from `lvs` (e.g. rl-root or rlm-root)
sudo lvextend -l +100%FREE /dev/mapper/vg_name-lv_name
sudo xfs_growfs /
```
> <img width="517" height="69" alt="image" src="https://github.com/user-attachments/assets/84692010-1143-439e-aeaa-358e1183de00" />
> <img width="517" height="209" alt="image" src="https://github.com/user-attachments/assets/674c5eb2-9ba5-4ceb-b86f-dbd1eef598b3" />

 
> **Reminder:** this expansion must only be done while the VM was powered off during Step 1, then grown from inside the running guest afterward — never resize a disk while the VM is powered on, and never force-power-off mid-resize. See the [XFS corruption note](./05-golden-baseline-rocky-linux-10.md#cloning-this-baseline-later) in the golden baseline document for what goes wrong if this order is violated.
 
Confirm the new size:
```bash
df -h /
```
> <img width="332" height="35" alt="image" src="https://github.com/user-attachments/assets/3bea1c62-209a-4a16-95bb-a7203e900afd" />
 
---
 
## Step 4 — Create swap
 
Swap is sized at 2× RAM (8 GB RAM → 20 GB swap, rounded to the nearest 10 GB per this guide's convention):
 
```bash
sudo fallocate -l 20G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
> <img width="452" height="91" alt="image" src="https://github.com/user-attachments/assets/d9317b78-11fb-450f-8547-572f0cf24bcb" />

Verify:
```bash
swapon --show
free -h
```
> <img width="486" height="78" alt="image" src="https://github.com/user-attachments/assets/3fd6df87-e243-4f9b-911c-9ac620ce7863" />
 
WEB01 does not run JVM-based services, so the default `vm.swappiness` (60) is left unchanged — the `swappiness=1` tuning used later for MON01/ELK01/LOG02 is specific to avoiding JVM garbage-collection stalls and doesn't apply here.
 
---
 
## Step 5 — LVM demo on the second disk
 
Using the 20 GB disk added in Step 1:
 
```bash
lsblk
```
> <img width="301" height="103" alt="image" src="https://github.com/user-attachments/assets/80b6e631-124c-4b1e-bf81-c37682e7e6c4" />

Confirm the new disk shows up with no existing partitions — it will be named `/dev/sdb` on a SATA/SCSI controller, or `/dev/nvme1n1` on an NVMe controller (the first NVMe disk is `nvme0n1`, so a second one is `nvme0n2`). Use the actual name from `lsblk` in place of `/dev/nvme0n2 ` below.
 
```bash
sudo pvcreate /dev/nvme0n2 
sudo vgcreate vg_data /dev/nvme0n2 
sudo lvcreate -L 15G -n lv_web vg_data
sudo mkfs.xfs /dev/vg_data/lv_web
 
sudo mkdir -p /mnt/webdata
echo '/dev/mapper/vg_data-lv_web /mnt/webdata xfs defaults 0 0' | sudo tee -a /etc/fstab
sudo mount -a
```
 
Verify:
```bash
df -h /mnt/webdata
```
> <img width="511" height="441" alt="image" src="https://github.com/user-attachments/assets/86b751f2-ca72-49b5-8244-3b22cf4b3d97" />
 
This volume isn't used by any specific service in this guide — it stands as a self-contained demonstration of LVM (physical volume → volume group → logical volume → filesystem → mount), separate from the OS disk.
 
---
 
## Step 6 — NFS server setup
 
```bash
sudo dnf install -y nfs-utils
sudo systemctl enable --now nfs-server
 
sudo mkdir -p /mnt/webdata/nfsshare
sudo chmod 755 /mnt/webdata/nfsshare
echo '/mnt/webdata/nfsshare 192.168.10.0/24(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra
 
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload
```
 
Verify:
```bash
showmount -e localhost
```
> <img width="236" height="35" alt="image" src="https://github.com/user-attachments/assets/16f0793d-f2bd-42bc-ba2b-33736e7f411e" />
 
---
 
## Step 7 — Domain-join to corp-lab.com.vn
 
```bash
sudo dnf install -y realmd sssd-common sssd-ad adcli krb5-workstation samba-common-tools oddjob oddjob-mkhomedir
 
sudo realm discover corp-lab.com.vn
sudo realm join -U administrator corp-lab.com.vn
```
Enter the `CORP-LAB\Administrator` password from [KeePass](./03-remote-access-tooling-setup.md#keepass) when prompted.
> <img width="361" height="176" alt="image" src="https://github.com/user-attachments/assets/4f2bb4ec-34ec-431d-9eea-322dc236116b" />

Verify:
```bash
realm list
id administrator@corp-lab.com.vn
```
> <img width="1034" height="220" alt="image" src="https://github.com/user-attachments/assets/e78189f4-cf64-4292-b108-6f3522f2864d" />

---
 
## Step 8 — Build MariaDB from source
 
```bash
cd ~
wget https://downloads.mariadb.org/rest-api/mariadb/12.3.2/mariadb-12.3.2.tar.gz
tar -xzvf mariadb-12.3.2.tar.gz
cd mariadb-12.3.2
 
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y cmake ncurses-devel bison libxml2-devel gnutls-devel \
    openssl-devel libaio-devel
 
mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/opt/mariadb -DMYSQL_DATADIR=/opt/mariadb/data
make -j$(nproc)
sudo make install
 
sudo useradd -r -s /sbin/nologin mysql
sudo mkdir -p /opt/mariadb/data
sudo chown -R mysql:mysql /opt/mariadb
sudo /opt/mariadb/scripts/mariadb-install-db --user=mysql --datadir=/opt/mariadb/data --basedir=/opt/mariadb
```
> <img width="754" height="365" alt="image" src="https://github.com/user-attachments/assets/2158c716-dc02-4533-a972-1d6f4fce3dbe" />

 
Create a systemd unit:
```bash
sudo tee /etc/systemd/system/mariadb.service << 'EOF'
[Unit]
Description=MariaDB database server
After=network.target
 
[Service]
User=mysql
Group=mysql
ExecStart=/opt/mariadb/bin/mariadbd --datadir=/opt/mariadb/data --basedir=/opt/mariadb
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
 
---
 
## Step 9 — Build Apache instance 1 + PHP
 
**Apache:**
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
  --prefix=/opt/apache1 \
  --with-mpm=prefork \
  --enable-so \
  --enable-unique-id \
  --enable-rewrite \
  --enable-headers \
  --enable-setenvif \
  --enable-logio \
  --enable-expires \
  --enable-proxy \
  --enable-proxy-http \
  --enable-proxy-balancer \
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
 
Set Apache instance 1 to listen on port **8080** (internal only, since Nginx fronts it):
```bash
sudo sed -i 's/^Listen 80/Listen 127.0.0.1:8080/' /opt/apache1/conf/httpd.conf
```
 
**PHP:**
```bash
cd ~
wget https://www.php.net/distributions/php-8.5.8.tar.gz
tar -xzvf php-8.5.8.tar.gz
cd php-8.5.8
 
sudo dnf install -y libxml2-devel sqlite-devel oniguruma-devel libcurl-devel \
    libzip-devel gmp-devel libicu-devel freetype-devel libjpeg-turbo-devel \
    libpng-devel libwebp-devel gd-devel gdbm-devel libtidy-devel
 
./configure \
  --with-apxs2=/opt/apache1/bin/apxs \
  --prefix=/opt/apache1/php \
  --with-config-file-path=/opt/apache1/php \
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
  --with-zip
 
make -j$(nproc)
sudo make install
sudo cp php.ini-production /opt/apache1/php/php.ini
echo "date.timezone = Asia/Ho_Chi_Minh" | sudo tee -a /opt/apache1/php/php.ini
```
 
Configure Apache to hand `.php` files to PHP:
```bash
sudo tee /opt/apache1/conf/extra/httpd-php.conf << 'EOF'
AddHandler application/x-httpd-php .php
DirectoryIndex index.php index.html
EOF
echo 'Include conf/extra/httpd-php.conf' | sudo tee -a /opt/apache1/conf/httpd.conf
```
 
Create a systemd unit and start:
```bash
sudo tee /etc/systemd/system/apache1.service << 'EOF'
[Unit]
Description=Apache HTTP Server instance 1
After=network.target
 
[Service]
Type=forking
ExecStart=/opt/apache1/bin/apachectl start
ExecStop=/opt/apache1/bin/apachectl stop
ExecReload=/opt/apache1/bin/apachectl graceful
PIDFile=/opt/apache1/logs/httpd.pid
 
[Install]
WantedBy=multi-user.target
EOF
 
sudo systemctl daemon-reload
sudo systemctl enable --now apache1
```
 
Test:
```bash
echo "<?php phpinfo(); ?>" | sudo tee /opt/apache1/htdocs/info.php
curl http://127.0.0.1:8080/info.php | head -20
```
 
---
 
## Step 10 — Build Apache instance 2 + PHP
 
Identical process, second instance, listening on port **8443**:
 
```bash
cd ~/httpd-2.4.68
make clean
 
./configure \
  --prefix=/opt/apache2 \
  --with-mpm=prefork \
  --enable-so \
  --enable-unique-id \
  --enable-rewrite \
  --enable-headers \
  --enable-setenvif \
  --enable-logio \
  --enable-expires \
  --enable-proxy \
  --enable-proxy-http \
  --enable-proxy-balancer \
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
sudo sed -i 's/^Listen 80/Listen 127.0.0.1:8443/' /opt/apache2/conf/httpd.conf
 
cd ~/php-8.5.8
make clean
./configure \
  --with-apxs2=/opt/apache2/bin/apxs \
  --prefix=/opt/apache2/php \
  --with-config-file-path=/opt/apache2/php \
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
  --with-zip
 
make -j$(nproc)
sudo make install
sudo cp php.ini-production /opt/apache2/php/php.ini
echo "date.timezone = Asia/Ho_Chi_Minh" | sudo tee -a /opt/apache2/php/php.ini
 
sudo tee /opt/apache2/conf/extra/httpd-php.conf << 'EOF'
AddHandler application/x-httpd-php .php
DirectoryIndex index.php index.html
EOF
echo 'Include conf/extra/httpd-php.conf' | sudo tee -a /opt/apache2/conf/httpd.conf
 
sudo tee /etc/systemd/system/apache2.service << 'EOF'
[Unit]
Description=Apache HTTP Server instance 2
After=network.target
 
[Service]
Type=forking
ExecStart=/opt/apache2/bin/apachectl start
ExecStop=/opt/apache2/bin/apachectl stop
ExecReload=/opt/apache2/bin/apachectl graceful
PIDFile=/opt/apache2/logs/httpd.pid
 
[Install]
WantedBy=multi-user.target
EOF
 
sudo systemctl daemon-reload
sudo systemctl enable --now apache2
```
 
Test:
```bash
echo "<?php phpinfo(); ?>" | sudo tee /opt/apache2/htdocs/info.php
curl http://127.0.0.1:8443/info.php | head -20
```
 
> Both instances share the same MariaDB (built in Step 8) — connect either one to it identically via `pdo_mysql`/`mysqli` once you deploy an application later.
 
---
 
## Step 11 — Build Nginx from source
 
```bash
cd ~
wget https://nginx.org/download/nginx-1.30.3.tar.gz
wget https://openssl.org/source/openssl-3.6.3.tar.gz
wget https://zlib.net/zlib-1.3.2.tar.gz
wget https://github.com/openresty/headers-more-nginx-module/archive/refs/tags/v0.40.tar.gz -O headers-more-nginx-module-0.40.tar.gz
 
tar -xzvf nginx-1.30.3.tar.gz
tar -xzvf openssl-3.6.3.tar.gz
tar -xzvf zlib-1.3.2.tar.gz
tar -xzvf headers-more-nginx-module-0.40.tar.gz
 
sudo dnf install -y pcre2-devel gcc make perl-FindBin perl-IPC-Cmd perl-devel perl-Time-Piece
```
> `perl-FindBin`, `perl-IPC-Cmd`, and `perl-Time-Piece` are all required by OpenSSL's `Configure`/`Makefile.in` generation, which `nginx`'s build process invokes automatically when building OpenSSL statically via `--with-openssl=../openssl-3.6.3`. Missing modules surface one at a time across separate build attempts (`Can't locate FindBin.pm`, then later `Can't locate Time/Piece.pm`) rather than all at once — install all three together before running `./configure` below to avoid repeating the build multiple times.
 
```bash
sudo useradd --system --no-create-home --shell /sbin/nologin nginx
 
cd nginx-1.30.3
./configure \
  --prefix=/opt/nginx \
  --sbin-path=/opt/nginx/sbin/nginx \
  --conf-path=/opt/nginx/conf/nginx.conf \
  --error-log-path=/opt/nginx/logs/error.log \
  --http-log-path=/opt/nginx/logs/access.log \
  --pid-path=/opt/nginx/logs/nginx.pid \
  --lock-path=/opt/nginx/nginx.lock \
  --user=nginx \
  --group=nginx \
  --with-http_stub_status_module \
  --with-http_ssl_module \
  --add-dynamic-module=../headers-more-nginx-module-0.40 \
  --with-zlib=../zlib-1.3.2 \
  --with-openssl=../openssl-3.6.3
 
make -j$(nproc)
sudo make install
 
sudo mkdir -p /opt/nginx/conf/ssl /opt/nginx/sites-enabled
```
 
Create a systemd unit:
```bash
sudo tee /etc/systemd/system/nginx.service << 'EOF'
[Unit]
Description=nginx HTTP and reverse proxy server
After=network.target
 
[Service]
Type=forking
PIDFile=/opt/nginx/logs/nginx.pid
ExecStartPre=/opt/nginx/sbin/nginx -t
ExecStart=/opt/nginx/sbin/nginx
ExecReload=/opt/nginx/sbin/nginx -s reload
ExecStop=/opt/nginx/sbin/nginx -s quit
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
EOF
 
sudo systemctl daemon-reload
```
 
**Open the firewall for HTTP and HTTPS** — without this, Nginx only answers requests from WEB01 itself (`localhost`/`127.0.0.1`); every other VM on `192.168.10.0/24` would be refused at the firewall before ever reaching Nginx:
 
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```
Confirm `http` and `https` both appear in the output.
 
---
 
## Step 12 — Configure Nginx as reverse proxy and load balancer
 
```bash
sudo tee /opt/nginx/sites-enabled/web-cluster.conf << 'EOF'
upstream apache_backend {
    server 127.0.0.1:8080;
    server 127.0.0.1:8443;
}
 
server {
    listen 80;
    server_name web01.corp-lab.com.vn;
 
    more_clear_headers Server;
 
    location / {
        proxy_pass http://apache_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```
 
> `load_module` is a main-context-only directive — it cannot appear inside a file reached via `include sites-enabled/*.conf;` (which is included from *inside* the `http {}` block). It's already placed correctly at the very top of `nginx.conf` above, before `worker_processes`; don't add another copy of it here or `nginx -t` will fail with a context error, and the systemd unit's `ExecStartPre=/opt/nginx/sbin/nginx -t` check will silently prevent Nginx from ever actually starting.
 
**Remove nginx's default sample server block** — the stock `/opt/nginx/conf/nginx.conf` generated by `make install` ships with a built-in example `server { listen 80; server_name localhost; location / { root html; ... } }` block. Since that block's `server_name` matches `localhost` exactly, any test request using `Host: localhost` (e.g. `curl http://localhost/...`) is routed there instead of to the reverse-proxy block just added above — silently returning nginx's own static `404`/welcome page rather than proxying to Apache, with no error to indicate anything is misconfigured.
 
Rather than manually commenting out individual lines of the stock config (easy to mismatch a `{`/`}` pair and break the file — for example leaving one stray closing brace uncommented, which silently closes the `http {}` block early and produces a confusing `"error_page" directive is not allowed here` error), overwrite `nginx.conf` cleanly from scratch:
 
```bash
sudo tee /opt/nginx/conf/nginx.conf << 'EOF'
load_module modules/ngx_http_headers_more_filter_module.so;
 
worker_processes  auto;
 
events {
    worker_connections  1024;
}
 
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
 
    include /opt/nginx/sites-enabled/*.conf;
}
EOF
```
 
> **Use an absolute path for `include /opt/nginx/sites-enabled/*.conf;`, not a relative one.** A relative path like `include sites-enabled/*.conf;` is resolved by nginx relative to the directory containing `nginx.conf` itself (`/opt/nginx/conf/`) — not relative to `--prefix` (`/opt/nginx/`) as it might seem. Since the `sites-enabled` folder created earlier in this guide lives directly under `/opt/nginx/`, a relative include silently matches zero files (no error from `nginx -t` — an empty glob is valid syntax) and every site config placed there is quietly never loaded. Verify the include actually picked up a file with `sudo /opt/nginx/sbin/nginx -T | grep "configuration file"` — it should list `web-cluster.conf` alongside `nginx.conf` and `mime.types`.
 
Test and start:
```bash
sudo /opt/nginx/sbin/nginx -t
sudo systemctl enable --now nginx
```
 
Verify load balancing — stop `apache2` temporarily and confirm Nginx still serves traffic from `apache1` alone, then restart it:
```bash
sudo systemctl stop apache2
curl -I http://localhost/info.php
sudo systemctl start apache2
```
 
**Test from another VM** — everything so far has only confirmed WEB01 can reach itself. From a different machine on `192.168.10.0/24` (e.g. DC01, or the host machine if it has a route to VMnet1), confirm the site is actually reachable over the network using WEB01's real IP rather than `localhost`:
 
- From a Linux VM (e.g. once MON01 exists later):
```bash
curl -I http://192.168.10.21
```
- From a Windows VM (e.g. DC01), PowerShell:
```powershell
Invoke-WebRequest -Uri http://192.168.10.21 -UseBasicParsing | Select-Object StatusCode
```
- From the host machine's own browser (if the host has connectivity to VMnet1 — typically true since it's the Host-only network): browse to `http://192.168.10.21`.
If any of these time out or get refused while the local `curl http://localhost` in this same step succeeds, the firewall rule above is the most likely cause — re-run `sudo firewall-cmd --list-services` and confirm `http`/`https` are present.
 
---
 
## Step 13 — SSL/TLS on Nginx
 
Generate a self-signed certificate for this lab:
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/nginx/conf/ssl/web01.key \
  -out /opt/nginx/conf/ssl/web01.crt \
  -subj "/C=VN/ST=HoChiMinh/L=HoChiMinh/O=CorpLab/CN=web01.corp-lab.com.vn"
```
 
Add HTTPS server block:
```bash
sudo tee -a /opt/nginx/sites-enabled/web-cluster.conf << 'EOF'
 
server {
    listen 443 ssl;
    server_name web01.corp-lab.com.vn;
 
    ssl_certificate     /opt/nginx/conf/ssl/web01.crt;
    ssl_certificate_key /opt/nginx/conf/ssl/web01.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
 
    more_clear_headers Server;
 
    location / {
        proxy_pass http://apache_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
 
sudo /opt/nginx/sbin/nginx -t
sudo systemctl reload nginx
```
 
Verify:
```bash
curl -Ik https://localhost
```
 
---
 
## Step 14 — Install phpMyAdmin
 
```bash
cd ~
wget https://www.phpmyadmin.net/downloads/phpMyAdmin-5.2.3-all-languages.tar.gz
tar -xzvf phpMyAdmin-5.2.3-all-languages.tar.gz
sudo mkdir -p /var/www
sudo mv phpMyAdmin-5.2.3-all-languages /var/www/phpMyAdmin
sudo chown -R nginx:nginx /var/www/phpMyAdmin
 
cd /var/www/phpMyAdmin
cp config.sample.inc.php config.inc.php
sed -i "s/\$cfg\['Servers'\]\[\$i\]\['host'\] = 'localhost';/\$cfg['Servers'][\$i]['host'] = '127.0.0.1';/" config.inc.php
```
 
Point Apache instance 1 at it:
```bash
sudo tee /opt/apache1/conf/extra/httpd-phpmyadmin.conf << 'EOF'
Alias /phpmyadmin /var/www/phpMyAdmin
 
<Directory /var/www/phpMyAdmin>
    DirectoryIndex index.php
    AllowOverride All
    Require all granted
</Directory>
EOF
echo 'Include conf/extra/httpd-phpmyadmin.conf' | sudo tee -a /opt/apache1/conf/httpd.conf
sudo systemctl restart apache1
```
 
Add a route through Nginx so phpMyAdmin is reachable externally the same way as the rest of the site (through the reverse proxy on ports 80/443, not directly against Apache's internal port):
 
```bash
sudo sed -i '/location \/ {/i\    location /phpmyadmin/ {\n        proxy_pass http://apache_backend;\n        proxy_set_header Host $host;\n        proxy_set_header X-Real-IP $remote_addr;\n        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;\n        proxy_set_header X-Forwarded-Proto $scheme;\n    }\n' /opt/nginx/sites-enabled/web-cluster.conf
 
sudo /opt/nginx/sbin/nginx -t
sudo systemctl reload nginx
```
 
Verify from another machine on the network (not just WEB01 itself):
```
http://192.168.10.21/phpmyadmin
https://192.168.10.21/phpmyadmin
```
 
---
 
## Step 15 — LogRotate
 
```bash
sudo mkdir -p /opt/script/logrotate
 
sudo tee /etc/logrotate.d/web01-stack << 'EOF'
/opt/nginx/logs/*.log {
    weekly
    notifempty
    missingok
    rotate 200
    compress
    create 644 nginx root
    dateext
    dateformat -%Y%m%d
    postrotate
        [ -f /opt/nginx/logs/nginx.pid ] && kill -USR1 `cat /opt/nginx/logs/nginx.pid`
    endscript
}
 
/opt/apache1/logs/*_log /opt/apache2/logs/*_log {
    weekly
    notifempty
    missingok
    rotate 200
    compress
    create 644 root root
    dateext
    dateformat -%Y%m%d
    sharedscripts
    postrotate
        /opt/apache1/bin/apachectl graceful > /dev/null 2>/dev/null || true
        /opt/apache2/bin/apachectl graceful > /dev/null 2>/dev/null || true
    endscript
}
EOF
```
 
Test the rotation config without waiting for the schedule:
```bash
sudo logrotate -d /etc/logrotate.d/web01-stack
```
 
---
 
## Step 16 — Final verification checklist
 
1. **Both Apache instances respond independently:**
```bash
curl -I http://127.0.0.1:8080
curl -I http://127.0.0.1:8443
```
 
2. **Website is reachable from other VMs, not just localhost** — from any other machine on `192.168.10.0/24` (or the host machine):
```bash
curl -I http://192.168.10.21
curl -Ik https://192.168.10.21
```
Both must succeed from an **external** machine, not just from within WEB01 itself — this is the check that actually confirms the firewall (`http`/`https` services opened in Step 11) and Nginx are correctly serving the rest of the network, not only `localhost`.
 
3. **Nginx load balances across both:** repeat a request several times and confirm both instances' access logs show hits:
```bash
for i in {1..10}; do curl -s http://localhost/info.php > /dev/null; done
tail -5 /opt/apache1/logs/access_log
tail -5 /opt/apache2/logs/access_log
```
 
4. **SSL is functioning:**
```bash
curl -Ik https://localhost
```
 
5. **MariaDB is running and reachable:**
```bash
sudo systemctl status mariadb
/opt/mariadb/bin/mariadb -u root -p -e "SELECT VERSION();"
```
 
6. **Swap is active and correctly sized:**
```bash
free -h
```
 
7. **LVM volume is mounted:**
```bash
df -h /mnt/webdata
```
 
8. **NFS export is visible:**
```bash
showmount -e localhost
```
 
9. **Domain-join succeeded:**
```bash
realm list
```
 
10. **Credentials are stored, not memorized** — confirm the local sudo user password and MariaDB root password are saved in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `WEB01_10.21` group.
If all ten checks pass, WEB01 is ready.
 
---
 
## Next step
 
Continue to [`08-web01-suricata-nids.md`](./08-web01-suricata-nids.md) to add Suricata as a network intrusion detection layer on this same VM.
