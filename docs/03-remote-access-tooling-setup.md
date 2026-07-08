# 03 — Remote access tooling setup

This guide installs the four tools used to connect into and secure access for every VM built in this lab: **PuTTY** (SSH), **mRemoteNG** (centralized RDP/SSH connection manager), **WinSCP** (SFTP file transfer), and **KeePass** (encrypted credential storage). All four run on the Windows host; if your host machine is Linux, alternatives are noted at the end of this document.

Set these up once now — every later VM build document assumes you already have a working way to connect to it once it's on the network.

---

## Table of contents

- [Why these three tools](#why-these-three-tools)
- [PuTTY](#putty)
- [mRemoteNG](#mremoteng)
- [WinSCP](#winscp)
- [KeePass](#keepass)
- [Pre-building the mRemoteNG connection list](#pre-building-the-mremoteng-connection-list)
- [Connecting mRemoteNG to a KeePass database](#connecting-mremoteng-to-a-keepass-database)
- [Linux host alternatives](#linux-host-alternatives)
- [Next step](#next-step)

---

## Why these tools

| Tool | Protocol | Used for |
|---|---|---|
| PuTTY | SSH | Quick one-off terminal sessions into Rocky Linux VMs (WEB01, MON01, ELK01, LOG02) |
| mRemoteNG | RDP + SSH | A single pane of glass listing every VM in this lab — switch between Windows RDP sessions and Linux SSH sessions without juggling separate windows |
| WinSCP | SFTP | Transferring files to/from Rocky Linux VMs — configuration files, downloaded source tarballs, exported logs |
| KeePass | Credential storage | A single encrypted database for every local admin/service account password created across all 7 VMs, instead of relying on memory or plaintext notes |

PuTTY and mRemoteNG overlap in purpose (both can open SSH sessions) — PuTTY is kept for quick single-session use and for its `puttygen`/`pageant` companion tools, while mRemoteNG is the primary tool once more than 1–2 VMs are running simultaneously. KeePass is introduced alongside them because this lab creates a meaningful number of local accounts across both operating systems (Golden Baseline local admin, AD accounts, SQL Server sa, Zabbix/Wazuh admin consoles) — a password manager from day one avoids reusing weak passwords out of convenience.

---

## PuTTY

1. Download: https://www.putty.org/
2. Select the **64-bit x86** MSI installer (or the standalone `putty.exe` if you prefer no installer).
3. Run the installer, accept defaults — this also installs `puttygen` (key generation), `pageant` (SSH agent), `plink` (command-line SSH), and `pscp`/`psftp`.
4. Launch PuTTY once to confirm it opens (**Start → PuTTY → PuTTY**).

No further configuration is needed — PuTTY is used later simply by entering a VM's IP address and clicking **Open**.

---

## mRemoteNG

1. Download: https://github.com/mRemoteNG/mRemoteNG (see the **Releases** section for the latest installer, or **Assets** on the latest release)
2. Run the installer, accept defaults.
3. On first launch, mRemoteNG prompts to set a **master password** for encrypting saved credentials in its connection file — set one and remember it (there is no recovery if forgotten; you'd need to re-enter every saved credential).
4. Familiarize yourself with the layout:
   - **Connections panel** (left) — tree of saved connections, organized into folders.
   - **Config panel** (right, when a connection is selected) — protocol, hostname, credentials, display settings.

---

## WinSCP

1. Download: https://winscp.net/
2. Run the installer. When prompted for **"interface style"**, either Commander (dual-pane, Norton Commander style) or Explorer (single-pane, Windows Explorer style) works — Commander is generally faster once you're moving files between the host and multiple VMs.
3. Launch WinSCP once to confirm it opens.

WinSCP sessions are configured per-VM as needed later (its own login dialog, or import connections directly from PuTTY's saved sessions via **Tools → Import Sites**).

---

## KeePass
 
1. Download: https://keepass.info/download.html
2. Choose the **Installer** (`.exe`) or **Portable** (`.zip`) package — either works; Portable is convenient if you want the database file and executable to travel together on removable media.
3. Run the installer, accept defaults.
4. On first launch, create a new database: **File → New**.
   - Save it somewhere backed up regularly (not inside any VM — the host machine, or a synced folder).
   - Set a strong master password. This single password protects every credential stored for all 7 VMs, so make it one you can reliably remember or store securely outside KeePass itself (e.g. written down and kept somewhere physically secure).
5. Create one group per VM to keep entries organized, matching the naming convention from [`02-network-architecture-planning.md`](./02-network-architecture-planning.md#vm-and-hostname-naming-convention):
```
   enterprise-infra-lab/
   ├── DC01_10.10/
   │   ├── Local Administrator
   │   └── Domain Admin
   ├── WINAPP01_10.15/
   │   ├── Local Administrator
   │   └── SQL Server sa
   ├── WEB01_10.21/
   │   ├── root / sudo user
   │   └── MariaDB root
   ├── MON01_10.40/
   │   ├── root / sudo user
   │   ├── Zabbix Frontend admin
   │   └── Wazuh Dashboard admin
   ├── OPS01_10.41/
   │   ├── root / sudo user
   │   ├── Ansible (SSH key passphrase, if used)
   │   └── Keycloak admin console
   ├── LOG01_10.50/
   │   ├── root / sudo user
   │   └── Kibana/Elasticsearch
   ├── LOG02_10.51/
   │   ├── root / sudo user
   │   └── OpenSearch Dashboards
   └── CLIENT01/
       └── Local Administrator
```
6. Add entries as each VM is built throughout the rest of this guide, rather than trying to fill the database in now — the point is to have the structure ready so no password ever gets written down in plaintext or trusted to memory.
---
 
## Pre-building the mRemoteNG connection list
 
Set up the full connection list now, using the [IP allocation table](./02-network-architecture-planning.md#ip-allocation-table) from the previous document. Credentials can be left blank and filled in as each VM is actually built (or linked to KeePass — see below) — this just pre-stages the list so nothing needs to be typed from memory later.
 
**Naming rule:** every connection name in mRemoteNG must follow the same `HOST_lastTwoOctets` convention used for the VMware VM name itself (see [`02-network-architecture-planning.md`](./02-network-architecture-planning.md#vm-and-hostname-naming-convention)). This keeps the connection list, the VMware Library, and this guide's documentation all referring to the same identifier — no translating between "the VM called X" and "the connection called Y".
 
1. In mRemoteNG, right-click the root of the **Connections** panel → **Add Folder** → name it `enterprise-infra-lab`.
2. Right-click that folder → **Add Connection** for each VM, using the connection name shown below (not just the bare hostname):

| Connection name | Protocol | Hostname/IP | Notes |
|---|---|---|---|
| `DC01_10.10` | RDP | `192.168.10.10` | Stays on the default port `3389` — see [`06`](./06-dc01-active-directory-dns-dhcp.md) |
| `WINAPP01_10.15` | RDP | `192.168.10.15` | Port to be confirmed once configured in [`09`](./09-winapp01-iis-sql-wsus.md) |
| `WEB01_10.21` | SSH | `192.168.10.21` | |
| `MON01_10.40` | SSH | `192.168.10.40` | |
| `OPS01_10.41` | SSH | `192.168.10.41` | |
| `LOG01_10.50` | SSH | `192.168.10.50` | |
| `LOG02_10.51` | SSH | `192.168.10.51` | |
| `CLIENT01_dhcp` | RDP | *(assign once DHCP lease is known)* | |
 
3. For each RDP connection, set **Protocol → RDP**, **Port → 3389** — this lab keeps the default RDP port throughout, so no later update is needed for these connections.
4. For each SSH connection, set **Protocol → SSH2**, **Port → 22**.
5. Save the connection file (**File → Save Connections**) somewhere you'll remember — mRemoteNG keeps this as a single encrypted `.mrng` file.

---

## Connecting mRemoteNG to a KeePass database

mRemoteNG can pull credentials directly from the KeePass database created above, instead of storing passwords a second time inside its own connection file.

1. In mRemoteNG, go to **Tools → Options → Advanced**.
2. Enable **Use KeePass integration** (or **Credential Manager**, depending on mRemoteNG version).
3. Point it at the KeePass database file created earlier.
4. When prompted, unlock the KeePass database with its master password — mRemoteNG will use the KeePass KPScript/API integration to read matching entries by title.
5. For each connection created above, set its **Credential** field to reference the matching KeePass entry (e.g. the `DC01_10.10 / Local Administrator` entry) instead of typing a username/password directly into mRemoteNG.

This means the master password protecting KeePass is the only credential you need to remember for the entire lab — every VM-specific password lives in one encrypted, purpose-built store rather than scattered across mRemoteNG's own storage, sticky notes, or memory.

---

## Linux host alternatives

If your host machine is Linux instead of Windows (per the dual-platform setup in [`00-vmware-workstation-installation.md`](./00-vmware-workstation-installation.md)), use these equivalents instead of the Windows-only tools above:

| Windows tool | Linux equivalent | Notes |
|---|---|---|
| PuTTY | native `ssh` in a terminal | No installation needed on most distributions |
| mRemoteNG | Remmina — https://remmina.org/ | Supports both RDP and SSH in one app, same "connection list" concept |
| WinSCP | `sftp` CLI, or FileZilla — https://filezilla-project.org/ | FileZilla gives a GUI closer to WinSCP's Commander mode |
| KeePass | KeePassXC — https://keepassxc.org/ | Fully compatible with the same `.kdbx` database format — a database created in KeePass on Windows opens directly in KeePassXC on Linux, and vice versa |

The rest of this guide's remote-access instructions reference PuTTY/mRemoteNG/WinSCP by name for consistency, but any SSH/RDP/SFTP client works identically against the same IP addresses and credentials.

---

## Next step

Continue to [`04-golden-baseline-windows-server-2025.md`](./04-golden-baseline-windows-server-2025.md) to build the sealed Windows Server 2025 template used to clone DC01 and WINAPP01.
