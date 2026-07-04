# 04 — Golden Baseline: Windows Server 2025

This document builds a single, sealed **Golden Baseline** VM for Windows Server 2025. Every Windows Server VM in this lab (DC01, WINAPP01) is created by cloning this template rather than installing Windows from scratch each time — this is the only place `DISM /Set-Edition`, `.NET Framework 3.5`, and KMS activation need to be configured.

Build this once. It is not assigned a static IP, a hostname matching the naming convention, or any role-specific software — it stays a minimal, patched, activated, sealed image.

---

## Table of contents

- [Why a golden baseline](#why-a-golden-baseline)
- [VM specification](#vm-specification)
- [Step 1 — Create the VM and install Windows Server 2025](#step-1--create-the-vm-and-install-windows-server-2025)
- [Step 2 — Verify VMware Tools](#step-2--verify-vmware-tools)
- [Step 3 — Configure the dual-NIC network adapters](#step-3--configure-the-dual-nic-network-adapters)
- [Step 4 — Enable .NET Framework 3.5 via DISM](#step-4--enable-net-framework-35-via-dism)
- [Step 5 — Convert Evaluation to Datacenter edition via DISM Set-Edition](#step-5--convert-evaluation-to-datacenter-edition-via-dism-set-edition)
- [Step 6 — Activate via GVLK against the online KMS host](#step-6--activate-via-gvlk-against-the-online-kms-host)
- [Step 7 — Apply Windows Update baseline](#step-7--apply-windows-update-baseline)
- [Step 8 — Clean up and seal the image](#step-8--clean-up-and-seal-the-image)
- [Step 9 — Convert to a reusable template](#step-9--convert-to-a-reusable-template)
- [Cloning this baseline later](#cloning-this-baseline-later)
- [Next step](#next-step)

---

## Why a golden baseline

Installing Windows Server from ISO, patching it, enabling .NET 3.5, converting its edition, and activating it takes real time — repeating that twice (once for DC01, once for WINAPP01) wastes effort on identical steps. Building it once, sealing it with `sysprep`, and cloning it afterward means every later document (DC01, WINAPP01) starts from "already patched, already activated, already has .NET 3.5" and jumps straight into role-specific configuration.

---

## VM specification

| Setting | Value |
|---|---|
| VM name (VMware Library) | `GoldenBaseline-WinServer2025` *(not part of the `HOST_lastTwoOctets` convention — this template has no IP or role of its own)* |
| vCPU | 2 |
| RAM | 4 GB |
| Disk | 60 GB (single disk — thin provisioned) |
| Network adapters | 2 (see [Step 3](#step-3--configure-the-dual-nic-network-adapters)) |
| ISO | `WindowsServer2025-Eval.iso` from [`01-iso-acquisition-and-verification.md`](./01-iso-acquisition-and-verification.md) |

These specs are intentionally modest — once cloned for DC01 or WINAPP01, adjust vCPU/RAM/disk up to each VM's actual requirement from the [README's VM inventory](../README.md#virtual-machine-inventory). Sysprep does not care about hardware sizing at clone time.

---

## Step 1 — Create the VM and install Windows Server 2025

1. In VMware Workstation: **File → New Virtual Machine → Custom (advanced)**.
2. Guest OS: **Microsoft Windows**, version **Windows Server 2025**.
> <img width="317" height="312" alt="image" src="https://github.com/user-attachments/assets/bb19a4ee-3f9f-4c00-968a-de7dc826ecdf" />
3. Point the installer to the ISO from your shared ISO folder ([`01-iso-acquisition-and-verification.md`](./01-iso-acquisition-and-verification.md)).
4. Name the VM `GoldenBaseline-WinServer2025` and choose a storage location.
5. Allocate resources per the [VM specification](#vm-specification) table above.
> <img width="571" height="584" alt="image" src="https://github.com/user-attachments/assets/f549e9d5-0d77-4529-b432-a6c0784b12b9" />
6. Complete the wizard and power on the VM.
7. In the Windows Setup screen, choose **Windows Server 2025 Standard/Datacenter (Desktop Experience)** — the specific edition selected here doesn't matter much, since [Step 5](#step-5--convert-evaluation-to-datacenter-edition-via-dism-set-edition) converts it explicitly regardless.
> <img width="512" height="383" alt="image" src="https://github.com/user-attachments/assets/04d324d7-6308-4445-b951-4c5397381a20" />
8. Complete installation: accept the license, choose **Custom install**, select the virtual disk, wait for installation to finish and reboot.
> <img width="512" height="384" alt="image" src="https://github.com/user-attachments/assets/0c0dea14-5cc0-4e8d-965c-ae1ef1e2289f" />
> <img width="512" height="384" alt="image" src="https://github.com/user-attachments/assets/8bbf5300-8d97-480a-be31-4eaf83428175" />
9. Set a local Administrator password when prompted.
> <img width="512" height="384" alt="image" src="https://github.com/user-attachments/assets/79e04d6e-4a10-4ae2-aae4-b469bc364320" />

---

## Step 2 — Verify VMware Tools

Recent versions of VMware Workstation install VMware Tools automatically — either as part of **Easy Install** during VM creation, or by auto-mounting and silently installing them shortly after the guest OS finishes its first boot. Manually running `setup64.exe` is usually unnecessary.

1. After Windows Server finishes installing and you're logged in, wait a minute or two, then check whether Tools are already present:

```powershell
Get-Service -Name VMTools
```
If this returns a service in `Running` state, VMware Tools is already installed — skip to [Step 3](#step-3--configure-the-dual-nic-network-adapters).

2. If the service doesn't exist, trigger installation manually:
   - **VM → Install VMware Tools** in the VMware Workstation menu (this mounts a virtual CD containing the installer).
   - 
     > <img width="510" height="383" alt="image" src="https://github.com/user-attachments/assets/b907bf07-ed46-40e6-8c0e-fc432e5108f5" />
   - Open the mounted drive in Windows Explorer, run `setup64.exe`.
   - 
     > <img width="510" height="383" alt="image" src="https://github.com/user-attachments/assets/8b3f1b52-c051-4150-a749-fe5aa27610a6" />
   - Click through the wizard with default options → **Install** → **Finish**.
   - 
     > <img width="250" height="200" alt="image" src="https://github.com/user-attachments/assets/322c9d7b-34d1-4a4d-8340-6bc39f6b6b51" />
   - Restart when prompted.

---

## Step 3 — Configure the dual-NIC network adapters

This baseline follows the [dual-interface network design](./02-network-architecture-planning.md) — every VM cloned from it inherits both adapters automatically.

1. Power off the VM.
2. **VM → Settings** — confirm **Network Adapter** is set to **NAT**. This is NIC 1.
3. **VM → Settings → Add… → Network Adapter** — set the new adapter to **Custom → VMnet1 (Host-only)**. This is NIC 2.
4. Power the VM back on.
5. Leave both adapters on **DHCP** for now — this baseline is never assigned a static IP itself. Static IPs are configured per-VM after cloning, in each VM's own build document.

---

## Step 4 — Enable .NET Framework 3.5 via DISM

Windows Server 2025 does not include .NET Framework 3.5 by default, and — unlike .NET 4.x — it cannot always be enabled purely through Windows Update in an offline/lab environment. Installing it from the mounted ISO's source files avoids depending on internet access for this specific feature.

1. With the installation ISO still attached (or re-attach it: **VM → Removable Devices → CD/DVD → Settings → Use ISO image file**), open an elevated PowerShell prompt.
2. Confirm the ISO's drive letter (commonly `D:`), then run:

```powershell
Dism /Online /Enable-Feature /FeatureName:NetFx3 /All /Source:D:\sources\sxs /LimitAccess
```

3. Confirm the operation completes with **"The operation completed successfully."**
4. Verify:

```powershell
Get-WindowsFeature -Name NET-Framework-Core
```

---

## Step 5 — Convert Evaluation to Datacenter edition via DISM Set-Edition

The ISO downloaded in [`01-iso-acquisition-and-verification.md`](./01-iso-acquisition-and-verification.md) is the free Evaluation edition, valid for 180 days. Converting it to a licensed retail edition removes that expiry and matches it to the GVLK used for activation in the next step.

> **Do this before installing any Windows Update.** `DISM /Set-Edition` runs a two-phase operation: it stages the new edition package online, then finalizes it offline during the reboot, before the desktop loads. That offline phase depends on the image's built-in servicing stack matching the exact build the ISO shipped with. If Windows Update has already patched the servicing stack or other components first, the offline finalize step can fail to load required files (commonly surfacing in `CBS.log` as `ERROR_MOD_NOT_FOUND` on a file like `cmsofflineservicing.dll`, with a visible version mismatch between the servicing stack and image version) — causing Windows to silently roll the edition change back on the next boot ("We couldn't complete the upgrade. No need to worry — undoing changes."), even though `Get-Volume` shows plenty of free disk space and the online DISM output itself reported success. Converting the edition on the pristine, freshly installed image — before any patching — avoids this version drift entirely.

1. Check the current edition:

```powershell
DISM /Online /Get-CurrentEdition
```

2. List the editions this installation can convert to:

```powershell
DISM /Online /Get-TargetEditions
```
Confirm `ServerDatacenter` appears in the list.

3. Convert to Datacenter, supplying the GVLK directly so the edition change and product key are applied together:

```powershell
DISM /Online /Set-Edition:ServerDatacenter /ProductKey:D764K-2NDRG-47T6Q-P8T8W-YP6DF /AcceptEula
```

4. The system reboots automatically to apply the edition change. **Wait for it to fully come back up and log in again before proceeding to Step 6** — running activation commands before this reboot completes is the single most common cause of the error below.

5. Confirm the edition change actually took effect before moving on:

```powershell
DISM /Online /Get-CurrentEdition
```
Expect `ServerDatacenter` with no `Eval` suffix. If it still shows `ServerDatacenterEval`, the conversion did not complete — repeat step 3 and make sure the reboot fully finishes this time.

> **Common error if this step is skipped or rushed:** running `slmgr /ipk` while still on the Evaluation edition (or before its reboot has completed) fails with `Error: 0xC004F069` / "The Software Licensing Service reported that the product SKU is not found." This is not a key or KMS problem — it means the edition conversion above hasn't actually taken effect yet. Re-run `DISM /Online /Get-CurrentEdition` to confirm, complete the reboot, and only then proceed to Step 6.

---

## Step 6 — Activate via GVLK against the online KMS host

This lab activates against an external, already-existing KMS host — see the [README's License activation section](../README.md#license-activation) for the full explanation of what a GVLK is and does.

1. Confirm the VM can reach the KMS host over NIC 1 (NAT) before attempting activation:

```powershell
Test-NetConnection kms.srv.crsoo.com -Port 1688
```
Expect `TcpTestSucceeded : True`. If this fails, activation will fail too — check NIC 1's internet connectivity and DNS resolution first.

2. Set the product key (redundant if already applied via `Set-Edition` above, but explicit and safe to repeat):

```powershell
slmgr /ipk D764K-2NDRG-47T6Q-P8T8W-YP6DF
```

3. Point activation at the KMS host:

```powershell
slmgr /skms kms.srv.crsoo.com:1688
```

4. Activate:

```powershell
slmgr /ato
```

5. Verify:

```powershell
slmgr /xpr
```
Expect output indicating the machine is permanently activated (KMS-activated Windows installations typically show a renewal interval rather than a fixed expiry — this is expected and normal for volume licensing).

---

## Step 7 — Apply Windows Update baseline

Windows Update runs over NIC 1 (NAT), which already has internet access. This step now runs **after** edition conversion and activation ([Step 5](#step-5--convert-evaluation-to-datacenter-edition-via-dism-set-edition), [Step 6](#step-6--activate-via-gvlk-against-the-online-kms-host)) — patching a properly licensed, correctly edited image avoids the servicing-stack version drift explained in Step 5's note.

1. **Settings → Windows Update → Check for updates**.
2. Install all available updates, rebooting as needed, until "You're up to date" appears.
3. This step patches known vulnerabilities in the base OS before the image is sealed and reused across every Windows Server VM in this lab.

---

## Step 8 — Clean up and seal the image

1. Remove the Windows Update component store cleanup residue:

```powershell
Dism /Online /Cleanup-Image /StartComponentCleanup
```

2. Empty the Recycle Bin, clear temp files if desired (optional, cosmetic).
3. Detach the ISO: **VM → Removable Devices → CD/DVD → Settings → uncheck Connected**.
4. Open an elevated Command Prompt and run Sysprep to generalize the image, ready for cloning:

```cmd
C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown
```

5. Wait for the VM to shut itself down automatically once sysprep completes — do **not** power it back on after this point until you're cloning it for a specific VM (powering on a generalized image without immediately going through OOBE setup can cause sysprep to need to be re-run).

---

## Step 9 — Convert to a reusable template

VMware Workstation doesn't have a dedicated "template" object like vSphere — the convention here is simply:

1. Leave the VM powered off in its sealed, generalized state.
2. (Recommended) Take a **Snapshot** named `sealed-baseline` before ever powering it on again, so you always have a known-good rollback point even if a future clone attempt accidentally boots this VM instead of a clone.
3. Treat this `.vmx` as read-only from this point forward — all future work happens on **Full Clones** made from it, never on this VM directly.

---

## Cloning this baseline later

When building DC01, WINAPP01, or any future Windows Server VM in this lab:

1. Right-click `GoldenBaseline-WinServer2025` in the VMware Library → **Manage → Clone…**
2. Choose **Clone from: The current state in the virtual machine**.
3. Choose **Create a full clone** (not linked — a full clone has no dependency on the baseline VM's files continuing to exist, which matters if you ever archive or move the baseline later).
4. Name the clone per the [naming convention](./02-network-architecture-planning.md#vm-and-hostname-naming-convention) — e.g. `DC01_10.10`.
5. On first boot of the clone, Windows runs through OOBE (Out-of-Box Experience) since the image was generalized — set the local Administrator password again, and proceed with that VM's own build document (e.g. [`06-dc01-active-directory-dns-dhcp.md`](./06-dc01-active-directory-dns-dhcp.md)) from there, including setting the OS hostname (`HOST` only, no IP suffix) and static IP.

---

## Next step

Continue to [`05-golden-baseline-rocky-linux-10.md`](./05-golden-baseline-rocky-linux-10.md) to build the equivalent sealed template for Rocky Linux 10, or skip ahead to [`06-dc01-active-directory-dns-dhcp.md`](./06-dc01-active-directory-dns-dhcp.md) if you're ready to clone this baseline into DC01 now.
