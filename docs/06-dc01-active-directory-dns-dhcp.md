# 06 — DC01: Active Directory, DNS, DHCP

This document clones the Windows Server 2025 [Golden Baseline](./04-golden-baseline-windows-server-2025.md) into **DC01** and builds it into this lab's Active Directory Domain Controller, DNS server, DHCP server, and NTP time source — the foundation every other Windows VM (WINAPP01, CLIENT01) and every domain-joined Linux VM depends on.

Every step below is written **GUI-first** — Server Manager wizards, MMC snap-ins, and Control Panel dialogs — since that's how this is actually operated day-to-day. A short PowerShell equivalent follows each step for quick verification or later scripting/automation (see [`18-automation-backup-scheduling.md`](./18-automation-backup-scheduling.md)), but it's optional.

---

## Table of contents

- [VM specification](#vm-specification)
- [Step 1 — Clone the golden baseline](#step-1--clone-the-golden-baseline)
- [Step 2 — Add the Storage Spaces disk](#step-2--add-the-storage-spaces-disk)
- [Step 3 — Set hostname and static IP](#step-3--set-hostname-and-static-ip)
- [Step 4 — Install Active Directory Domain Services](#step-4--install-active-directory-domain-services)
- [Step 5 — Configure DNS](#step-5--configure-dns)
- [Step 6 — Install and configure DHCP](#step-6--install-and-configure-dhcp)
- [Step 7 — Configure NTP](#step-7--configure-ntp)
- [Step 8 — Storage Spaces demo](#step-8--storage-spaces-demo)
- [Step 9 — Hardening baseline](#step-9--hardening-baseline)
- [Step 10 — Remote access configuration](#step-10--remote-access-configuration)
- [Step 11 — OU structure and baseline GPOs](#step-11--ou-structure-and-baseline-gpos)
- [Step 12 — Final verification checklist](#step-12--final-verification-checklist)
- [Next step](#next-step)

---

## VM specification

| Setting | Value |
|---|---|
| VM name (VMware Library) | `DC01_10.10` |
| OS hostname | `DC01` |
| vCPU | 2 |
| RAM | 4–6 GB |
| Disk 1 (OS) | 60 GB |
| Disk 2 (data) | 40 GB — Storage Spaces demo (see [Step 2](#step-2--add-the-storage-spaces-disk)) |
| NIC 1 (NAT) | DHCP |
| NIC 2 (Host-only) | Static — `192.168.10.10` |
| Domain name used in this guide | `corp-lab.com.vn` *(substitute your own if you prefer — every subsequent document references it by this name)* |

---

## Step 1 — Clone the golden baseline

Follow [`04-golden-baseline-windows-server-2025.md`'s cloning instructions](./04-golden-baseline-windows-server-2025.md#cloning-this-baseline-later):

1. Right-click `GoldenBaseline-WinServer2025` in the VMware Library → **Manage → Clone…** → **Create a full clone**.
2. Name the clone `DC01_10.10`.
3. Adjust vCPU/RAM in **VM → Settings** per the [VM specification](#vm-specification) above.
4. Do **not** power on yet — add the second disk first (next step), since VMware Workstation only allows adding a disk while the VM is off.

---

## Step 2 — Add the Storage Spaces disk

1. With the clone still powered off: **VM → Settings → Add… → Hard Disk → Create a new virtual disk**.
2. Size: **40 GB**, thin provisioned, store as a single file (or split, either works for this lab).
3. Confirm the VM now shows two virtual disks in **VM → Settings**.
4. Power on the VM.

---

## Step 3 — Set hostname and static IP

1. Log in with the local Administrator account (set during golden baseline OOBE per [`04`](./04-golden-baseline-windows-server-2025.md#cloning-this-baseline-later)).

**Set the hostname** — **`HOST` only, no IP suffix**, per the [naming convention](./02-network-architecture-planning.md#vm-and-hostname-naming-convention):

2. Open **Server Manager** (opens automatically at login) → **Local Server** in the left pane.
3. Click the current computer name next to **Computer name**.
4. In the **System Properties** dialog, click **Change…**.
5. Enter `DC01` under **Computer name**, click **OK**, then **OK** again, then **Restart Now** when prompted.

**Assign the static IP to NIC 2 (Host-only):**

6. After reboot, open **Control Panel → Network and Internet → Network and Sharing Center → Change adapter settings** (or right-click the network icon in the system tray → **Open Network & Internet settings → Advanced network settings → More network adapter options**).
7. Two adapters are listed. Identify NIC 2 (Host-only) by opening each one's **Status → Details** — NIC 1 (NAT) shows a DHCP-assigned address in VMnet8's range; NIC 2 (Host-only) shows either an APIPA (`169.254.x.x`) address or a Host-only DHCP address, per [`02-network-architecture-planning.md`](./02-network-architecture-planning.md#configuring-the-virtual-network-editor-windows-host).
8. **Rename both adapters for clarity** before continuing — right-click each → **Rename**: call the NAT one `NAT-Internet` and the Host-only one `Internal-LabNet`. Every later step and every other Windows VM's build document (WINAPP01, CLIENT01) refers to adapters by these same two names, so keeping them consistent now avoids ambiguity later.
9. Right-click `Internal-LabNet` → **Properties** → select **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**.
10. Select **Use the following IP address**:
   - IP address: `192.168.10.10`
   - Subnet mask: `255.255.255.0`
   - Default gateway: *(leave blank — per the [dual-interface design](./02-network-architecture-planning.md#design-overview), internet traffic goes out NIC 1, not NIC 2)*
11. Leave DNS server fields blank for now — [Step 5](#step-5--configure-dns) sets this explicitly once DNS is installed.
12. Click **OK**, close the dialogs.

**PowerShell equivalent (optional):**
```powershell
Rename-Computer -NewName "DC01" -Restart
# after reboot:
Get-NetAdapter
Rename-NetAdapter -Name "Ethernet0" -NewName "NAT-Internet"
Rename-NetAdapter -Name "Ethernet1" -NewName "Internal-LabNet"
New-NetIPAddress -InterfaceAlias "Internal-LabNet" -IPAddress 192.168.10.10 -PrefixLength 24
```

---

## Step 4 — Install Active Directory Domain Services

**Install the role:**

1. In **Server Manager** → **Manage → Add Roles and Features**.
2. Click through: **Before You Begin → Next**, **Installation Type: Role-based or feature-based installation → Next**, **Select destination server: this server → Next**.
3. **Server Roles**: check **Active Directory Domain Services**. Accept the prompt to add required management tools/features.
4. **Next** through Features, AD DS intro page, **Confirmation → Install**.
5. Wait for installation to complete. Do **not** close the wizard yet — leave it, or close it and use the notification flag in the next step.

**Promote to a domain controller:**

6. Click the notification flag (yellow triangle) in Server Manager's top bar → **Promote this server to a domain controller**.
7. **Deployment Configuration**: select **Add a new forest**. Root domain name: `corp-lab.com.vn` → **Next**.
8. **Domain Controller Options**: keep default Forest/Domain functional level (Windows Server 2025). Ensure **DNS Server** is checked (it should be, by default). Set the **Directory Services Restore Mode (DSRM) password** — use a strong password and store it in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `DC01_10.10` group. This is separate from the domain Administrator password and is rarely needed but critical during disaster recovery → **Next**.
9. **DNS Options**: a warning about delegation may appear — this is expected for a new forest's first DC, ignore it → **Next**.
10. **Additional Options**: confirm the NetBIOS name shows `CORP-LAB` → **Next**.
11. **Paths**: accept the default database/log/SYSVOL paths → **Next**.
12. **Review Options**, then **Prerequisites Check** — resolve any warnings shown, then **Install**.
13. The server reboots automatically to complete promotion.

14. After reboot, log in as `CORP-LAB\Administrator` (same password as the local Administrator account before promotion — domain promotion converts it into the Domain Administrator account).

15. Confirm the domain is up by opening **Server Manager → Tools → Active Directory Users and Computers** and confirming the `corp-lab.com.vn` domain tree appears.

**PowerShell equivalent (optional):**
```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Import-Module ADDSDeployment
Install-ADDSForest `
    -DomainName "corp-lab.com.vn" `
    -DomainNetbiosName "CORP-LAB" `
    -InstallDns:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "Your-DSRM-Password-Here" -AsPlainText -Force) `
    -Force:$true
```

---

## Step 5 — Configure DNS

The DNS Server role was installed automatically with AD DS. Two things need explicit configuration: NIC 2's own DNS resolver setting, and a forwarder for external name resolution.

**Point `Internal-LabNet` at itself for DNS:**

1. **Control Panel → Network and Internet → Network Connections** → right-click `Internal-LabNet` → **Properties → Internet Protocol Version 4 (TCP/IPv4) → Properties**.
2. Set **Preferred DNS server** to `192.168.10.10` (DC01's own address). Leave the alternate blank. **OK**.

**Stop `NAT-Internet` from registering itself in DNS** — by default, Windows registers *every* active adapter's address under the domain zone via Dynamic DNS updates. Left unchecked, DC01's own NAT-assigned address ends up as a second, spurious `A` record for `dc01.corp-lab.com.vn` alongside the correct `192.168.10.10` one — any client that happens to receive the NAT address instead of the internal one will fail to reach DC01 at all:

3. **Network Connections** → right-click `NAT-Internet` → **Properties** → **Internet Protocol Version 4 (TCP/IPv4)** → **Properties** → **Advanced…** → **DNS** tab.
4. Uncheck **Register this connection's addresses in DNS**. **OK** through the dialogs.

**Add a forwarder for external resolution:**

5. **Server Manager → Tools → DNS**.
6. In the DNS Manager tree, right-click the server name (`DC01`) → **Properties**.
7. Go to the **Forwarders** tab → **Edit…**.
8. Add `8.8.8.8` and `1.1.1.1`, one per line → **OK** → **OK**.

This forwarder is what lets domain-joined machines using DC01 as their resolver still reach the internet — package repositories, and critically, the [external KMS host](../README.md#license-activation) used for Windows activation.

**Add a forward `A` record for each Linux VM:**

Windows domain-joined machines (DC01, WINAPP01, CLIENT01) register their own forward `A` record automatically via Dynamic DNS. **Linux VMs do not** — `realmd`/`sssd` domain-join has no equivalent auto-registration, forward or reverse. Without this step, `web01.corp-lab.com.vn` simply doesn't resolve at all, and any CNAME created later that points at it (see the per-service subdomain table further down) would be pointing at a name that doesn't exist.

| VM | IP | Forward record |
|---|---|---|
| WEB01 | `192.168.10.21` | `web01.corp-lab.com.vn` |
| MON01 | `192.168.10.40` | `mon01.corp-lab.com.vn` |
| OPS01 | `192.168.10.41` | `ops01.corp-lab.com.vn` |
| LOG01 | `192.168.10.50` | `log01.corp-lab.com.vn` |
| LOG02 | `192.168.10.51` | `log02.corp-lab.com.vn` |

9. For each row: **DNS Manager → Forward Lookup Zones → corp-lab.com.vn** → right-click → **New Host (A or AAAA)…** → **Name** (just the short hostname, e.g. `web01`), **IP address** → **Add Host**. Create only the `web01` entry now, since WEB01 is the only Linux VM built so far — add each remaining row when that VM's own document is reached.

**Clean up any stray NAT address already registered** (skip if DC01's DNS zone is freshly created and this hasn't happened yet):

10. In DNS Manager, expand **Forward Lookup Zones → corp-lab.com.vn**, find the host record(s) for `dc01`. If two `A` records exist — one showing `192.168.10.10` and another showing a `192.168.x.x` NAT-range address — delete the incorrect NAT one, keeping only `192.168.10.10`.

**Add a reverse lookup zone:**

Reverse DNS (IP → hostname) isn't required for anything in this lab to function, but it makes troubleshooting considerably easier — `nslookup -x 192.168.10.21` returning `web01.corp-lab.com.vn` instead of nothing saves real time when reading logs from Zabbix, Wazuh, or Suricata later, all of which log primarily by IP.

11. **DNS Manager** → right-click **Reverse Lookup Zones → New Zone…**.
12. **Primary zone**, check **Store the zone in Active Directory** → **Next**.
13. Replicate to **All DNS servers in this domain** (default is fine for a single-DC lab) → **Next**.
14. **IPv4 Reverse Lookup Zone**, **Network ID**: `192.168.10.0/24` → **Next**.
15. **Dynamic updates**: select **Allow only secure dynamic updates** — this lets domain-joined Windows machines (DC01, WINAPP01, CLIENT01) register their own PTR record automatically, the same way they already register forward `A` records → **Next** → **Finish**.

Windows domain-joined machines register their PTR record automatically the next time they refresh DNS registration (or immediately via `ipconfig /registerdns` on that machine) — this covers DC01, WINAPP01, and CLIENT01 with no manual step needed. **Linux VMs do not** — same gap as the forward `A` record above, `realmd`/`sssd` domain-join doesn't register any DNS records, forward or reverse. Add a PTR record manually for every Linux VM in this lab:

| VM | IP | PTR target |
|---|---|---|
| WEB01 | `192.168.10.21` | `web01.corp-lab.com.vn.` |
| MON01 | `192.168.10.40` | `mon01.corp-lab.com.vn.` |
| OPS01 | `192.168.10.41` | `ops01.corp-lab.com.vn.` |
| LOG01 | `192.168.10.50` | `log01.corp-lab.com.vn.` |
| LOG02 | `192.168.10.51` | `log02.corp-lab.com.vn.` |

16. For each row above: right-click the new `10.168.192.in-addr.arpa` zone → **New Pointer (PTR)…** → enter that row's **Host IP number** and **Host name** (note the trailing dot) → **OK**. Do this now for WEB01 (already built); repeat for MON01, OPS01, LOG01, and LOG02 as each is built in later documents — this doesn't need to happen all at once.

**Force DC01 to register its own PTR record now:**

The reverse zone was just created — DC01 itself won't automatically appear in it until the next time it re-registers with DNS (normally triggered by an IP change or reboot, neither of which is happening right now). Force it immediately:

```powershell
ipconfig /registerdns
```

Wait about 10-15 seconds, then confirm the record appeared:
```
nslookup 192.168.10.10
```
Expect `dc01.corp-lab.com.vn` in the response. Run the same `ipconfig /registerdns` on WINAPP01 and CLIENT01 once each is domain-joined, for the same reason.

**Add a friendly subdomain (CNAME) for each tool/service:**

Several VMs in this lab host more than one distinct web UI (MON01 alone will run both Zabbix and Wazuh). Rather than making everyone remember which VM hostname (and which port) each tool lives on, give each tool its own subdomain that points at the underlying VM — the same pattern real infrastructure uses so a service can move to a different host later without every bookmark/link breaking.

| Subdomain | Points to (CNAME target) | Service | Built in |
|---|---|---|---|
| `phpmyadmin.corp-lab.com.vn` | `web01.corp-lab.com.vn` | phpMyAdmin | [`07`](./07-web01-lamp-nginx-loadbalancer.md) |
| `wsus.corp-lab.com.vn` | `winapp01.corp-lab.com.vn` | WSUS console | [`09`](./09-winapp01-iis-sql-wsus.md) |
| `zabbix.corp-lab.com.vn` | `mon01.corp-lab.com.vn` | Zabbix Frontend — **HTTPS-only, on port `8443`** (not the default 443, since Wazuh Dashboard occupies 443 on this same VM; plain HTTP is disabled entirely, not just supplemented — see [`12`'s Step 19](./12-mon01-zabbix-server-configuration.md#step-19--enable-https-only-access)) | [`12`](./12-mon01-zabbix-server-configuration.md) |
| `wazuh.corp-lab.com.vn` | `mon01.corp-lab.com.vn` | Wazuh Dashboard | [`13`](./13-mon01-wazuh-manager-configuration.md) |
| `grafana.corp-lab.com.vn` | `mon01.corp-lab.com.vn` | Grafana — **HTTPS-only, on port `9443`** | [`14`](./14-mon01-prometheus-grafana-monitoring.md) |
| `kibana.corp-lab.com.vn` | `log01.corp-lab.com.vn` | Kibana | [`15`](./15-log01-elasticsearch-logstash-kibana.md) |
| `opensearch.corp-lab.com.vn` | `log02.corp-lab.com.vn` | OpenSearch Dashboards | [`16`](./16-log02-opensearch-deployment.md) |
| `keycloak.corp-lab.com.vn` | `ops01.corp-lab.com.vn` | Keycloak admin console | [`19`](./19-ops01-ansible-keycloak.md) |

17. For each row: **DNS Manager → Forward Lookup Zones → corp-lab.com.vn** → right-click → **New Alias (CNAME)…** → **Alias name** (e.g. `phpmyadmin`), **Fully qualified domain name (FQDN) for target host** (e.g. `web01.corp-lab.com.vn.`) → **OK**. Create only `phpmyadmin.corp-lab.com.vn` now, since WEB01 is the only VM built so far — add each remaining row when that VM's own document is reached, exactly like the PTR records above.

**Verify:**

18. Confirm the forward `A` record resolves:
```
nslookup web01.corp-lab.com.vn
```
Expect `192.168.10.21` in the response.

19. In DNS Manager, right-click the server → **Launch nslookup** (or open Command Prompt) and query an external name:
```
nslookup active.orientsoftware.asia
nslookup rockylinux.org
```
Both should resolve successfully. If either fails, confirm NIC 1 (NAT) still has working internet connectivity independent of NIC 2 — DNS forwarding relies on DC01 itself being able to reach `8.8.8.8`/`1.1.1.1` over NIC 1.

20. Confirm reverse lookup works for WEB01:
```
nslookup 192.168.10.21
```
Expect `web01.corp-lab.com.vn` in the response.

21. Confirm the CNAME resolves:
```
nslookup phpmyadmin.corp-lab.com.vn
```
Expect it to resolve through to `192.168.10.21` (the same address as `web01.corp-lab.com.vn`).

**PowerShell equivalent (optional):**
```powershell
Set-DnsClientServerAddress -InterfaceAlias "Internal-LabNet" -ServerAddresses 192.168.10.10
Set-DnsClient -InterfaceAlias "NAT-Internet" -RegisterThisConnectionsAddress $false
Add-DnsServerForwarder -IPAddress 8.8.8.8, 1.1.1.1
Resolve-DnsName active.orientsoftware.asia

# Add each Linux VM's forward A record as it's built — not all at once
Add-DnsServerResourceRecordA -ZoneName "corp-lab.com.vn" -Name "web01" -IPv4Address "192.168.10.21"
Add-DnsServerResourceRecordA -ZoneName "corp-lab.com.vn" -Name "mon01" -IPv4Address "192.168.10.40"
Add-DnsServerResourceRecordA -ZoneName "corp-lab.com.vn" -Name "ops01" -IPv4Address "192.168.10.41"
Add-DnsServerResourceRecordA -ZoneName "corp-lab.com.vn" -Name "log01" -IPv4Address "192.168.10.50"
Add-DnsServerResourceRecordA -ZoneName "corp-lab.com.vn" -Name "log02" -IPv4Address "192.168.10.51"

Add-DnsServerPrimaryZone -NetworkId "192.168.10.0/24" -ReplicationScope "Domain" -DynamicUpdate "Secure"

# Add each Linux VM's PTR record as it's built — not all at once
Add-DnsServerResourceRecordPtr -ZoneName "10.168.192.in-addr.arpa" -Name "21" -PtrDomainName "web01.corp-lab.com.vn."
Add-DnsServerResourceRecordPtr -ZoneName "10.168.192.in-addr.arpa" -Name "40" -PtrDomainName "mon01.corp-lab.com.vn."
Add-DnsServerResourceRecordPtr -ZoneName "10.168.192.in-addr.arpa" -Name "41" -PtrDomainName "ops01.corp-lab.com.vn."
Add-DnsServerResourceRecordPtr -ZoneName "10.168.192.in-addr.arpa" -Name "50" -PtrDomainName "log01.corp-lab.com.vn."
Add-DnsServerResourceRecordPtr -ZoneName "10.168.192.in-addr.arpa" -Name "51" -PtrDomainName "log02.corp-lab.com.vn."

# Add a CNAME per tool/service — create each as its VM/service is built
Add-DnsServerResourceRecordCName -ZoneName "corp-lab.com.vn" -Name "phpmyadmin" -HostNameAlias "web01.corp-lab.com.vn."
```

---

## Step 6 — Install and configure DHCP

**Install the role:**

1. **Server Manager → Manage → Add Roles and Features → Server Roles** → check **DHCP Server** → accept prompts → **Install**.

**Authorize the DHCP server in AD:**

2. Click the notification flag → **Complete DHCP configuration** → **Commit** → **Close**. This authorizes the server in Active Directory, required before it will hand out leases on a domain-joined network.

**Create the scope:**

3. **Server Manager → Tools → DHCP**.
4. Expand `dc01.corp-lab.com.vn` → right-click **IPv4** → **New Scope…**.
5. Follow the wizard:
   - Name: `Clients`
   - Start IP: `192.168.10.100`, End IP: `192.168.10.200`, Subnet mask: `255.255.255.0` (matches the reserved DHCP range from [`02-network-architecture-planning.md`](./02-network-architecture-planning.md#ip-allocation-table))
   - Exclusions/delay: none needed
   - Lease duration: default is fine
   - **Configure DHCP Options: Yes**
   - Router (default gateway): **leave blank** — deliberately, since per the [dual-interface design](./02-network-architecture-planning.md#design-overview), every VM reaches the internet independently through its own NIC 1 (NAT), not through NIC 2 (Host-only)
   - Domain name and DNS servers: Parent domain `corp-lab.com.vn`, DNS server `192.168.10.10`
   - WINS: skip
   - Activate this scope now: **Yes**
6. Finish the wizard.

**Configure DNS dynamic update integration:**

By default, DHCP only updates DNS on a client's behalf if that client explicitly requests it — most modern Windows clients do, but this is worth setting explicitly rather than assuming, since it's what makes CLIENT01 (and any future DHCP-assigned client) reliably get both its forward `A` record and reverse `PTR` record registered without manual intervention, the same automatic registration DC01/WINAPP01 get from their static IP configuration.

7. In the DHCP console, right-click **IPv4** → **Properties** → **DNS** tab.
8. Confirm **Enable DNS dynamic updates according to the settings below** is checked, with **Dynamically update DNS records only if requested by the DHCP clients** selected (the default — appropriate here since this lab's Windows/Linux clients already handle their own registration correctly).
9. Check **Dynamically update DNS records for DHCP clients that do not request updates** — this is the setting that actually matters: it's a safety net covering any older or non-standard DHCP client that doesn't ask for dynamic DNS itself, ensuring it still gets registered rather than silently going unregistered.
10. **OK**.

**PowerShell equivalent (optional):**
```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName "DC01.corp-lab.com.vn" -IPAddress 192.168.10.10
Add-DhcpServerv4Scope -Name "Clients" -StartRange 192.168.10.100 -EndRange 192.168.10.200 -SubnetMask 255.255.255.0 -State Active
Set-DhcpServerv4OptionValue -ScopeId 192.168.10.0 -DnsServer 192.168.10.10 -DnsDomain "corp-lab.com.vn"
Set-DhcpServerv4DnsSetting -DynamicUpdates "OnClientRequest" -UpdateDnsRRForOlderClients $true
```

---

## Step 7 — Configure NTP

DC01 is the only domain controller in this lab, so it also holds the PDC Emulator FSMO role by default — configure it to sync from an external, authoritative time source rather than the (inaccurate) hardware clock default. This specific setting isn't exposed anywhere in Control Panel's Date & Time dialog for a domain controller, so the command line is the only practical way to configure it — there's no GUI equivalent worth using here.

Open an elevated Command Prompt or PowerShell window:

```
w32tm /config /manualpeerlist:"time.windows.com,0x9 pool.ntp.org,0x9" /syncfromflags:manual /reliable:yes /update
net stop w32time
net start w32time
```

Verify:

```
w32tm /query /status
```
Look for `Source:` showing one of the configured peers rather than `Local CMOS Clock`.

Every other domain-joined machine in this lab (WINAPP01, CLIENT01, and any domain-joined Linux VM) will automatically sync its time from DC01 via the domain hierarchy once joined — no separate NTP configuration is needed on those machines.

---

## Step 8 — Storage Spaces demo

Using the second disk added in [Step 2](#step-2--add-the-storage-spaces-disk):

**Create the Storage Pool:**

1. **Server Manager → File and Storage Services → Storage Pools**.
2. Under **Storage Pools**, click **Tasks → New Storage Pool…**.
3. Follow the wizard: Name `DataPool`, select the 40 GB physical disk from the list → **Create**.

**Create the Virtual Disk:**

4. Right-click the new `DataPool` → **New Virtual Disk…**.
5. Follow the wizard: Name `DataVirtualDisk`, Storage layout **Simple**, Provisioning **Thin** or **Fixed** (Fixed is simpler for a lab), Size **35 GB**.

**Create the Volume:**

6. The wizard offers to continue directly into the **New Volume Wizard** — accept.
7. Select the new virtual disk, assign drive letter **D:**, file system **NTFS**, volume label `DataStorage` → **Create**.

**Verify:**

8. Open **File Explorer** → confirm `D:` (`DataStorage`) appears with roughly 35 GB capacity.

**PowerShell equivalent (optional):**
```powershell
Get-Disk
Initialize-Disk -Number 1 -PartitionStyle GPT
$disk = Get-PhysicalDisk | Where-Object { $_.Size -eq 40GB }
New-StoragePool -FriendlyName "DataPool" -StorageSubsystemFriendlyName "Windows Storage*" -PhysicalDisks $disk
New-VirtualDisk -StoragePoolFriendlyName "DataPool" -FriendlyName "DataVirtualDisk" -Size 35GB -ResiliencySettingName Simple
Get-VirtualDisk -FriendlyName "DataVirtualDisk" | Get-Disk | Initialize-Disk -PartitionStyle GPT -PassThru | New-Partition -DriveLetter D -UseMaximumSize | Format-Volume -FileSystem NTFS -NewFileSystemLabel "DataStorage" -Confirm:$false
```

---

> **Note:** an SMB file share demo was deliberately **not** added here on DC01. Hosting general-purpose file shares on a domain controller is avoided in this lab for the same reason IIS and SQL Server live on WINAPP01 instead of DC01 — a domain controller should carry as few extra roles as possible to minimize its attack surface, and DC01 already hosts two highly sensitive shares by necessity (`SYSVOL`, `NETLOGON`, used for Group Policy distribution). The SMB share demo instead appears in [`09-winapp01-iis-sql-wsus.md`](./09-winapp01-iis-sql-wsus.md), on a server whose role already includes general application/data services. The `D:` volume above still serves as a useful example of Storage Spaces management on its own — it doesn't need an SMB share to be a complete demonstration.

---

## Step 9 — Hardening baseline

1. Run `mmc.exe` → **File → Add/Remove Snap-in → Security Configuration and Analysis → Add → OK**.
2. Right-click **Security Configuration and Analysis → Open Database**, create a new database (e.g. `DC01-baseline.sdb`).
3. Right-click the snap-in again → **Import Template…** → select [`configs/dc01-baseline-security-template.inf`](../configs/dc01-baseline-security-template.inf) from this repository. This template covers password policy, account lockout policy, and a conservative set of security options appropriate for a lab domain controller (see the comments at the top of the file for exactly what it does and doesn't touch).
4. Right-click the snap-in → **Analyze Computer Now** to compare current settings against the imported template.
5. Review discrepancies (a red X marks a setting that differs from the template; a green check marks a match), then **Configure Computer Now** to apply the template once you've reviewed what it changes.
6. Review the applied log if needed: **%windir%\security\logs\scesrv.log**.

> **Important caveat for a domain controller:** Password Policy and Account Lockout Policy settings from this local template can be **overridden by domain-level Group Policy** — specifically the **Default Domain Policy**, which Windows applies on top of local security settings for any domain-joined machine, including the DC itself. If `secedit`/this snap-in shows the template applied successfully but `net accounts` later reports different effective values, the Default Domain Policy is taking precedence, which is expected AD behavior, not a fault in this template. To make password/lockout policy authoritative domain-wide, those settings ultimately belong in **Default Domain Policy** (Group Policy Management → `corp-lab.com.vn` → Default Domain Policy) rather than only in this local template — this local baseline still matters for the other security options (registry values, privilege rights) that aren't controlled by Default Domain Policy.

> Looking for something more comprehensive than this lab's minimal template? Microsoft publishes an official, far more thorough set of baselines for free — the [Security Compliance Toolkit](https://www.microsoft.com/en-us/download/details.aspx?id=55319) — which can be imported the same way in place of the file above.

**Baseline audit policy** — instead of `auditpol` at the command line, this is configured through Group Policy on a domain controller:

5. **Server Manager → Tools → Group Policy Management**.
6. Expand `corp-lab.com.vn` → **Domain Controllers → Default Domain Controllers Policy** → right-click → **Edit**.
7. Navigate to **Computer Configuration → Policies → Windows Settings → Security Settings → Advanced Audit Policy Configuration → Audit Policies**.
8. Under **Account Logon**, **Logon/Logoff**, and **Object Access**, double-click each subcategory and enable both **Success** and **Failure** auditing.
9. Close the Group Policy Management Editor — changes apply automatically on next policy refresh (or run `gpupdate /force` from Command Prompt to apply immediately).

---

## Step 10 — Remote access configuration

**Confirm RDP is enabled** (enabled by default on Windows Server, verify rather than assume):

1. **Server Manager → Local Server** → check the **Remote Desktop** value — should read **Enabled**. If not, click it and enable **Allow remote connections to this computer**.

2. Enabling Remote Desktop this way automatically enables the built-in **Remote Desktop** rule group in Windows Defender Firewall — no separate firewall rule is needed. Confirm it's active:
   - **Control Panel → System and Security → Windows Defender Firewall → Advanced Settings** (or run `wf.msc`) → **Inbound Rules** → look for **Remote Desktop - User Mode (TCP-In)** showing **Enabled: Yes**.

This lab keeps RDP on its default port (**3389**) rather than moving it to a custom port — no further action is needed here, and the [mRemoteNG connection](./03-remote-access-tooling-setup.md#pre-building-the-mremoteng-connection-list) for `DC01_10.10` can stay on port 3389 as originally configured.

**(Optional) Disable IPv6:**

3. **Network Connections** → right-click each adapter → **Properties** → uncheck **Internet Protocol Version 6 (TCP/IPv6)** → **OK**.

**PowerShell equivalent (optional):**
```powershell
Get-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections"
# Expect 0 (RDP allowed)
Get-NetFirewallRule -DisplayGroup "Remote Desktop" | Select-Object DisplayName, Enabled
```

---

## Step 11 — OU structure and baseline GPOs

**Create the OU structure:**

1. **Server Manager → Tools → Active Directory Users and Computers**.
2. Right-click `corp-lab.com.vn` → **New → Organizational Unit**.
3. Create four OUs one at a time: `Servers`, `Workstations`, `ServiceAccounts`, `Users` (uncheck "Protect container from accidental deletion" only if you're comfortable with that — leaving it checked is the safer default).

**Create a baseline GPO:**

4. **Server Manager → Tools → Group Policy Management**.
5. Expand `corp-lab.com.vn` → right-click **Group Policy Objects → New**.
6. Name: `Baseline-Policy` → **OK**. Leave it unlinked and unpopulated for now — it's configured further in [`10-gpo-wsus-client-policy.md`](./10-gpo-wsus-client-policy.md) once WSUS exists.

**Link it to the Servers and Workstations OUs:**

7. Right-click **Servers** OU → **Link an Existing GPO…** → select `Baseline-Policy` → **OK**.
8. Repeat for the **Workstations** OU.

> **Easy mistake to make:** in the GPMC tree, it's simple to accidentally right-click the domain root (`corp-lab.com.vn`) instead of the `Servers`/`Workstations` OU underneath it — both context menus look identical, and a GPO linked at the domain root silently applies to *every* OU beneath it, including `Domain Controllers`. Verify the link landed on the correct object before moving on:
> ```powershell
> (Get-GPInheritance -Target "OU=Servers,DC=corp-lab,DC=com,DC=vn").GpoLinks.DisplayName
> (Get-GPInheritance -Target "DC=corp-lab,DC=com,DC=vn").GpoLinks.DisplayName
> ```
> The first command should list `Baseline-Policy`; the second (domain root) should **not** — if it does, the link landed in the wrong place and needs `Remove-GPLink -Name "Baseline-Policy" -Target "DC=corp-lab,DC=com,DC=vn"` followed by re-linking to the correct OU.

**PowerShell equivalent (optional):**
```powershell
New-ADOrganizationalUnit -Name "Servers" -Path "DC=corp-lab,DC=com,DC=vn"
New-ADOrganizationalUnit -Name "Workstations" -Path "DC=corp-lab,DC=com,DC=vn"
New-ADOrganizationalUnit -Name "ServiceAccounts" -Path "DC=corp-lab,DC=com,DC=vn"
New-ADOrganizationalUnit -Name "Users" -Path "DC=corp-lab,DC=com,DC=vn"
New-GPO -Name "Baseline-Policy"
New-GPLink -Name "Baseline-Policy" -Target "OU=Servers,DC=corp-lab,DC=com,DC=vn"
New-GPLink -Name "Baseline-Policy" -Target "OU=Workstations,DC=corp-lab,DC=com,DC=vn"
```

---

## Step 12 — Final verification checklist

Before treating DC01 as complete and moving on to build other VMs, run this consolidated set of checks — catching a misconfiguration now is far cheaper than discovering it after WINAPP01 or CLIENT01 depend on it.

1. **Active Directory health** — open an elevated Command Prompt:
```
dcdiag
```
Scan the output for any line reading `... failed test ...`. A clean run shows every test passing.

2. **DNS resolving both internally and externally** — open Command Prompt:
```
nslookup dc01.corp-lab.com.vn
nslookup active.orientsoftware.asia
```
The first confirms internal DNS is authoritative for the domain; the second confirms the forwarder from [Step 5](#step-5--configure-dns) still works.

3. **DHCP is authorized and scope is active** — **Server Manager → Tools → DHCP** → expand `dc01.corp-lab.com.vn` → **IPv4** → confirm a green up-arrow icon (authorized) and that the `Clients` scope shows **Active**.

4. **NTP is synced to an external source** — Command Prompt:
```
w32tm /query /status
```
Confirm `Source:` is not `Local CMOS Clock`.

5. **Storage Spaces volume is healthy** — **File Explorer** → confirm `D:` (`DataStorage`) is present and accessible.

6. **Remote access works end-to-end** — from the host machine, connect via the `DC01_10.10` entry in [mRemoteNG](./03-remote-access-tooling-setup.md#pre-building-the-mremoteng-connection-list) and confirm a successful RDP session on port 3389.

7. **Credentials are stored, not memorized** — confirm the local Administrator password, the domain Administrator password, and the DSRM password are all saved in [KeePass](./03-remote-access-tooling-setup.md#keepass) under the `DC01_10.10` group before moving on.

If all seven checks pass, DC01 is ready to serve as the foundation for the rest of this lab.

---

## Next step

Continue to [`07-web01-lamp-nginx-loadbalancer.md`](./07-web01-lamp-nginx-loadbalancer.md) to build WEB01, or [`09-winapp01-iis-sql-wsus.md`](./09-winapp01-iis-sql-wsus.md) if you're ready to build the application server next — both depend only on DC01 being in place, not on each other.
