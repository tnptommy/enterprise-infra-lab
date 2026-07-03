# 02 — Network architecture planning

This guide configures the two VMware virtual networks every VM in this lab will use, and documents the IP/naming conventions followed throughout the rest of the build.

Do this once, before creating any VM — both Golden Baseline templates ([`04`](./04-golden-baseline-windows-server-2025.md) and [`05`](./05-golden-baseline-rocky-linux-10.md)) and every VM cloned from them depend on this network already being in place.

---

## Table of contents

- [Design overview](#design-overview)
- [Configuring the Virtual Network Editor (Windows host)](#configuring-the-virtual-network-editor-windows-host)
- [Configuring the Virtual Network Editor (Linux host)](#configuring-the-virtual-network-editor-linux-host)
- [Per-VM dual-NIC configuration](#per-vm-dual-nic-configuration)
- [IP allocation table](#ip-allocation-table)
- [VM and hostname naming convention](#vm-and-hostname-naming-convention)
- [Static IP configuration reference](#static-ip-configuration-reference)
- [Verifying connectivity](#verifying-connectivity)
- [Next step](#next-step)

---

## Design overview

Every VM uses **two virtual network adapters**:

| Adapter | VMware network | Subnet | Purpose |
|---|---|---|---|
| NIC 1 | NAT (VMnet8) | VMware's default NAT subnet (unchanged) | Outbound internet access — package updates, source downloads, KMS activation |
| NIC 2 | Host-only (VMnet1) | `192.168.10.0/24` (re-addressed from VMware's default) | Internal lab traffic — inter-VM communication, domain join, DNS, monitoring, all static IPs |

**Why two adapters instead of one:**
- NAT alone would work for internet access, but re-addressing NAT's subnet risks breaking the host's own outbound connectivity conventions and makes it harder to reason about "internal lab traffic" vs "internet traffic" separately.
- Host-only alone would give a stable internal subnet but no VM would be able to reach the internet for updates, package downloads, or KMS activation.
- Two adapters cleanly separates the two concerns: NIC 1 always just works for internet access and needs no attention after this guide; NIC 2 is where every static IP, hostname, and firewall rule in the rest of this guide actually applies.

> You may substitute your own internal subnet — `192.168.10.0/24` is simply what this guide uses consistently across every document. If you change it, every static IP reference in this guide (and in your own VM configuration) needs to change accordingly.

---

## Configuring the Virtual Network Editor (Windows host)

1. Open VMware Workstation.
2. Go to **Edit → Virtual Network Editor**.
3. Click **Change Settings** (requires Administrator elevation — accept the UAC prompt).
4. Select **VMnet1 (Host-only)** from the list.
> <img width="443" height="385" alt="image" src="https://github.com/user-attachments/assets/10c40043-f4e5-4e97-951f-1b3de1bb2972" />
5. Set:
   - **Subnet IP:** `192.168.10.0`
   - **Subnet mask:** `255.255.255.0`
6. Uncheck **"Use local DHCP service to distribute IP address to VMs"** — every VM on this network uses a static IP (except CLIENT01, which is documented separately in [`11-client01-domain-join-wsus-verification.md`](./11-client01-domain-join-wsus-verification.md) as receiving DHCP from DC01 itself, not from VMware).
7. Leave **VMnet8 (NAT)** untouched — its default subnet is fine, and its own DHCP service should remain enabled since NIC 1 on every VM is DHCP-assigned.
8. Click **Apply**, then **OK**.

---

## Configuring the Virtual Network Editor (Linux host)

The Linux equivalent tool is `vmware-netcfg`, or the same **Virtual Network Editor** accessible from the VMware Workstation menu.

1. Launch VMware Workstation (`vmware &`), or run the editor directly:
   ```bash
   sudo /usr/bin/vmware-netcfg
   ```
2. Select **VMnet1 (Host-only)**.
3. Set:
   - **Subnet IP:** `192.168.10.0`
   - **Subnet mask:** `255.255.255.0`
4. Uncheck the DHCP service for VMnet1.
5. Leave **VMnet8 (NAT)** untouched.
6. Apply changes and restart the VMware networking service if prompted:
   ```bash
   sudo systemctl restart vmware-networks.service
   ```

---

## Per-VM dual-NIC configuration

VMware's default **New Virtual Machine Wizard** only creates a single NIC (bridged or NAT depending on your last selection). Add the second adapter manually for every VM:

1. With the VM powered off, open **VM → Settings → Add… → Network Adapter**.
2. For the adapter that will be **NIC 1**: set network connection to **NAT**.
3. For the adapter that will be **NIC 2**: set network connection to **Custom → VMnet1 (Host-only)**.
4. Confirm both adapters show as **Connected** and **Connect at power on** is checked for both.

This exact two-adapter configuration is baked into both Golden Baseline templates ([`04`](./04-golden-baseline-windows-server-2025.md), [`05`](./05-golden-baseline-rocky-linux-10.md)), so every VM cloned from them inherits it automatically — you only need to repeat this step manually if building a VM from scratch outside the Golden Baseline workflow.

---

## IP allocation table

Two separate naming conventions are used throughout this lab — see [VM and hostname naming convention](#vm-and-hostname-naming-convention) below for the reasoning.

| Host | VMware VM name | OS hostname | NIC 2 static IP | Role |
|---|---|---|---|---|
| DC01 | `DC01_10.10` | `DC01` | `192.168.10.10` | Active Directory, DNS, DHCP, NTP |
| WINAPP01 | `WINAPP01_10.15` | `WINAPP01` | `192.168.10.15` | IIS, SQL Server, WSUS |
| WEB01 | `WEB01_10.21` | `WEB01` | `192.168.10.21` | Nginx (reverse proxy / load balancer), Apache × 2, MariaDB, Suricata |
| MON01 | `MON01_10.40` | `MON01` | `192.168.10.40` | Zabbix, Wazuh |
| ELK01 | `ELK01_10.50` | `ELK01` | `192.168.10.50` | Elasticsearch, Logstash, Kibana |
| LOG02 | `LOG02_10.51` | `LOG02` | `192.168.10.51` | OpenSearch, OpenSearch Dashboards |
| CLIENT01 | `CLIENT01_dhcp` | `CLIENT01` | DHCP (`192.168.10.100`–`200` pool, served by DC01) | Domain-joined test client |

**Reserved ranges within `192.168.10.0/24`:**

| Range | Purpose |
|---|---|
| `.1`–`.9` | Reserved (gateway, infrastructure) |
| `.10`–`.19` | Windows infrastructure servers (DC01, WINAPP01) |
| `.20`–`.29` | Web tier (WEB01) |
| `.40`–`.49` | Monitoring tier (MON01) |
| `.50`–`.59` | Log analytics tier (ELK01, LOG02) |
| `.100`–`.200` | DHCP pool for domain-joined clients (CLIENT01 and any future additions) |

---

## VM and hostname naming convention

- **VMware Workstation VM name** (the Library display name and `.vmx` folder name) always follows `HOST_lastTwoOctets`, using only the **last two octets** of the NIC 2 IP address — e.g. `DC01_10.10` for `192.168.10.10`. This keeps names short while still identifying each VM's role and IP at a glance in the Workstation Library, without opening it.
- **OS hostname** (set via `hostnamectl` on Linux or `Rename-Computer` on Windows) is just `HOST` — e.g. `DC01`. This is the name used for DNS records, domain join, and everywhere inside the OS itself. Never mix the two naming schemes.

---

## Static IP configuration reference

These are generic examples; the exact steps for each specific VM are repeated inline in that VM's own build document.

### Windows (NIC 2 / Host-only)

```powershell
# Replace "Ethernet1" with the actual interface name (Get-NetAdapter to list them)
New-NetIPAddress -InterfaceAlias "Ethernet1" -IPAddress 192.168.10.10 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias "Ethernet1" -ServerAddresses 192.168.10.10
```
(DNS server address is DC01's own IP once DC01 is built; other VMs will point to `192.168.10.10` as their resolver.)

### Rocky Linux (NIC 2 / Host-only)

Using `nmtui` (interactive):
```bash
sudo nmtui
```
Select the connection tied to the Host-only NIC → Edit → set IPv4 to **Manual** → enter address `192.168.10.21/24`, gateway blank (NIC 2 has no gateway — internet traffic goes out NIC 1), DNS `192.168.10.10`.

Or directly via `nmcli`:
```bash
sudo nmcli con mod "Wired connection 2" ipv4.addresses 192.168.10.21/24
sudo nmcli con mod "Wired connection 2" ipv4.dns 192.168.10.10
sudo nmcli con mod "Wired connection 2" ipv4.method manual
sudo nmcli con up "Wired connection 2"
```

---

## Verifying connectivity

After assigning static IPs to at least two VMs, confirm NIC 2 (internal) connectivity between them:

```bash
ping 192.168.10.10
```

And confirm NIC 1 (NAT/internet) is independently functional on each VM:

```bash
ping 8.8.8.8
curl -I https://rockylinux.org
```

Both should succeed independently — internal ping over NIC 2, internet reachability over NIC 1 — confirming the dual-interface split is working as designed.

---

## Next step

Continue to [`03-remote-access-tooling-setup.md`](./03-remote-access-tooling-setup.md) to install PuTTY, mRemoteNG, and WinSCP on the host machine.
