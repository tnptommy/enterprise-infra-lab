# 10 — GPO: WSUS Client Policy

This document populates the `Baseline-Policy` GPO created (but left empty) in [`06`'s Step 12](./06-dc01-active-directory-dns-dhcp.md#step-11--ou-structure-and-baseline-gpos) with settings that point every domain-joined machine at the WSUS server built in [`09`](./09-winapp01-iis-sql-wsus.md), and adds client-side targeting so machines automatically land in the correct WSUS Computer Group (`Servers` or `Workstations`) without manual sorting in the WSUS console.

All of this is done from **DC01**, using Group Policy Management — no changes happen on WINAPP01 or CLIENT01 directly; they simply pick up these settings the next time Group Policy refreshes.

---

## Table of contents

- [Step 1 — Move WINAPP01 into the Servers OU](#step-1--move-winapp01-into-the-servers-ou)
- [Step 2 — Point clients at the WSUS server](#step-2--point-clients-at-the-wsus-server)
- [Step 3 — Configure the automatic update schedule](#step-3--configure-the-automatic-update-schedule)
- [Step 4 — Client-side targeting for Servers](#step-4--client-side-targeting-for-servers)
- [Step 5 — Client-side targeting for Workstations](#step-5--client-side-targeting-for-workstations)
- [Step 6 — Force policy refresh and update detection on WINAPP01](#step-6--force-policy-refresh-and-update-detection-on-winapp01)
- [Step 7 — Verify in the WSUS console](#step-7--verify-in-the-wsus-console)
- [Step 8 — Final verification checklist](#step-8--final-verification-checklist)
- [Next step](#next-step)

---

## Step 1 — Move WINAPP01 into the Servers OU

Domain-joined computers land in the default **Computers** container, not any custom OU — and GPOs can't be linked to that container at all. Without this step, `Baseline-Policy` and every GPO created below would never actually apply to WINAPP01, despite being linked to the `Servers` OU.

1. On DC01: **Server Manager → Tools → Active Directory Users and Computers**.
2. Expand `corp-lab.com.vn` → click the **Computers** container → confirm `WINAPP01` is listed there.
3. Drag `WINAPP01` onto the **Servers** OU (or right-click → **Move…** → select **Servers** → **OK**).
4. Confirm: right-click **Servers** OU → it should now list `WINAPP01`.
> <img width="641" height="394" alt="image" src="https://github.com/user-attachments/assets/55883046-a6d8-4d15-840e-7b62d59d1479" />

Repeat this step for every future Windows Server VM in this lab, and do the equivalent for Windows 11 client machines into the **Workstations** OU once [`11-client01-domain-join-wsus-verification.md`](./11-client01-domain-join-wsus-verification.md) joins CLIENT01 to the domain.

**PowerShell equivalent (optional):**
```powershell
Get-ADComputer "WINAPP01" | Move-ADObject -TargetPath "OU=Servers,DC=corp-lab,DC=com,DC=vn"
```

---

## Step 2 — Point clients at the WSUS server

1. **Server Manager → Tools → Group Policy Management**.
2. Expand `corp-lab.com.vn` → **Group Policy Objects** → right-click **Baseline-Policy** → **Edit**.
3. Navigate to **Computer Configuration → Policies → Administrative Templates → Windows Components → Windows Update → Manage updates offered from Windows Server Update Service**.
4. Double-click **Specify intranet Microsoft update service location** → **Enabled**.
5. Set both fields to WINAPP01's default WSUS HTTP port (`8530`):
   - **Set the intranet update service for detecting updates**: `http://winapp01.corp-lab.com.vn:8530`
   - **Set the intranet statistics server**: `http://winapp01.corp-lab.com.vn:8530`

     > <img width="1096" height="369" alt="image" src="https://github.com/user-attachments/assets/6eca2951-1c5d-4664-be0b-e6c64bbf4fc9" />

7. **OK**.

---

## Step 3 — Configure the automatic update schedule

Still in the `Baseline-Policy` editor:

1. Navigate to **Computer Configuration → Policies → Administrative Templates → Windows Components → Windows Update → Manage end user experience**.
2. Double-click **Configure Automatic Updates** → **Enabled**.
3. **Configure automatic updating**: option **4 - Auto download and schedule the install**.
4. **Scheduled install day**: `0 - Every day`. **Scheduled install time**: `03:00` (matches the WSUS synchronization schedule set in [`09`'s Step 9](./09-winapp01-iis-sql-wsus.md#step-9--configure-wsus), so clients install shortly after new updates are approved).
> <img width="1107" height="346" alt="image" src="https://github.com/user-attachments/assets/7861bf23-6694-42fb-b60b-12e10c332dd4" />

5. **OK**. Close the Group Policy Management Editor.

---

## Step 4 — Client-side targeting for Servers

Client-side targeting automatically places a computer into the matching WSUS Computer Group based on which GPO applies to it — this is what makes the `Servers`/`Workstations` groups created in [`09`'s Step 9](./09-winapp01-iis-sql-wsus.md#step-9--configure-wsus) actually populate themselves, instead of every machine landing in WSUS's default **Unassigned Computers** group.

This needs its own GPO (rather than living in the shared `Baseline-Policy`) because the target group name differs between servers and workstations, even though both share the same WSUS server location and update schedule from Steps 2–3.

1. **Group Policy Management** → right-click **Group Policy Objects → New**. Name: `WSUS-Servers-Targeting` → **OK**.
2. Right-click it → **Edit**.
3. Navigate to **Computer Configuration → Policies → Administrative Templates → Windows Components → Windows Update → Manage updates offered from Windows Server Update Service**.
4. Double-click **Enable client-side targeting** → **Enabled**. **Target group name for this computer**: `Servers`.
> <img width="683" height="279" alt="image" src="https://github.com/user-attachments/assets/68b9240d-c47b-458d-b578-cbd4889457d2" />

5. **OK**. Close the editor.
6. Right-click the **Servers** OU → **Link an Existing GPO…** → select `WSUS-Servers-Targeting` → **OK**.

---

## Step 5 — Client-side targeting for Workstations

Identical process, targeting the `Workstations` group instead:

1. **Group Policy Objects → New**. Name: `WSUS-Workstations-Targeting` → **OK**.
2. **Edit** → same path as Step 4 → **Enable client-side targeting** → **Enabled** → **Target group name for this computer**: `Workstations`.
> <img width="514" height="280" alt="image" src="https://github.com/user-attachments/assets/79392f37-750f-459e-8c20-92d3b7f1753a" />

3. **OK**. Close the editor.
4. Right-click the **Workstations** OU → **Link an Existing GPO…** → select `WSUS-Workstations-Targeting` → **OK**.

**PowerShell equivalent for Steps 2–5 (optional):**
```powershell
# Baseline-Policy: WSUS location + update schedule
Set-GPRegistryValue -Name "Baseline-Policy" -Key "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate" -ValueName "WUServer" -Type String -Value "http://winapp01.corp-lab.com.vn:8530"
Set-GPRegistryValue -Name "Baseline-Policy" -Key "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate" -ValueName "WUStatusServer" -Type String -Value "http://winapp01.corp-lab.com.vn:8530"
Set-GPRegistryValue -Name "Baseline-Policy" -Key "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate\AU" -ValueName "UseWUServer" -Type DWord -Value 1
Set-GPRegistryValue -Name "Baseline-Policy" -Key "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate\AU" -ValueName "AUOptions" -Type DWord -Value 4
Set-GPRegistryValue -Name "Baseline-Policy" -Key "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate\AU" -ValueName "ScheduledInstallDay" -Type DWord -Value 0
Set-GPRegistryValue -Name "Baseline-Policy" -Key "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate\AU" -ValueName "ScheduledInstallTime" -Type DWord -Value 3

# WSUS-Servers-Targeting
New-GPO -Name "WSUS-Servers-Targeting"
Set-GPRegistryValue -Name "WSUS-Servers-Targeting" -Key "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate" -ValueName "TargetGroupEnabled" -Type DWord -Value 1
Set-GPRegistryValue -Name "WSUS-Servers-Targeting" -Key "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate" -ValueName "TargetGroup" -Type String -Value "Servers"
New-GPLink -Name "WSUS-Servers-Targeting" -Target "OU=Servers,DC=corp-lab,DC=com,DC=vn"

# WSUS-Workstations-Targeting
New-GPO -Name "WSUS-Workstations-Targeting"
Set-GPRegistryValue -Name "WSUS-Workstations-Targeting" -Key "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate" -ValueName "TargetGroupEnabled" -Type DWord -Value 1
Set-GPRegistryValue -Name "WSUS-Workstations-Targeting" -Key "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate" -ValueName "TargetGroup" -Type String -Value "Workstations"
New-GPLink -Name "WSUS-Workstations-Targeting" -Target "OU=Workstations,DC=corp-lab,DC=com,DC=vn"
```

---

## Step 6 — Force policy refresh and update detection on WINAPP01

WINAPP01 is the only machine in the `Servers` OU built so far — use it to test everything above end-to-end without waiting for a future document.

On **WINAPP01**:

```powershell
gpupdate /force
```

Then force an immediate Windows Update detection cycle (rather than waiting for the default periodic check):
```powershell
UsoClient StartScan
```

---

## Step 7 — Verify in the WSUS console

On **WINAPP01**, confirm the GPO settings actually landed in the registry:
```powershell
gpresult /r
Get-ItemProperty "HKLM:\Software\Policies\Microsoft\Windows\WindowsUpdate" | Select-Object WUServer, WUStatusServer, TargetGroup, TargetGroupEnabled
```
> <img width="859" height="107" alt="image" src="https://github.com/user-attachments/assets/1fdfe9e5-7895-4e79-bf9b-d70d396b1d52" />

Expect `WUServer`/`WUStatusServer` showing `http://winapp01.corp-lab.com.vn:8530`, `TargetGroup` showing `Servers`, `TargetGroupEnabled` showing `1`.

On **DC01** (or via SSMS/WSUS console), confirm WINAPP01 appears where expected:

1. **Server Manager → Tools → Windows Server Update Services** on WINAPP01.
2. Expand **Computers → All Computers → Servers**.
3. `WINAPP01` should appear here within a few minutes of the detection cycle in Step 6 — if it's still sitting under **Unassigned Computers** instead, double-check Step 7's registry values and re-run `gpupdate /force`.

---

## Step 8 — Final verification checklist

1. **WINAPP01 is in the correct OU:**
```powershell
Get-ADComputer "WINAPP01" | Select-Object DistinguishedName
```
Expect it to show `OU=Servers,...` in the path.

2. **Baseline-Policy is linked and contains the WSUS settings** — Group Policy Management → `Servers` OU → **Linked Group Policy Objects** tab → confirm `Baseline-Policy` and `WSUS-Servers-Targeting` both appear, both **Enabled**.

3. **Registry values landed correctly on WINAPP01** (repeat the check from Step 7).

4. **WINAPP01 shows up in the correct WSUS Computer Group** (repeat the check from Step 7).

5. **No GPO processing errors:**
```powershell
gpresult /r /scope:computer
```
Confirm `Baseline-Policy` and `WSUS-Servers-Targeting` are listed under **Applied Group Policy Objects**, not **Denied Group Policy Objects**.

If all five checks pass, WSUS client policy is fully wired up for the `Servers` OU — the `Workstations` side of this same setup will be verified once CLIENT01 exists.

---

## Next step

Continue to [`11-client01-domain-join-wsus-verification.md`](./11-client01-domain-join-wsus-verification.md) to build CLIENT01 and confirm the `Workstations` side of this GPO setup end-to-end.
