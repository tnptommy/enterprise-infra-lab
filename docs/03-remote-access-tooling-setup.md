# 03 — Remote access tooling setup

This guide installs the three tools used to connect into every VM built in this lab: **PuTTY** (SSH), **mRemoteNG** (centralized RDP/SSH connection manager), and **WinSCP** (SFTP file transfer). All three run on the Windows host; if your host machine is Linux, alternatives are noted at the end of this document.

Set these up once now — every later VM build document assumes you already have a working way to connect to it once it's on the network.

---

## Table of contents

- [Why these three tools](#why-these-three-tools)
- [PuTTY](#putty)
- [mRemoteNG](#mremoteng)
- [WinSCP](#winscp)
- [Pre-building the mRemoteNG connection list](#pre-building-the-mremoteng-connection-list)
- [Linux host alternatives](#linux-host-alternatives)
- [Next step](#next-step)

---

## Why these three tools

| Tool | Protocol | Used for |
|---|---|---|
| PuTTY | SSH | Quick one-off terminal sessions into Rocky Linux VMs (WEB01, MON01, ELK01, LOG02) |
| mRemoteNG | RDP + SSH | A single pane of glass listing every VM in this lab — switch between Windows RDP sessions and Linux SSH sessions without juggling separate windows |
| WinSCP | SFTP | Transferring files to/from Rocky Linux VMs — configuration files, downloaded source tarballs, exported logs |

PuTTY and mRemoteNG overlap in purpose (both can open SSH sessions) — PuTTY is kept for quick single-session use and for its `puttygen`/`pageant` companion tools, while mRemoteNG is the primary tool once more than 1–2 VMs are running simultaneously.

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

## Pre-building the mRemoteNG connection list

Set up the full connection list now, using the [IP allocation table](./02-network-architecture-planning.md#ip-allocation-table) from the previous document. Credentials can be left blank and filled in as each VM is actually built — this just pre-stages the list so nothing needs to be typed from memory later.

1. In mRemoteNG, right-click the root of the **Connections** panel → **Add Folder** → name it `enterprise-infra-lab`.
2. Right-click that folder → **Add Connection** for each VM:

| Connection name | Protocol | Hostname/IP | Notes |
|---|---|---|---|
| `DC01` | RDP | `192.168.10.10` | Custom RDP port set once configured in [`06`](./06-dc01-active-directory-dns-dhcp.md) |
| `WINAPP01` | RDP | `192.168.10.15` | Custom RDP port set once configured in [`09`](./09-winapp01-iis-sql-wsus.md) |
| `WEB01` | SSH | `192.168.10.21` | |
| `MON01` | SSH | `192.168.10.40` | |
| `ELK01` | SSH | `192.168.10.50` | |
| `LOG02` | SSH | `192.168.10.51` | |
| `CLIENT01` | RDP | *(assign once DHCP lease is known)* | |

3. For each RDP connection, set **Protocol → RDP**, **Port → 3389** initially (update later once each Windows VM's build document changes the RDP port for hardening purposes).
4. For each SSH connection, set **Protocol → SSH2**, **Port → 22**.
5. Save the connection file (**File → Save Connections**) somewhere you'll remember — mRemoteNG keeps this as a single encrypted `.mrng` file.

---

## Linux host alternatives

If your host machine is Linux instead of Windows (per the dual-platform setup in [`00-vmware-workstation-installation.md`](./00-vmware-workstation-installation.md)), use these equivalents instead of the Windows-only tools above:

| Windows tool | Linux equivalent | Notes |
|---|---|---|
| PuTTY | native `ssh` in a terminal | No installation needed on most distributions |
| mRemoteNG | Remmina — https://remmina.org/ | Supports both RDP and SSH in one app, same "connection list" concept |
| WinSCP | `sftp` CLI, or FileZilla — https://filezilla-project.org/ | FileZilla gives a GUI closer to WinSCP's Commander mode |

The rest of this guide's remote-access instructions reference PuTTY/mRemoteNG/WinSCP by name for consistency, but any SSH/RDP/SFTP client works identically against the same IP addresses and credentials.

---

## Next step

Continue to [`04-golden-baseline-windows-server-2025.md`](./04-golden-baseline-windows-server-2025.md) to build the sealed Windows Server 2025 template used to clone DC01 and WINAPP01.
