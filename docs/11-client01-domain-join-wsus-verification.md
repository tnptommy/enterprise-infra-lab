# 11 — CLIENT01: Domain Join and WSUS Verification

This document builds **CLIENT01**, a Windows 11 workstation, and uses it to verify the `Workstations` side of the GPO/WSUS setup from [`10`](./10-gpo-wsus-client-policy.md) end-to-end — the counterpart to WINAPP01 verifying the `Servers` side.

CLIENT01 is the only Windows 11 VM in this lab, so it's built directly from ISO rather than through a golden baseline (a baseline only pays off when cloning more than one VM from it — see [`04`](./04-golden-baseline-windows-server-2025.md#why-a-golden-baseline) and [`05`](./05-golden-baseline-rocky-linux-10.md#why-a-golden-baseline) for where that pattern is actually used).

---

## Table of contents

- [VM specification](#vm-specification)
- [Step 1 — Create the VM and install Windows 11](#step-1--create-the-vm-and-install-windows-11)
- [Step 2 — Activate via GVLK](#step-2--activate-via-gvlk)
- [Step 3 — Configure the dual-NIC network adapters](#step-3--configure-the-dual-nic-network-adapters)
- [Step 4 — Set hostname and confirm DHCP](#step-4--set-hostname-and-confirm-dhcp)
- [Step 5 — Domain-join](#step-5--domain-join)
- [Step 6 — Move CLIENT01 into the Workstations OU](#step-6--move-client01-into-the-workstations-ou)
- [Step 7 — Force policy refresh and verify WSUS targeting](#step-7--force-policy-refresh-and-verify-wsus-targeting)
- [Step 8 — Test cross-VM access](#step-8--test-cross-vm-access)
- [Step 9 — Final verification checklist](#step-9--final-verification-checklist)
- [Next step](#next-step)

---

## VM specification

| Setting | Value |
|---|---|
| VM name (VMware Library) | `CLIENT01_dhcp` |
| OS hostname | `CLIENT01` |
| vCPU | 2 |
| RAM | 2–4 GB |
| Disk | 40 GB (single disk) |
| NIC 1 (NAT) | DHCP |
| NIC 2 (Host-only) | DHCP — leased from DC01's `Clients` scope, `192.168.10.100`–`200` |

---

## Step 1 — Create the VM and install Windows 11

1. In VMware Workstation: **File → New Virtual Machine → Custom (advanced)**.
2. Guest OS: **Microsoft Windows**, version **Windows 11**.
3. Point the installer to `Windows11-24H2.iso` from [`01-iso-acquisition-and-verification.md`](./01-iso-acquisition-and-verification.md).
4. Name the VM `CLIENT01_dhcp` and choose a storage location.
5. Allocate resources per the [VM specification](#vm-specification) above.
6. Complete the wizard and power on the VM.
7. Work through Windows 11 setup: language/region, **I don't have internet** (or connect if you prefer, then skip Microsoft account) → **Continue with limited setup** → create a local account for now (domain-join replaces the need for this later, but setup requires an account to complete).
8. Once at the desktop, verify VMware Tools installed automatically (same as the [Windows Server golden baseline note](./04-golden-baseline-windows-server-2025.md#step-2--verify-vmware-tools)):
```powershell
Get-Service -Name VMTools
```

---

## Step 2 — Activate via GVLK

```powershell
slmgr /ipk W269N-WFGWX-YVC9B-4J6C9-T83GX
slmgr /skms active.orientsoftware.asia:1688
slmgr /ato
slmgr /xpr
```
> <img width="1120" height="676" alt="image" src="https://github.com/user-attachments/assets/57b1be83-1bf2-4843-9eb0-3d9a70984b45" />

See the [README's License activation section](../README.md#license-activation) for what this GVLK is and why it's safe to use here.

---

## Step 3 — Configure the dual-NIC network adapters

Same pattern as every other VM in this lab — see [`02-network-architecture-planning.md`](./02-network-architecture-planning.md#design-overview) for the full rationale.

1. Power off the VM.
2. **VM → Settings** — confirm the existing adapter is set to **NAT**. This is NIC 1.
3. **VM → Settings → Add… → Network Adapter** — set the new adapter to **Custom → VMnet1 (Host-only)**. This is NIC 2.
> <img width="575" height="582" alt="image" src="https://github.com/user-attachments/assets/3457b70e-0a5a-43b2-9fe5-ac2c07946387" />
4. Power the VM back on.
5. Rename both adapters for clarity, matching every other Windows VM in this lab:

```powershell
Get-NetAdapter
Rename-NetAdapter -Name "Ethernet0" -NewName "NAT-Internet"
Rename-NetAdapter -Name "Ethernet1" -NewName "Internal-LabNet"
```

6. Leave both on **DHCP** — unlike DC01 and WINAPP01, CLIENT01 doesn't get a static IP; it leases one from DC01's `Clients` scope automatically.

7. Disable NIC 1's auto-DNS, same as every other VM (prevents VMware's NAT DNS from interfering once DC01 becomes the resolver on NIC 2):
```powershell
nmcli con show 2>$null  # (ignore — this is the Linux command shown for reference; Windows equivalent below)
Get-NetIPInterface -InterfaceAlias "NAT-Internet"
Set-DnsClient -InterfaceAlias "NAT-Internet" -RegisterThisConnectionsAddress $false
```

---

## Step 4 — Set hostname and confirm DHCP

```powershell
Rename-Computer -NewName "CLIENT01" -Restart
```

After reboot, confirm NIC 2 received a lease from DC01's scope:
```powershell
ipconfig /all
```
> <img width="456" height="389" alt="image" src="https://github.com/user-attachments/assets/64c175bc-a0f2-4664-9ec2-e8d3c2ed30a3" />

Expect an address in `192.168.10.100`–`200` on `Internal-LabNet`, with DNS server `192.168.10.10` and connection-specific DNS suffix `corp-lab.com.vn` — both pushed automatically via the DHCP scope options configured in [`06`'s Step 6](./06-dc01-active-directory-dns-dhcp.md#step-6--install-and-configure-dhcp).

If the DNS suffix or server is missing, release/renew:
```powershell
ipconfig /release "Internal-LabNet"
ipconfig /renew "Internal-LabNet"
```

---

## Step 5 — Domain-join

1. **Settings → System → About → Domain or workgroup** (or **Control Panel → System**) → **Change settings** → **Change…**.
2. Select **Domain**, enter `corp-lab.com.vn` → **OK**.
3. Enter credentials: `CORP-LAB\Administrator` and its password (from [KeePass](./03-remote-access-tooling-setup.md#keepass)).
4. Accept the welcome message → **OK** → **Restart Now**.

**PowerShell equivalent (optional):**
```powershell
Add-Computer -DomainName "corp-lab.com.vn" -Credential (Get-Credential "CORP-LAB\Administrator") -Restart
```

After reboot, log in with a domain account (`CORP-LAB\Administrator`, or create a dedicated user later).

---

## Step 6 — Move CLIENT01 into the Workstations OU

Same reasoning as [`10`'s Step 1](./10-gpo-wsus-client-policy.md#step-1--move-winapp01-into-the-servers-ou) — domain-joined computers land in the default **Computers** container, not any OU, and GPOs can't be linked to that container.

On **DC01**:

1. **Server Manager → Tools → Active Directory Users and Computers**.
2. Expand `corp-lab.com.vn` → click **Computers** → confirm `CLIENT01` is listed there.
3. Drag `CLIENT01` onto the **Workstations** OU (or right-click → **Move…** → select **Workstations** → **OK**).
> <img width="641" height="395" alt="image" src="https://github.com/user-attachments/assets/bc9296cc-cb53-4da9-9673-45a7e88fe731" />

**PowerShell equivalent (optional, run on DC01):**
```powershell
Get-ADComputer "CLIENT01" | Move-ADObject -TargetPath "OU=Workstations,DC=corp-lab,DC=com,DC=vn"
```

---

## Step 7 — Force policy refresh and verify WSUS targeting

On **CLIENT01**:

```powershell
gpupdate /force
gpresult /r /scope:computer
```
Confirm **Applied Group Policy Objects** includes `Baseline-Policy` and `WSUS-Workstations-Targeting`.

```powershell
Get-ItemProperty "HKLM:\Software\Policies\Microsoft\Windows\WindowsUpdate" | Select-Object WUServer, WUStatusServer, TargetGroup, TargetGroupEnabled
```
Expect `TargetGroup` showing `Workstations`.

Trigger a detection cycle:
```powershell
Get-ScheduledTask -TaskName "Schedule Scan" -TaskPath "\Microsoft\Windows\UpdateOrchestrator\" | Start-ScheduledTask
```

> Unlike WINAPP01 (which had to contact its own WSUS instance running on itself — the scenario that ran into the NTLM loopback check documented while debugging [`09`](./09-winapp01-iis-sql-wsus.md)), CLIENT01 is a genuinely separate machine from WINAPP01, so that specific issue doesn't apply here. If CLIENT01 still doesn't appear in WSUS after a few minutes, revisit the general troubleshooting steps from that same session — checking `Get-WinEvent -LogName "Microsoft-Windows-WindowsUpdateClient/Operational"` for errors, confirming `wuauserv`/`UsoSvc` are running, and confirming the WSUS Reporting Web Service responds — before assuming something new is wrong.

On **WINAPP01**, open the WSUS console and confirm:
1. **Computers → All Computers → Workstations**.
2. `CLIENT01` should appear here within a few minutes.

---

## Step 8 — Test cross-VM access

Confirm the domain, DNS, and file shares set up across earlier documents all work from a genuinely different machine — everything up to now has largely been tested from the machine that configured it.

**Reach WEB01's website by name:**
```powershell
Invoke-WebRequest -Uri "http://web01.corp-lab.com.vn" -UseBasicParsing | Select-Object StatusCode
```

**Reach WINAPP01's file share:**
```powershell
Test-Path "\\WINAPP01\CorpShare"
```
<img width="836" height="443" alt="image" src="https://github.com/user-attachments/assets/ecf1ac65-9d99-4a05-9c3e-35bc64063074" />


**Reach DC01's Storage Spaces share** (if a share was created on the `D:` volume in [`06`'s Step 8](./06-dc01-active-directory-dns-dhcp.md#step-8--storage-spaces-demo) — otherwise skip, since that step only demonstrated the volume itself, not a share on it):
```powershell
Test-Path "\\DC01\D$"
```

**Confirm domain DNS resolves correctly:**
```powershell
nslookup web01.corp-lab.com.vn
nslookup winapp01.corp-lab.com.vn
```
> <img width="386" height="173" alt="image" src="https://github.com/user-attachments/assets/81f2f038-18a9-4825-af14-ebc1591464cf" />

---

## Step 9 — Final verification checklist

1. **Domain-joined correctly:**
```powershell
Get-ComputerInfo | Select-Object CsDomain, CsDomainRole
```

2. **CLIENT01 is in the correct OU:**
```powershell
Get-ADComputer "CLIENT01" | Select-Object DistinguishedName
```
Run this on DC01 — expect `OU=Workstations,...`.

3. **GPO applied correctly** (repeat the check from Step 7).

4. **WSUS shows CLIENT01 under Workstations** (repeat the check from Step 7).

5. **Time syncs from the domain hierarchy** (no separate NTP configuration needed — see [`06`'s Step 7](./06-dc01-active-directory-dns-dhcp.md#step-7--configure-ntp)):
```powershell
w32tm /query /status
```
Confirm the source is the domain hierarchy, not `Local CMOS Clock`.

6. **Cross-VM access works** (repeat the checks from Step 8).

7. **Credentials are stored, not memorized** — confirm the local account password (if kept) and any domain account used are noted in [KeePass](./03-remote-access-tooling-setup.md#keepass) — CLIENT01 doesn't need its own KeePass group unless a dedicated local account was created, since it authenticates as whichever domain user logs in.

If all seven checks pass, CLIENT01 is ready, and both sides (`Servers` and `Workstations`) of the GPO/WSUS setup from [`10`](./10-gpo-wsus-client-policy.md) are now confirmed working end-to-end.

---

## Next step

Continue to [`12-mon01-zabbix-server-configuration.md`](./12-mon01-zabbix-server-configuration.md) to build MON01 and begin the monitoring stack.
