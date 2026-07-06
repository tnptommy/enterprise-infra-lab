# 09 — WINAPP01: IIS, SQL Server, WSUS

This document clones the Windows Server 2025 [Golden Baseline](./04-golden-baseline-windows-server-2025.md) into **WINAPP01** and builds it into this lab's application server — IIS, SQL Server, WSUS, and the general-purpose file share that was deliberately kept off DC01 (see [`06`'s note](./06-dc01-active-directory-dns-dhcp.md#step-8--storage-spaces-demo) on why a domain controller shouldn't also host application/file services).

Every step below is written **GUI-first**, per this guide's convention for Windows Server — PowerShell equivalents follow where useful for later automation.

| Component | Version used | Source |
|---|---|---|
| SQL Server | 2025 (17.x), Enterprise Developer Edition — free for non-production use | https://www.microsoft.com/en-us/sql-server/sql-server-downloads |
| SQL Server Management Studio (SSMS) | 22.x (latest) | https://learn.microsoft.com/en-us/ssms/install/install |

---

## Table of contents

- [VM specification](#vm-specification)
- [Step 1 — Clone the golden baseline and provision disks](#step-1--clone-the-golden-baseline-and-provision-disks)
- [Step 2 — Set hostname and static IP](#step-2--set-hostname-and-static-ip)
- [Step 3 — Domain-join before installing any role](#step-3--domain-join-before-installing-any-role)
- [Step 4 — Create the SQL Server service account](#step-4--create-the-sql-server-service-account)
- [Step 5 — Install IIS](#step-5--install-iis)
- [Step 6 — Install SQL Server 2025](#step-6--install-sql-server-2025)
- [Step 7 — Install SQL Server Management Studio](#step-7--install-sql-server-management-studio)
- [Step 8 — Install WSUS](#step-8--install-wsus)
- [Step 9 — Configure WSUS](#step-9--configure-wsus)
- [Step 10 — File Server role and SMB share demo](#step-10--file-server-role-and-smb-share-demo)
- [Step 11 — Remote access and hardening baseline](#step-11--remote-access-and-hardening-baseline)
- [Step 12 — Final verification checklist](#step-12--final-verification-checklist)
- [Next step](#next-step)

---

## VM specification

| Setting | Value |
|---|---|
| VM name (VMware Library) | `WINAPP01_10.15` |
| OS hostname | `WINAPP01` |
| vCPU | 4 |
| RAM | 8–10 GB |
| Disk 1 (OS) | 60 GB |
| Disk 2 (data) | 40 GB — SQL Server data/log files, and the file share created in this document |
| NIC 1 (NAT) | DHCP |
| NIC 2 (Host-only) | Static — `192.168.10.15` |

---

## Step 1 — Clone the golden baseline and provision disks

Follow [`04-golden-baseline-windows-server-2025.md`'s cloning instructions](./04-golden-baseline-windows-server-2025.md#cloning-this-baseline-later):

1. Right-click `GoldenBaseline-WinServer2025` → **Manage → Clone…** → **Create a full clone**.
2. Name the clone `WINAPP01_10.15`.
3. Adjust vCPU/RAM in **VM → Settings** per the [VM specification](#vm-specification) above.
4. With the VM still powered off, add the second disk: **VM → Settings → Add… → Hard Disk → Create a new virtual disk**, size **40 GB**.
5. Power on the VM.

---

## Step 2 — Set hostname and static IP

Identical process to [DC01's Step 3](./06-dc01-active-directory-dns-dhcp.md#step-3--set-hostname-and-static-ip), with this VM's own values:

1. **Server Manager → Local Server** → click the computer name → **Change…** → set `WINAPP01` → restart.
2. Rename both network adapters for clarity, matching the convention used on every Windows VM in this lab: `NAT-Internet` (NIC 1) and `Internal-LabNet` (NIC 2).
3. Set `Internal-LabNet`'s IPv4 to static: address `192.168.10.15`, subnet mask `255.255.255.0`, gateway blank, DNS `192.168.10.10` (DC01).

**PowerShell equivalent (optional):**
```powershell
Rename-Computer -NewName "WINAPP01" -Restart
# after reboot:
Get-NetAdapter
Rename-NetAdapter -Name "Ethernet0" -NewName "NAT-Internet"
Rename-NetAdapter -Name "Ethernet1" -NewName "Internal-LabNet"
New-NetIPAddress -InterfaceAlias "Internal-LabNet" -IPAddress 192.168.10.15 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias "Internal-LabNet" -ServerAddresses 192.168.10.10
```

---

## Step 3 — Domain-join before installing any role

Join the domain **now**, before IIS, SQL Server, or WSUS are installed — this ensures every role's default permissions, service account contexts, and later Group Policy application are all evaluated in the domain context from the start, rather than needing to be reconciled after the fact.

1. **Server Manager → Local Server** → click **Workgroup**.
2. **Computer Name/Domain Changes** → **Change…**.
> <img width="301" height="352" alt="image" src="https://github.com/user-attachments/assets/6259632e-6de2-46d1-820b-ecfe2aba908b" />

3. Select **Domain**, enter `corp-lab.com.vn` → **OK**.
> <img width="240" height="298" alt="image" src="https://github.com/user-attachments/assets/bb8e66fb-4156-411d-ab89-df45ee2e0c6f" />

4. Enter credentials: `CORP-LAB\Administrator` and its password (from [KeePass](./03-remote-access-tooling-setup.md#keepass)).
> <img width="338" height="273" alt="image" src="https://github.com/user-attachments/assets/aa601c34-51e2-4538-8691-866eab188bbc" />

5. Accept the welcome-to-the-domain message → **OK** → **Restart Now**.

**PowerShell equivalent (optional):**
```powershell
Add-Computer -DomainName "corp-lab.com.vn" -Credential (Get-Credential "CORP-LAB\Administrator") -Restart
```

After reboot, log in as `CORP-LAB\Administrator` (or another domain admin account) going forward.

---

## Step 4 — Create the SQL Server service account

SQL Server should run under a dedicated domain service account rather than a built-in identity (Local System/Network Service) — this is standard practice for auditing, delegation, and limiting blast radius if the service account is ever compromised.

Since Active Directory Users and Computers isn't installed on a member server by default, do this step from **DC01** instead:

1. On DC01: **Server Manager → Tools → Active Directory Users and Computers**.
2. Expand `corp-lab.com.vn` → right-click the **ServiceAccounts** OU (created in [`06`'s Step 11](./06-dc01-active-directory-dns-dhcp.md#step-11--ou-structure-and-baseline-gpos)) → **New → User**.
3. Full name: `svc-sqlserver`, User logon name: `svc-sqlserver` → **Next**.
4. Set a strong password, check **Password never expires**, uncheck **User must change password at next logon** (service accounts don't interactively log on to change passwords) → **Next** → **Finish**.
5. Store this password in [KeePass](./03-remote-access-tooling-setup.md#keepass) under a `WINAPP01_10.15` group entry named `svc-sqlserver`.

**PowerShell equivalent (optional, run on DC01):**
```powershell
New-ADUser -Name "svc-sqlserver" -SamAccountName "svc-sqlserver" `
    -UserPrincipalName "svc-sqlserver@corp-lab.com.vn" `
    -Path "OU=ServiceAccounts,DC=corp-lab,DC=com,DC=vn" `
    -AccountPassword (ConvertTo-SecureString "Your-Strong-Password-Here" -AsPlainText -Force) `
    -PasswordNeverExpires $true -Enabled $true
```

---

## Step 5 — Install IIS

1. On WINAPP01: **Server Manager → Manage → Add Roles and Features**.
2. **Next** through **Before You Begin**, **Installation Type: Role-based or feature-based → Next**, **Select destination server: this server → Next**.
3. **Server Roles**: check **Web Server (IIS)** → accept the prompt to add management tools → **Next** through **Features**, **Web Server Role (IIS)** intro, **Role Services** (defaults are fine for this lab) → **Confirmation → Install**.
4. Once complete, verify: open a browser on WINAPP01 and go to `http://localhost` — the default IIS welcome page should appear.

**PowerShell equivalent (optional):**
```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
```

---

## Step 6 — Install SQL Server 2025

1. Download the **Enterprise Developer edition** installer from https://www.microsoft.com/en-us/sql-server/sql-server-downloads (free for non-production/lab use — do not use this edition in any production deployment).
2. Run the downloaded bootstrapper → choose **Download Media** (keeps a reusable copy) or **Basic** for a quick default install. This guide uses **Custom** to control the service account:
   - From the SQL Server Installation Center, choose **Installation → New SQL Server stand-alone installation**.
3. **Product Key**: select **Developer** (free) if not already pre-filled → **Next**.
4. Accept license terms → **Next** through **Microsoft Update** (optional) → **Install Rules** (resolve any warnings shown) → **Next**.
5. **Feature Selection**: check **Database Engine Services** at minimum; add **Full-Text and Semantic Extractions** if you plan to use full-text search later → **Next**.
6. **Instance Configuration**: use **Default instance** (`MSSQLSERVER`) → **Next**.
7. **Server Configuration** — this is where the dedicated service account matters:
   - SQL Server Database Engine service account: `CORP-LAB\svc-sqlserver`, enter its password from KeePass.
   - Startup type: **Automatic**.
8. **Database Engine Configuration**:
   - **Authentication Mode**: **Mixed Mode** (allows both Windows and SQL authentication — useful for later app integrations that don't support Windows auth). Set a strong `sa` password, store it in KeePass.
   - **Specify SQL Server administrators**: add `CORP-LAB\Administrator` (or your own admin account).
   - **Data Directories** tab: point the data/log/tempdb directories at the second disk (e.g. `D:\SQLData`, `D:\SQLLogs`) rather than the OS disk — create these folders first if the installer doesn't do so automatically.
9. **Next** through remaining screens → **Install**.
10. Once complete, confirm the service is running: **Services** (`services.msc`) → **SQL Server (MSSQLSERVER)** should show **Running**.

**Open the firewall for SQL Server** (default instance uses TCP 1433):
```powershell
New-NetFirewallRule -DisplayName "SQL-Server-1433" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow
```

---

## Step 7 — Install SQL Server Management Studio

1. Download the SSMS bootstrapper from https://learn.microsoft.com/en-us/ssms/install/install (`vs_SSMS.exe`).
2. Run it as Administrator → accept the license terms → **Install**.
3. Once complete, launch **SQL Server Management Studio** → connect to `localhost` (or `WINAPP01`) using Windows Authentication to confirm connectivity.

---

## Step 8 — Install WSUS

1. **Server Manager → Manage → Add Roles and Features → Server Roles** → check **Windows Server Update Services**.
2. Accept the prompt to add required features/management tools → **Next**.
3. **Role Services**: ensure **WID Connectivity** is **unchecked** and **Database** is checked instead — this lab points WSUS at the SQL Server instance just installed rather than the default Windows Internal Database, giving hands-on practice connecting WSUS to an external SQL Server.
4. **Content location selection**: choose a path on the second disk, e.g. `D:\WSUS`.
5. **Confirmation → Install**.

---

## Step 9 — Configure WSUS

**Post-installation configuration:**

1. Click the notification flag → **Launch Post-Installation tasks**, or open **Server Manager → Tools → Windows Server Update Services** and let the **WSUS Server Configuration Wizard** run automatically on first launch.
2. When prompted for the database, specify the local SQL Server instance (`WINAPP01` or `WINAPP01\MSSQLSERVER`) — this is the step that binds WSUS to SQL Server instead of WID.
3. Continue through the wizard: **Upstream Server** (this is the only WSUS server in this lab, leave as **Synchronize from Microsoft Update**), **Proxy Server** (skip unless your network requires one).

**Products and Classifications:**

4. In the WSUS console, go to **Options → Products and Classifications**.
5. Under **Products**, limit selection to what this lab actually needs — check **Windows Server 2025** and **Windows 11** only (unchecking everything else keeps the WSUS content store from growing unnecessarily large, per the [README's note on this](../README.md#license-activation)).
6. Under **Classifications**, check **Critical Updates**, **Security Updates**, and **Updates** at minimum.

**Synchronization schedule:**

7. **Options → Synchronization Schedule** → set **Synchronize automatically**, once daily at a time that suits (e.g. 02:00).
8. **Synchronizations** view → **Synchronize Now** to run an initial sync.

**Computer Groups and Auto-Approval:**

9. In the left tree, right-click **All Computers** → **Add Computer Group…** → create groups matching this lab's OUs, e.g. `Servers` and `Workstations`.
10. **Options → Automatic Approvals** → create a rule: approve **Critical Updates** and **Security Updates** for the **Servers** and **Workstations** groups automatically, so machines don't sit unpatched waiting for manual approval in this lab context.

---

## Step 10 — File Server role and SMB share demo

This is the general-purpose file share deliberately **not** placed on DC01 (see [`06`'s note](./06-dc01-active-directory-dns-dhcp.md#step-8--storage-spaces-demo)) — WINAPP01, as the application/data server, is the appropriate place for it.

1. **Server Manager → Manage → Add Roles and Features → Server Roles** → check **File and Storage Services → File Server** (may already be installed — File Server is often included by default; verify rather than assume) → **Install**.
2. Format the second disk's remaining free space (beyond what SQL Server is using) if not already done, or create a dedicated folder for the share:
```powershell
New-Item -Path "D:\CorpShare" -ItemType Directory
```
3. **Server Manager → File and Storage Services → Shares → Tasks → New Share…**.
4. Choose **SMB Share - Quick**, path `D:\CorpShare`, share name `CorpShare`.
5. On the permissions page, grant **Full Control** to `CORP-LAB\Domain Admins` and **Modify** to `CORP-LAB\Domain Users` → **Create**.

Verify from another domain-joined machine once one exists (CLIENT01, [`11`](./11-client01-domain-join-wsus-verification.md)):
```
\\WINAPP01\CorpShare
```

**PowerShell equivalent (optional):**
```powershell
New-SmbShare -Name "CorpShare" -Path "D:\CorpShare" -FullAccess "CORP-LAB\Domain Admins" -ChangeAccess "CORP-LAB\Domain Users"
```

---

## Step 11 — Remote access and hardening baseline

**Confirm RDP is enabled** (this lab keeps the default port `3389` throughout, matching the choice made for [DC01](./06-dc01-active-directory-dns-dhcp.md#step-10--remote-access-configuration) — no custom port change needed here either):

1. **Server Manager → Local Server** → confirm **Remote Desktop** shows **Enabled**.
2. Confirm the built-in **Remote Desktop** firewall rule group is active: **Windows Defender Firewall → Advanced Settings** (`wf.msc`) → **Inbound Rules** → look for **Remote Desktop - User Mode (TCP-In)** showing **Enabled: Yes**.
3. Update the saved connection for `WINAPP01_10.15` in [mRemoteNG](./03-remote-access-tooling-setup.md#pre-building-the-mremoteng-connection-list) if it isn't already configured — port stays `3389`.

**Hardening baseline** — apply the same [Security Configuration and Analysis](./06-dc01-active-directory-dns-dhcp.md#step-9--hardening-baseline) process used for DC01, importing [`configs/dc01-baseline-security-template.inf`](../configs/dc01-baseline-security-template.inf) as a starting point (its password/lockout policy settings are equally applicable to a member server; skip re-reading the DC-specific GPO precedence note, since WINAPP01 is not a domain controller and its local policy isn't overridden the same way unless a GPO from `corp-lab.com.vn` specifically targets it).

---

## Step 12 — Final verification checklist

1. **Domain-joined correctly:**
```powershell
Get-ComputerInfo | Select-Object CsDomain, CsDomainRole
```

2. **IIS serving the default site:**
```powershell
Invoke-WebRequest -Uri http://localhost -UseBasicParsing | Select-Object StatusCode
```

3. **SQL Server running under the dedicated service account:**
```powershell
Get-Service MSSQLSERVER | Select-Object Status
Get-CimInstance Win32_Service -Filter "Name='MSSQLSERVER'" | Select-Object StartName
```
Confirm `StartName` shows `CORP-LAB\svc-sqlserver`, not a built-in identity.

4. **WSUS bound to SQL Server, not WID:**
Open WSUS console → **Options → WSUS Server Configuration Wizard** → confirm it references the SQL Server instance.

5. **WSUS has synchronized at least once:**
WSUS console → **Reports → Synchronization Results**, or check **Synchronizations** node for a completed run.

6. **File share reachable:**
```powershell
Test-Path "\\WINAPP01\CorpShare"
```
(Run from another domain-joined machine once available; on WINAPP01 itself this always resolves locally regardless of share permissions, so it's not a full test on its own.)

7. **Credentials are stored, not memorized** — confirm the local Administrator password, `svc-sqlserver` password, and `sa` password are all saved in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `WINAPP01_10.15` group.

If all seven checks pass, WINAPP01 is ready.

---

## Next step

Continue to [`10-gpo-wsus-client-policy.md`](./10-gpo-wsus-client-policy.md) to create the GPO on DC01 that points domain-joined clients at this WSUS server.
