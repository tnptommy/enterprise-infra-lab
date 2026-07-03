# 01 — ISO acquisition and verification

This guide downloads and verifies the three installation ISOs used throughout the rest of this lab. Doing this once up front means every later VM build (Golden Baseline, DC01, WINAPP01, CLIENT01, WEB01, MON01, ELK01, LOG02) can simply reference an already-verified local file instead of re-downloading.

---

## Table of contents

- [ISO summary](#iso-summary)
- [Shared ISO storage folder](#shared-iso-storage-folder)
- [Windows Server 2025 (Evaluation)](#windows-server-2025-evaluation)
- [Windows 11 Pro (24H2)](#windows-11-pro-24h2)
- [Rocky Linux 10 (Minimal)](#rocky-linux-10-minimal)
- [Checksum verification commands](#checksum-verification-commands)
- [Next step](#next-step)

---

## ISO summary

| ISO | Used for | Edition | Download page |
|---|---|---|---|
| Windows Server 2025 | DC01, WINAPP01 (via Golden Baseline) | Datacenter (Desktop Experience), converted from Evaluation | https://www.microsoft.com/evalcenter/evaluate-windows-server-2025 |
| Windows 11 | CLIENT01 | Pro (24H2) | https://www.microsoft.com/software-download/windows11 |
| Rocky Linux 10 | WEB01, MON01, ELK01, LOG02 (via Golden Baseline) | Minimal | https://rockylinux.org/download |

Only one copy of each ISO is needed — every VM built from a given OS is created from the same file, either by mounting it directly or by cloning the sealed [Golden Baseline](./04-golden-baseline-windows-server-2025.md) template built from it.

---

## Shared ISO storage folder

Pick one folder on the host machine to store all ISOs, and reuse it for every VM build in this guide. Keeping ISOs outside of any individual VM's folder avoids duplicating multi-gigabyte files across 7 VMs.

**Windows host example:**
```
D:\ISOs\
├── WindowsServer2025-Eval.iso
├── Windows11-24H2.iso
└── Rocky-10-x86_64-minimal.iso
```

**Linux host example:**
```
~/isos/
├── WindowsServer2025-Eval.iso
├── Windows11-24H2.iso
└── Rocky-10-x86_64-minimal.iso
```

When creating each VM later in this guide, point the virtual CD/DVD drive at the corresponding file in this shared folder.

---

## Windows Server 2025 (Evaluation)

1. Go to: https://www.microsoft.com/evalcenter/evaluate-windows-server-2025
2. Register/sign in with a Microsoft account if prompted (required for the Eval Center).
3. Select the **ISO** download option (not the VHD) and choose the 64-bit edition.
> <img width="1133" height="530" alt="image" src="https://github.com/user-attachments/assets/e72dafef-ccd4-4eba-b460-db5b5bbdc80d" />

4. Save the file into your shared ISO folder, renamed for clarity, e.g. `WindowsServer2025-Eval.iso`.

> This is the free 180-day Evaluation edition. It is converted to a licensed **Datacenter** edition using `DISM /Set-Edition` and activated against a KMS host during the [Golden Baseline build](./04-golden-baseline-windows-server-2025.md) — do not worry about the evaluation period expiring before that step.

---

## Windows 11 Pro (24H2)

1. Go to: https://www.microsoft.com/software-download/windows11
2. Under **"Download Windows 11 Disk Image (ISO)"**, select the edition (this guide uses the general consumer/business ISO, which includes Pro) and language.
3. Click **Download**, save into your shared ISO folder as `Windows11-24H2.iso`.
> <img width="760" height="422" alt="image" src="https://github.com/user-attachments/assets/15ef871e-d87e-4315-9991-13f49c150a45" />

> The generic Windows 11 ISO from this page lets you choose the edition (Home/Pro/etc.) during setup, or it installs Pro by default depending on the build — either way, the GVLK for Windows 11 Pro used later in this guide (`W269N-WFGWX-YVC9B-4J6C9-T83GX`) will correctly activate a Pro installation regardless of which edition selector appeared during setup.

---

## Rocky Linux 10 (Minimal)

1. Go to: https://rockylinux.org/download
2. Select **Rocky Linux 10**, architecture **x86_64**.
3. Choose a mirror close to your location, then download the **Minimal** ISO (not DVD or Boot — Minimal keeps the base install small, matching this guide's Golden Baseline approach of adding only what's needed afterward).
> <img width="920" height="347" alt="image" src="https://github.com/user-attachments/assets/ec03220a-a49d-4f3e-a094-3e38a12c96c7" />

4. Save into your shared ISO folder as `Rocky-10-x86_64-minimal.iso`.
5. On the same mirror page, also note the `CHECKSUM` file link (usually `CHECKSUM` or `*-CHECKSUM`) — needed for verification below.

---

## Checksum verification commands

Verifying the SHA256 checksum of every downloaded ISO confirms the file wasn't corrupted or tampered with in transit. Run the appropriate command below depending on your **host** OS (from [`00-vmware-workstation-installation.md`](./00-vmware-workstation-installation.md)), then compare the output against the checksum published on each download page.

### Windows host

```powershell
certutil -hashfile "D:\ISOs\WindowsServer2025-Eval.iso" SHA256
certutil -hashfile "D:\ISOs\Windows11-24H2.iso" SHA256
certutil -hashfile "D:\ISOs\Rocky-10-x86_64-minimal.iso" SHA256
```

### Linux host

```bash
sha256sum ~/isos/WindowsServer2025-Eval.iso
sha256sum ~/isos/Windows11-24H2.iso
sha256sum ~/isos/Rocky-10-x86_64-minimal.iso
```

For Rocky Linux specifically, you can verify directly against the published checksum file instead of comparing by eye:

```bash
cd ~/isos
curl -O https://download.rockylinux.org/pub/rocky/10/isos/x86_64/CHECKSUM
sha256sum -c CHECKSUM --ignore-missing
```
Expect `OK` next to the Rocky Linux ISO filename.

> Microsoft does not always publish a standalone checksum file for Evaluation Center downloads. In that case, checksum verification mainly protects against a corrupted/incomplete download (re-download if `certutil`/`sha256sum` fails to complete or the file size looks wrong) rather than a published-hash comparison.

---

## Next step

Continue to [`02-network-architecture-planning.md`](./02-network-architecture-planning.md) to configure the dual-interface VMware network (NAT + Host-only on `192.168.10.0/24`) before building any VM.
