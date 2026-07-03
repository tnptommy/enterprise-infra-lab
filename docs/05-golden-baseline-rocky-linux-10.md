# 05 — Golden Baseline: Rocky Linux 10

This document builds a single, sealed **Golden Baseline** VM for Rocky Linux 10 Minimal. Every Linux VM in this lab (WEB01, MON01, ELK01, LOG02) is created by cloning this template rather than installing Rocky Linux from scratch each time.

Build this once. It is not assigned a static IP, a hostname matching the naming convention, or any role-specific software — it stays a minimal, patched, sealed image.

---

## Table of contents

- [Why a golden baseline](#why-a-golden-baseline)
- [VM specification](#vm-specification)
- [Step 1 — Create the VM and install Rocky Linux 10 Minimal](#step-1--create-the-vm-and-install-rocky-linux-10-minimal)
- [Step 2 — Enable SSH and switch to PuTTY](#step-2--enable-ssh-and-switch-to-putty)
- [Step 3 — Install open-vm-tools](#step-3--install-open-vm-tools)
- [Step 4 — Configure the dual-NIC network adapters](#step-4--configure-the-dual-nic-network-adapters)
- [Step 5 — Apply package baseline](#step-5--apply-package-baseline)
- [Step 6 — Seal the image](#step-6--seal-the-image)
- [Step 7 — Convert to a reusable template](#step-7--convert-to-a-reusable-template)
- [Cloning this baseline later](#cloning-this-baseline-later)
- [Next step](#next-step)

---

## Why a golden baseline

Installing Rocky Linux from ISO, patching it, and installing `open-vm-tools` is identical across all four Linux VMs in this lab — repeating it four times wastes effort on steps that produce the exact same result each time. Building it once and cloning it afterward means every later document (WEB01, MON01, ELK01, LOG02) starts from "already patched, already has VMware Tools" and jumps straight into role-specific configuration.

---

## VM specification

| Setting | Value |
|---|---|
| VM name (VMware Library) | `GoldenBaseline-Rocky10` *(not part of the `HOST_lastTwoOctets` convention — this template has no IP or role of its own)* |
| vCPU | 2 |
| RAM | 2 GB |
| Disk | 20 GB (single disk — thin provisioned) |
| Network adapters | 2 (see [Step 4](#step-4--configure-the-dual-nic-network-adapters)) |
| ISO | `Rocky-10-x86_64-minimal.iso` from [`01-iso-acquisition-and-verification.md`](./01-iso-acquisition-and-verification.md) |

These specs are intentionally modest — once cloned for WEB01, MON01, ELK01, or LOG02, both RAM and disk are increased to that VM's actual requirement from the [README's VM inventory](../README.md#virtual-machine-inventory). Disk expansion after cloning is covered in [Cloning this baseline later](#cloning-this-baseline-later).

---

## Step 1 — Create the VM and install Rocky Linux 10 Minimal

1. In VMware Workstation: **File → New Virtual Machine → Custom (advanced)**.
2. Guest OS: **Linux**, version **Red Hat Enterprise Linux 9 64-bit** (or the closest RHEL version your VMware Workstation release lists — VMware Workstation does not list Rocky Linux by name, and RHEL is its correct upstream match; the installer works identically regardless of this label).
3. Point the installer to the ISO from your shared ISO folder ([`01-iso-acquisition-and-verification.md`](./01-iso-acquisition-and-verification.md)).
4. Name the VM `GoldenBaseline-Rocky10` and choose a storage location.
5. Allocate resources per the [VM specification](#vm-specification) table above.
6. Complete the wizard and power on the VM.
7. In the Rocky Linux installer (Anaconda), choose **Minimal Install** under Software Selection.
8. Under Installation Destination, select the disk and choose **Custom** partitioning, then create the following layout manually:

| Mount point | Size | Filesystem | Notes |
|---|---|---|---|
| `/boot` | 1 GiB | xfs | Standard partition, outside LVM |
| `/boot/efi` | 200 MiB | EFI System Partition | Only if the VM firmware is UEFI — check **VM → Settings → Options → Advanced → Firmware type** beforehand; skip this row if using BIOS |
| LVM Volume Group `rl` → Logical Volume `root` | Remaining space | xfs | Mount at `/` |

No swap partition is created here deliberately — swap is sized per-VM later (2× that VM's RAM, per the [README's resource table](../README.md#virtual-machine-inventory)) using a swap file, so it isn't locked into the golden baseline's disk layout. Using LVM for the root volume (rather than a plain partition) is what makes disk expansion after cloning straightforward later — see [Cloning this baseline later](#cloning-this-baseline-later).

9. On the **Network & Host Name** screen, confirm the network interface is enabled (toggle it **On** if not) so the VM has an IP address as soon as it boots — this is needed for the SSH step immediately following installation.
10. Set the root password and create a local admin user with sudo privileges when prompted.
11. Complete installation and reboot.

---

## Step 2 — Enable SSH and switch to PuTTY

Rather than continuing the rest of this build inside the cramped VMware console window, switch to PuTTY as soon as the VM is reachable over the network. Everything from here through the end of this document can be done over SSH.

1. Log in at the console once (root or the sudo user created during installation).
2. Confirm `sshd` is running — Rocky Linux's minimal install enables it by default, but verify rather than assume:

```bash
sudo systemctl status sshd
```
If it isn't running for any reason:

```bash
sudo systemctl enable --now sshd
```

3. Confirm the firewall allows SSH (the default `public` zone permits it out of the box, but verify):

```bash
sudo firewall-cmd --list-services
```
Expect `ssh` in the output. If missing:

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

4. Get the VM's current IP address (only one NIC exists at this point — NIC 2 is added in the next step):

```bash
ip a
```
Note the address shown for the active interface (commonly `ens160` or similar, on VMware's default NAT/DHCP range at this stage).

5. From the host machine, open **PuTTY** ([installed in `03-remote-access-tooling-setup.md`](./03-remote-access-tooling-setup.md)), enter that IP address, and connect.
6. Accept the host key prompt on first connection, then log in with the sudo user created during installation.

From this point forward, every command in this document is run from the PuTTY session rather than the VMware console.

---

## Step 3 — Install open-vm-tools

Rocky Linux does not include VMware Tools by default — install the open-source equivalent, which is the recommended approach for RHEL-based distributions (see [`00-vmware-workstation-installation.md`](./00-vmware-workstation-installation.md#troubleshooting) if the underlying VMware kernel modules on the **host** ever need attention — this step is unrelated to that and only concerns the **guest**).

```bash
sudo dnf install -y open-vm-tools
sudo systemctl enable --now vmtoolsd
sudo systemctl status vmtoolsd
```

Confirm it's running (`active (running)` in the status output) before continuing.

---

## Step 4 — Configure the dual-NIC network adapters

This baseline follows the [dual-interface network design](./02-network-architecture-planning.md) — every VM cloned from it inherits both adapters automatically.

1. Power off the VM (from the PuTTY session: `sudo shutdown -h now`, then power off from VMware Workstation once the guest has fully shut down).
2. **VM → Settings** — confirm **Network Adapter** is set to **NAT**. This is NIC 1.
3. **VM → Settings → Add… → Network Adapter** — set the new adapter to **Custom → VMnet1 (Host-only)**. This is NIC 2.
4. Power the VM back on.
5. Reconnect with PuTTY using the same (or possibly renewed) NAT IP address from [Step 2](#step-2--enable-ssh-and-switch-to-putty), then confirm both interfaces come up automatically via DHCP:

```bash
nmcli device status
ip a
```
You should see two interfaces (commonly `ens160` and `ens192`, or similar) each with a DHCP-assigned address — one from VMnet8's NAT range, one from VMnet1's Host-only range. Leave both on DHCP for now; static IPs are configured per-VM after cloning, in each VM's own build document.

---

## Step 5 — Apply package baseline

```bash
sudo dnf update -y
sudo dnf install -y net-tools lftp curl tar wget zip telnet vim rsync
sudo dnf groupinstall -y "Development Tools"
```

Reboot if the kernel was updated:

```bash
sudo reboot
```
Reconnect with PuTTY once the VM finishes rebooting.

---

## Step 6 — Seal the image

Rocky Linux has no single equivalent to Windows' `sysprep`, but the same goal — ensuring every clone gets unique identity artifacts instead of duplicating the baseline's — is achieved with a few manual steps.

1. Clean the package manager cache:

```bash
sudo dnf clean all
```

2. Clear the machine ID so each clone generates its own on first boot:

```bash
sudo truncate -s 0 /etc/machine-id
```

3. Remove the SSH host keys so each clone generates its own unique keys (prevents every cloned VM from sharing identical SSH host identities, which would trigger host-key-mismatch warnings and is a genuine security concern for any SSH-based service):

```bash
sudo rm -f /etc/ssh/ssh_host_*
sudo systemctl enable sshd-keygen@rsa.service sshd-keygen@ecdsa.service sshd-keygen@ed25519.service
```
These `sshd-keygen@` units regenerate host keys automatically on first boot after cloning. If a clone doesn't have them for any reason, keys can also be regenerated manually with `sudo ssh-keygen -A`.

> This step deliberately removes the very host keys your current PuTTY session's saved host key fingerprint is based on. That's expected — end this PuTTY session after this step, since the baseline is about to be sealed and shut down anyway. Each future clone will present its own freshly generated key on first connection.

4. Clear shell history and logs:

```bash
history -c
cat /dev/null > ~/.bash_history
sudo journalctl --vacuum-time=1s
sudo rm -f /var/log/*.log
```

5. Shut down:

```bash
sudo shutdown -h now
```

---

## Step 7 — Convert to a reusable template

VMware Workstation doesn't have a dedicated "template" object like vSphere — the convention here is simply:

1. Leave the VM powered off in its sealed state.
2. (Recommended) Take a **Snapshot** named `sealed-baseline` before ever powering it on again, so you always have a known-good rollback point even if a future clone attempt accidentally boots this VM instead of a clone.
3. Treat this `.vmx` as read-only from this point forward — all future work happens on **Full Clones** made from it, never on this VM directly.

---

## Cloning this baseline later

When building WEB01, MON01, ELK01, LOG02, or any future Rocky Linux VM in this lab:

1. Right-click `GoldenBaseline-Rocky10` in the VMware Library → **Manage → Clone…**
2. Choose **Clone from: The current state in the virtual machine**.
3. Choose **Create a full clone** (not linked — a full clone has no dependency on the baseline VM's files continuing to exist, which matters if you ever archive or move the baseline later).
4. Name the clone per the [naming convention](./02-network-architecture-planning.md#vm-and-hostname-naming-convention) — e.g. `WEB01_10.21`.
5. Before powering on, adjust vCPU/RAM in **VM → Settings** to match that VM's actual requirement from the [README's VM inventory](../README.md#virtual-machine-inventory).
6. Power on the clone and reconnect via PuTTY using its NAT-assigned IP (`ip a`, same approach as [Step 2](#step-2--enable-ssh-and-switch-to-putty)) — the regenerated SSH host key from sealing means PuTTY will prompt to accept a new host key fingerprint the first time, which is expected for every fresh clone.
7. Expand the virtual disk if the target VM needs more than the baseline's 20 GB:
   - **VM → Settings → Hard Disk → Utilities → Expand…** — set the new maximum size (e.g. 50 GB for WEB01's OS disk).
   - With the clone powered back on, grow the LVM volume and filesystem to use the newly available space. First confirm which partition holds the LVM physical volume — this is partition 3 with the layout from [Step 1](#step-1--create-the-vm-and-install-rocky-linux-10-minimal) if using UEFI (`/boot/efi`, `/boot`, then the LVM partition), or partition 2 if using BIOS (no `/boot/efi`):

```bash
# Confirm the new raw disk size is visible, and identify the LVM partition number
lsblk

# Grow the partition (adjust the partition number — 3 for UEFI layouts, 2 for BIOS layouts)
sudo growpart /dev/sda 3

# Extend the LVM physical volume to use the newly available space
sudo pvresize /dev/sda3

# Extend the logical volume to use all remaining free space in the volume group
sudo lvextend -l +100%FREE /dev/mapper/rl-root

# Grow the XFS filesystem to fill the extended logical volume (Rocky Linux default fs)
sudo xfs_growfs /
```

8. Set the OS hostname (`HOST` only, no IP suffix):

```bash
sudo hostnamectl set-hostname WEB01
```

9. Configure the static IP on NIC 2 per [`02-network-architecture-planning.md`](./02-network-architecture-planning.md#static-ip-configuration-reference).
10. Add any second virtual disk this VM needs (e.g. WEB01's LVM demo disk, MON01/ELK01/LOG02's data disk) per that VM's own build document — the golden baseline intentionally ships with only one disk.
11. Proceed with that VM's own build document (e.g. [`07-web01-lamp-nginx-loadbalancer.md`](./07-web01-lamp-nginx-loadbalancer.md)) from there.

---

## Next step

Continue to [`06-dc01-active-directory-dns-dhcp.md`](./06-dc01-active-directory-dns-dhcp.md) to clone the Windows Server 2025 baseline into DC01 and build Active Directory, DNS, and DHCP.

