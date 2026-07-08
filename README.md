# Enterprise Infra Lab

A free, step-by-step guide for building a self-hosted enterprise infrastructure simulation on **VMware Workstation** — combining Windows Server (Active Directory, IIS, SQL Server, WSUS) and Rocky Linux (LAMP stack, reverse proxy/load balancing, monitoring, and SIEM/log analytics) into a single cohesive lab environment.

This repository exists to help anyone — students, career-changers, or working sysadmins wanting hands-on practice — build the same lab from scratch, using only free/evaluation software and publicly documented tools. Every document under [`docs/`](./docs) corresponds to one build phase, listed in the exact order it should be executed, so you can follow along end to end or jump straight to the section you need.

No prior lab experience is required — just a reasonably capable host machine and the patience to work through each phase in order.

---

## Table of contents

- [Project overview](#project-overview)
- [Hardware requirements](#hardware-requirements)
- [Software requirements](#software-requirements)
- [Guest operating systems](#guest-operating-systems)
- [License activation](#license-activation)
- [Network architecture summary](#network-architecture-summary)
- [Virtual machine inventory](#virtual-machine-inventory)
- [Build order](#build-order)
- [Documentation conventions](#documentation-conventions)
- [Disclaimer](#disclaimer)

---

## Project overview

This lab reproduces the core building blocks of a small-to-medium enterprise IT environment, entirely virtualized on a single host machine:

- **Identity & directory services** — Active Directory Domain Services, DNS, DHCP
- **Patch management** — WSUS integrated with Group Policy
- **Web infrastructure** — LAMP stack behind an Nginx reverse proxy with load balancing and SSL/TLS termination
- **Database & application services** — Microsoft SQL Server, IIS
- **Network intrusion detection** — Suricata (NIDS)
- **Monitoring & SIEM** — Zabbix, Wazuh, Elastic Stack (ELK), OpenSearch
- **Identity federation & SSO** — Keycloak, running on a dedicated operations VM (OPS01), providing single sign-on across every monitoring/log dashboard (Kibana, OpenSearch Dashboards, Wazuh Dashboard, Zabbix Frontend)
- **Configuration management** — Ansible, run from a control node on OPS01, for pushing configuration and automating repeatable tasks across every other VM
- **Metrics monitoring (separate ecosystem from Zabbix)** — Prometheus and Grafana, added alongside Zabbix rather than replacing it, for hands-on practice with the pull-based/exporter model common in cloud-native and DevOps-oriented environments
- **Automation & hardening** — scripted backups, firewall/SELinux hardening, scheduled maintenance

The goal is hands-on, reproducible practice with the full lifecycle of enterprise systems administration: provisioning, configuration, security hardening, monitoring, and automation — all documented as a public runbook.

---

## Hardware requirements

| Component | Minimum | Recommended |
|---|---|---|
| CPU | 8 cores / 16 threads, VT-x / AMD-V enabled in BIOS | 12–16 cores / 24–32 threads |
| RAM | 52 GB | 64–72 GB |
| Storage | 800 GB free (SSD) | 1 TB NVMe |
| Network | 1 host NIC (for NAT outbound internet access) | — |

> Total VM resource footprint is approximately **48–58 GB RAM** and **~590 GB disk** across 8 virtual machines. See [Virtual machine inventory](#virtual-machine-inventory) for the full breakdown.

---

## Software requirements

Install these on the **host machine** before building any VM.

| Software | Purpose | Download |
|---|---|---|
| VMware Workstation Pro | Hypervisor (latest version) | https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Workstation%20Pro&freeDownloads=true |
| PuTTY | SSH client for Linux VMs | https://www.putty.org/ |
| mRemoteNG | Centralized RDP/SSH connection manager | https://github.com/mRemoteNG/mRemoteNG |
| WinSCP | SFTP file transfer to Linux VMs | https://winscp.net/ |
| KeePass | Encrypted credential storage for all VM accounts | https://keepass.info/download.html |

Detailed installation steps: [`docs/00-vmware-workstation-installation.md`](./docs/00-vmware-workstation-installation.md), [`docs/03-remote-access-tooling-setup.md`](./docs/03-remote-access-tooling-setup.md)

---

## Guest operating systems

| VM(s) | Operating system | Edition | ISO source |
|---|---|---|---|
| DC01, WINAPP01 | Windows Server 2025 | **Datacenter** (Desktop Experience) | https://www.microsoft.com/evalcenter/evaluate-windows-server-2025 |
| CLIENT01 | Windows 11 24H2 | **Pro** | https://www.microsoft.com/software-download/windows11 |
| WEB01, MON01, OPS01, LOG01, LOG02 | Rocky Linux 10 | Minimal | https://rockylinux.org/download |

ISO acquisition and checksum verification steps: [`docs/01-iso-acquisition-and-verification.md`](./docs/01-iso-acquisition-and-verification.md)

Golden baseline (sysprep'd template) build steps:
- Windows: [`docs/04-golden-baseline-windows-server-2025.md`](./docs/04-golden-baseline-windows-server-2025.md)
- Rocky Linux: [`docs/05-golden-baseline-rocky-linux-10.md`](./docs/05-golden-baseline-rocky-linux-10.md)

---

## License activation

All Windows machines in this lab are activated against an **external, online KMS host** (not part of this lab's VM inventory).

> **If you're following this guide yourself:** the GVLK keys below are Microsoft's own [publicly documented generic volume license keys](https://learn.microsoft.com/en-us/windows-server/get-started/kms-client-activation-keys) — they only work against a KMS host that already holds a valid Windows Server/volume-licensing agreement, they do not grant a free license on their own. The KMS hostname is specific to this author's environment; replace it with your own organization's KMS host, or use Microsoft's Evaluation edition without activation if you don't have one available.

| Edition | GVLK | KMS host (example — replace with your own) |
|---|---|---|
| Windows Server 2025 Datacenter | `D764K-2NDRG-47T6Q-P8T8W-YP6DF` | `active.orientsoftware.asia:1688` |
| Windows 11 Pro | `W269N-WFGWX-YVC9B-4J6C9-T83GX` | `active.orientsoftware.asia:1688` |

Windows Server images are provisioned from the free Evaluation ISO and converted to a retail edition via `DISM /Set-Edition` **before** activation — the full sequence (edition check, edition conversion, connectivity test, activation, verification) is documented in [`docs/04-golden-baseline-windows-server-2025.md`](./docs/04-golden-baseline-windows-server-2025.md).

> The KMS host is external to this lab's VM inventory and is reached over the **NAT adapter (NIC 1)**, not the internal `192.168.10.0/24` network. All Windows VMs must be able to **resolve** the KMS hostname and reach it on **TCP 1688** through NAT. Once DC01's DNS is in use as the client resolver, ensure a DNS forwarder to a public resolver is configured so KMS activation continues to work after domain join.

---

## Network architecture summary

Every VM in this lab uses **two virtual network adapters**:

| Adapter | VMware network | Subnet | Purpose |
|---|---|---|---|
| NIC 1 | NAT (VMnet8) | VMware's default NAT subnet (unchanged) | Outbound internet access — package updates, source downloads, KMS activation |
| NIC 2 | Host-only (VMnet1) | `192.168.10.0/24` | Internal lab traffic — inter-VM communication, domain join, DNS, monitoring, all static IPs |

All static IP addresses and hostnames referenced throughout this guide belong to **NIC 2** (Host-only). NIC 1 (NAT) only needs a working DHCP-assigned address and is otherwise left alone — it exists purely so each VM can reach the internet independently of the internal lab network.

Full instructions for configuring both adapters, repointing VMnet1 to `192.168.10.0/24`, and wiring each VM with the correct dual-NIC setup: [`docs/02-network-architecture-planning.md`](./docs/02-network-architecture-planning.md)

> Feel free to substitute your own internal subnet — `192.168.10.0/24` is simply what this guide uses throughout for consistency between documents. If you change it, update every static IP reference accordingly.

> Because KMS activation and package downloads happen over NIC 1 (NAT), the KMS hostname and package repositories only need to be reachable through that adapter — not through the internal 192.168.10.0/24 network. If you later add a DNS forwarder on DC01 for domain-joined clients, make sure NAT (NIC 1) remains functional so outbound lookups still resolve.

### IP allocation

Two separate naming conventions are used throughout this lab:

- **VMware Workstation VM name** (the Library display name and `.vmx` folder name) — always `HOST_lastTwoOctets`, using only the **last two octets** of the IP address, e.g. `DC01_10.10` for `192.168.10.10`. This keeps names short while still identifiable at a glance in the Workstation Library without opening the VM.
- **OS hostname** (set via `hostnamectl` on Linux or `Rename-Computer` on Windows) — just `HOST`, e.g. `DC01`. This is the name used for DNS records, domain join, and everywhere inside the OS itself.

| Host | VMware VM name | OS hostname | IP address | Role |
|---|---|---|---|---|
| DC01 | `DC01_10.10` | `DC01` | `192.168.10.10` | Active Directory, DNS, DHCP, NTP |
| WINAPP01 | `WINAPP01_10.15` | `WINAPP01` | `192.168.10.15` | IIS, SQL Server, WSUS |
| WEB01 | `WEB01_10.21` | `WEB01` | `192.168.10.21` | Nginx (reverse proxy / load balancer), Apache × 2, MariaDB, Suricata |
| MON01 | `MON01_10.40` | `MON01` | `192.168.10.40` | Zabbix, Wazuh, Prometheus, Grafana |
| OPS01 | `OPS01_10.41` | `OPS01` | `192.168.10.41` | Ansible control node, Keycloak (SSO) |
| LOG01 | `LOG01_10.50` | `LOG01` | `192.168.10.50` | Elasticsearch, Logstash, Kibana |
| LOG02 | `LOG02_10.51` | `LOG02` | `192.168.10.51` | OpenSearch, OpenSearch Dashboards |
| CLIENT01 | `CLIENT01_dhcp` | `CLIENT01` | DHCP (`192.168.10.100`–`200` pool) | Domain-joined test client |

---

## Virtual machine inventory

| VM | vCPU | RAM | Disk 1 (OS + swap) | Disk 2 (data) | Total disk | Role |
|---|---|---|---|---|---|---|
| DC01 | 2 | 4–6 GB | 60 GB | 40 GB *(Storage Spaces demo)* | 100 GB | AD DS, DNS, DHCP, NTP |
| WINAPP01 | 4 | 8–10 GB | 60 GB | 40 GB *(SQL Server data)* | 100 GB | IIS, SQL Server, WSUS |
| CLIENT01 | 2 | 2–4 GB | 40 GB | — | 40 GB | Domain-joined test client |
| WEB01 | 4–6 | 8 GB | 50 GB *(30 OS + 20 swap)* | 20 GB *(LVM demo)* | 70 GB | Nginx LB, Apache × 2, MariaDB, Suricata |
| MON01 | 4 | 10–12 GB | 60 GB *(40 OS + 20 swap)* | 20 GB *(Zabbix/Wazuh/Prometheus data)* | 80 GB | Zabbix, Wazuh, Prometheus, Grafana |
| OPS01 | 2 | 4 GB | 40 GB *(20 OS + 20 swap)* | — | 40 GB | Ansible control node, Keycloak (SSO) |
| LOG01 | 4 | 8 GB | 60 GB *(40 OS + 20 swap)* | 20 GB *(Elasticsearch index data)* | 80 GB | Elasticsearch, Logstash, Kibana |
| LOG02 | 4 | 8 GB | 60 GB *(40 OS + 20 swap)* | 20 GB *(OpenSearch data)* | 80 GB | OpenSearch, OpenSearch Dashboards |
| **Total** | **26–30** | **48–58 GB** | | | **~590 GB** | |

**Notes:**
- Swap size follows a fixed rule of **2× RAM**, rounded up to the nearest 10 GB.
- Any VM with a second virtual disk uses it for a specific hands-on exercise (Storage Spaces, LVM, or dedicated data storage) rather than general capacity — this is called out explicitly in the relevant build document.
- For JVM-heavy services (MON01, LOG01, LOG02), `vm.swappiness` is tuned low despite the swap partition being present, to avoid JVM garbage-collection stalls caused by aggressive swapping. OPS01 runs no JVM services, so this doesn't apply there.

---

## Build order

Documents are numbered in the order they must be executed. Later phases assume earlier ones are complete (e.g. DC01 must exist before any domain-join step).

| # | Document | Phase |
|---|---|---|
| 00 | [`vmware-workstation-installation.md`](./docs/00-vmware-workstation-installation.md) | Host setup |
| 01 | [`iso-acquisition-and-verification.md`](./docs/01-iso-acquisition-and-verification.md) | Host setup |
| 02 | [`network-architecture-planning.md`](./docs/02-network-architecture-planning.md) | Host setup |
| 03 | [`remote-access-tooling-setup.md`](./docs/03-remote-access-tooling-setup.md) | Host setup |
| 04 | [`golden-baseline-windows-server-2025.md`](./docs/04-golden-baseline-windows-server-2025.md) | Golden baseline |
| 05 | [`golden-baseline-rocky-linux-10.md`](./docs/05-golden-baseline-rocky-linux-10.md) | Golden baseline |
| 06 | [`dc01-active-directory-dns-dhcp.md`](./docs/06-dc01-active-directory-dns-dhcp.md) | Domain controller |
| 07 | [`web01-lamp-nginx-loadbalancer.md`](./docs/07-web01-lamp-nginx-loadbalancer.md) | Web infrastructure |
| 08 | [`web01-suricata-nids.md`](./docs/08-web01-suricata-nids.md) | Web infrastructure |
| 09 | [`winapp01-iis-sql-wsus.md`](./docs/09-winapp01-iis-sql-wsus.md) | Application server |
| 10 | [`gpo-wsus-client-policy.md`](./docs/10-gpo-wsus-client-policy.md) | Application server |
| 11 | [`client01-domain-join-wsus-verification.md`](./docs/11-client01-domain-join-wsus-verification.md) | Client verification |
| 12 | [`mon01-zabbix-server-configuration.md`](./docs/12-mon01-zabbix-server-configuration.md) | Monitoring |
| 13 | [`mon01-wazuh-manager-configuration.md`](./docs/13-mon01-wazuh-manager-configuration.md) | Monitoring |
| 14 | [`mon01-prometheus-grafana-monitoring.md`](./docs/14-mon01-prometheus-grafana-monitoring.md) | Monitoring (separate ecosystem from Zabbix) |
| 15 | [`log01-elasticsearch-logstash-kibana.md`](./docs/15-log01-elasticsearch-logstash-kibana.md) | Log analytics |
| 16 | [`log02-opensearch-deployment.md`](./docs/16-log02-opensearch-deployment.md) | Log analytics |
| 17 | [`agent-deployment-all-vms.md`](./docs/17-agent-deployment-all-vms.md) | Monitoring rollout |
| 18 | [`secops-firewall-selinux-hardening.md`](./docs/18-secops-firewall-selinux-hardening.md) | Security hardening |
| 19 | [`ops01-ansible-keycloak.md`](./docs/19-ops01-ansible-keycloak.md) | Operations tooling |
| 20 | [`automation-backup-scheduling.md`](./docs/20-automation-backup-scheduling.md) | Automation |

### Dependency summary

```
Host setup (00–03)
        │
        ▼
Golden baselines (04–05)
        │
        ├──► DC01 (06) ──► WINAPP01 + WSUS (09–10) ──► CLIENT01 (11)
        │
        ├──► WEB01 + Suricata (07–08)
        │
        └──► MON01 — Zabbix (12) ──► Wazuh (13) ──► Prometheus & Grafana (14)
                │
                ▼
        LOG01 (15)
                │
                ▼
        LOG02 (16)
                │
                ▼
        Agent deployment (17)
        (requires WEB01, MON01, OPS01, LOG01, LOG02
         to already exist)
                │
                ▼
        SecOps hardening (18)
                │
                ▼
        OPS01 — Ansible + Keycloak (19)
        (new VM: clone golden baseline, domain-join,
         then Ansible control node — needs SSH access
         to every other VM already in place — followed
         by Keycloak, which requires Kibana, OpenSearch
         Dashboards, Wazuh Dashboard, and Zabbix Frontend
         to already exist for SSO integration)
                │
                ▼
        Automation & backup (20)
        (can use Ansible from Step 19 to push
         scripts/scheduling across every VM)
```

---

## Documentation conventions

- All documents are written in **English**, formatted as Markdown.
- All shell/PowerShell commands are in **English**, using fenced code blocks.
- Every piece of software built or installed **from source** includes a direct download link at the point it is first referenced — no undocumented dependencies.
- File naming follows `NN-topic-name.md`, where `NN` reflects build order, not importance.
- **Supporting files**: any file referenced by a build document that isn't just an inline command — security templates, config files, scripts — is committed to this repository under [`configs/`](./configs) rather than left as a "bring your own" placeholder. Each build document links directly to the exact file it uses.
- **VM naming**: the VMware Workstation VM name (Library display name / `.vmx` folder) always follows `HOST_lastTwoOctets`, using only the last two octets of the IP address (e.g. `DC01_10.10` for `192.168.10.10`) so every VM is identifiable at a glance without opening it. The OS-level hostname (set via `hostnamectl` on Linux or `Rename-Computer` on Windows) uses just `HOST` (e.g. `DC01`) — this is the name used for DNS, domain join, and inside the OS itself. Never mix the two.
- Any VM provisioned with more than one virtual disk has that disk's purpose explicitly labeled in its build document (never a silent capacity-only disk).

---

## Contributing

Found an error, an outdated link, or a step that doesn't work on a newer release? Issues and pull requests are welcome — this guide is meant to stay useful as software versions change.

---

## Disclaimer

This repository is a **free, community-facing learning guide** and is not affiliated with, nor endorsed by, Microsoft, Rocky Enterprise Software Foundation, or any other vendor referenced. All IP ranges, hostnames, and KMS endpoints shown are illustrative examples from the author's own environment — substitute your own values when following along. GVLK keys are Microsoft's publicly published generic volume license keys and require a valid, properly licensed KMS host to activate against; this guide does not provide or imply a free Windows license.
