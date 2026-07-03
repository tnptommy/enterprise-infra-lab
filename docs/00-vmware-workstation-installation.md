# 00 — VMware Workstation installation

This is the first document in the build order. Everything else in this guide assumes VMware Workstation Pro is already installed and working on your host machine.

VMware Workstation Pro runs on both **Windows** and **Linux** hosts. Follow the section that matches your host operating system — the rest of this guide (network configuration, VM creation, OS installation) is identical afterward regardless of which host OS you used.

---

## Table of contents

- [Prerequisites](#prerequisites)
- [Download](#download)
- [Installing on a Windows host](#installing-on-a-windows-host)
- [Installing on a Linux host](#installing-on-a-linux-host)
- [Post-install verification (both platforms)](#post-install-verification-both-platforms)
- [Troubleshooting](#troubleshooting)
- [Next step](#next-step)

---

## Prerequisites

Regardless of host OS, hardware-assisted virtualization must be enabled in the BIOS/UEFI firmware before VMware Workstation can create or run any VM.

1. Reboot the host machine and enter BIOS/UEFI setup (commonly `Del`, `F2`, `F10`, or `Esc` at boot — check your motherboard/laptop manufacturer).
2. Locate the virtualization setting:
   - Intel CPUs: **Intel VT-x** (sometimes labeled "Intel Virtualization Technology")
   - AMD CPUs: **AMD-V** (sometimes labeled "SVM Mode")
3. Set it to **Enabled**.
4. If running nested virtualization matters to you later, also enable **VT-d** (Intel) or **AMD-Vi/IOMMU** (AMD) — not required for this guide, but harmless to enable.
5. Save and exit BIOS/UEFI.

To confirm virtualization is active from within the OS before installing anything:

**Windows:**
```powershell
Get-ComputerInfo -Property "HyperVRequirementVirtualizationFirmwareEnabled"
```

<img width="671" height="144" alt="image" src="https://github.com/user-attachments/assets/ce80aa9e-0643-4c84-b1f4-bc700e2281ac" />

Expect `True`. If `False`, the BIOS/UEFI setting above was not saved correctly.

**Linux:**
```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```
Expect a number greater than `0` (one per CPU thread). `vmx` = Intel VT-x, `svm` = AMD-V.

Also confirm hardware requirements from the [README](../README.md#hardware-requirements) are met before proceeding — this lab's 7 VMs require significantly more RAM and disk than a default install.

---

## Download

VMware Workstation Pro is available free for personal use directly from Broadcom (VMware's current owner as of the Broadcom acquisition).

- Official download page: https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Workstation%20Pro&freeDownloads=true
- Requires a free Broadcom account to download — create one if you don't already have it.

Both the Windows and Linux installers are distributed from the same download page — select your host platform there.

---

## Installing on a Windows host

1. Run the downloaded installer (`VMware-workstation-full-<version>.exe`) as Administrator.
2. Accept the license agreement, choose **Typical** setup type (sufficient for this guide).
3. Choose the installation location (default `C:\Program Files (x86)\VMware\VMware Workstation\` is fine, or point it to a drive with more free space if your system drive is small).
4. Leave both **"Enhanced Keyboard Driver"** and **"Add VMware Workstation console tools into system PATH"** checked — the PATH option is convenient for scripting VM operations later via `vmrun`.
5. Decide on user experience settings (check for updates, send usage data) — either is fine for a lab environment.
6. Click **Install**, wait for completion, then **Restart** the host when prompted (required — the installer adds low-level networking and virtualization drivers).
7. On first launch, enter a license key if you have one, or choose **Try VMware Workstation Pro** for the evaluation period, or use the **free personal use** license option now offered directly in the product (follow the in-app prompt — Broadcom moved licensing into the application itself rather than a separate key).

---

## Installing on a Linux host

VMware Workstation Pro for Linux ships as a self-extracting `.bundle` installer. Rocky Linux 10 is used as an example below, but the same steps apply to any modern RPM- or Debian-based distribution with minor package-manager substitutions.

### 1. Install build dependencies

VMware Workstation compiles two kernel modules (`vmmon` and `vmnet`) against your running kernel during installation, so a compiler toolchain and matching kernel headers are required first.

**Rocky Linux / RHEL-based:**
```bash
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y kernel-headers kernel-devel gcc make elfutils-libelf-devel
```

**Debian/Ubuntu-based:**
```bash
sudo apt update
sudo apt install -y build-essential gcc make linux-headers-$(uname -r)
```

### 2. Run the installer

```bash
chmod +x VMware-Workstation-Full-<version>.x86_64.bundle
sudo ./VMware-Workstation-Full-<version>.x86_64.bundle
```

This launches a graphical (or text-mode, over SSH) installation wizard. Accept the license agreement and use default install paths unless you have a specific reason to change them.

### 3. Allow kernel module compilation

Near the end of installation, the installer attempts to build and load `vmmon`/`vmnet` against your current kernel automatically. If it fails (common on very new kernels not yet supported by the installer's bundled sources), build them manually using the community-maintained module source:

```bash
git clone https://github.com/mkubecek/vmware-host-modules.git
cd vmware-host-modules
git checkout workstation-<version>   # match your installed VMware Workstation version
make
sudo make install
```
Source: https://github.com/mkubecek/vmware-host-modules

### 4. Start the required services

```bash
sudo systemctl enable --now vmware.service
sudo systemctl status vmware.service
```

### 5. Launch VMware Workstation

```bash
vmware &
```
On first launch, enter a license key, start an evaluation, or use the in-app free personal use option — same as the Windows flow described above.

---

## Post-install verification (both platforms)

1. Open VMware Workstation. The **Library** pane should be empty but visible with no errors.
2. Open **Edit → Virtual Network Editor** (Windows) or **Edit → Virtual Network Editor** (Linux) and confirm at least these default networks exist:
   - `VMnet0` (Bridged)
   - `VMnet1` (Host-only)
   - `VMnet8` (NAT)

   These three networks are exactly what this guide's [dual-interface network design](../README.md#network-architecture-summary) is built on top of — NAT (VMnet8) for internet access and Host-only (VMnet1, later re-addressed to `192.168.10.0/24`) for internal lab traffic. Detailed reconfiguration steps are in [`01-network-architecture-planning.md`](./02-network-architecture-planning.md).

3. Confirm virtualization is functioning by creating a throwaway test VM (any small Linux ISO, or skip this if you're confident BIOS settings are correct) and powering it on — it should boot without a "VT-x/AMD-V is not enabled" error.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "This host supports Intel VT-x, but Intel VT-x is disabled" on VM power-on | Virtualization disabled in BIOS/UEFI, or disabled by another hypervisor (Hyper-V on Windows) | Re-check BIOS/UEFI setting; on Windows, disable Hyper-V/Windows Hypervisor Platform/Credential Guard if present (`bcdedit /set hypervisorlaunchtype off`, then reboot) |
| `vmmon`/`vmnet` fail to load on Linux after a kernel update | Modules were built against the old kernel | Rebuild using `vmware-host-modules` (see step 3 above) after every kernel upgrade, or switch to a distro with a stable/LTS kernel to reduce rebuild frequency |
| Windows host shows "VMware Authorization Service" not running | Service failed to start | `services.msc` → start **VMware Authorization Service** manually, set to Automatic |
| Linux host: `vmware` command not found | PATH not updated, or shell not reloaded | Close and reopen the terminal, or run `hash -r` |

---

## Next step

Continue to [`01-iso-acquisition-and-verification.md`](./01-iso-acquisition-and-verification.md) to download and verify the ISOs used throughout this guide.
