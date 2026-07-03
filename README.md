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
- **Automation & hardening** — scripted backups, firewall/SELinux hardening, scheduled maintenance

The goal is hands-on, reproducible practice with the full lifecycle of enterprise systems administration: provisioning, configuration, security hardening, monitoring, and automation — all documented as a public runbook.

---

## Hardware requirements

| Component | Minimum | Recommended |
|---|---|---|
| CPU | 8 cores / 16 threads, VT-x / AMD-V enabled in BIOS | 12–16 cores / 24–32 threads |
| RAM | 48 GB | 64 GB |
| Storage | 800 GB free (SSD) | 1 TB NVMe |
| Network | 1 host NIC (for NAT outbound internet access) | — |

> Total VM resource footprint is approximately **46–52 GB RAM** and **~550 GB disk** across 7 virtual machines. See [Virtual machine inventory](#virtual-machine-inventory) for the full breakdown.

---

## Software requirements

Install these on the **host machine** before building any VM.

| Software | Purpose | Download |
|---|---|---|
| VMware Workstation Pro | Hypervisor (latest version) | https://www.vmware.com/products/workstation-pro.html |
| PuTTY | SSH client for Linux VMs | https://www.putty.org/ |
| mRemoteNG | Centralized RDP/SSH connection manager | https://mremoteng.org/ |
| WinSCP | SFTP file transfer to Linux VMs | https://winscp.net/ |

Detailed installation steps: [`docs/00-vmware-workstation-installation.md`](./docs/00-vmware-workstation-installation.md), [`docs/03-remote-access-tooling-setup.md`](./docs/03-remote-access-tooling-setup.md)

---

## Guest operating systems

| VM(s) | Operating system | Edition | ISO source |
|---|---|---|---|
| DC01, WINAPP01 | Windows Server 2025 | **Datacenter** (Desktop Experience) | https://www.microsoft.com/evalcenter/evaluate-windows-server-2025 |
| CLIENT01 | Windows 11 24H2 | **Pro** | https://www.microsoft.com/software-download/windows11 |
| WEB01, MON01, ELK01, LOG02 | Rocky Linux 10 | Minimal | https://rockylinux.org/download |

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
| Windows Server 2025 Datacenter | `D764K-2NDRG-47T6Q-P8T8W-YP6DF` | `kms.srv.crsoo.com:1688` |
| Windows 11 Pro | `W269N-WFGWX-YVC9B-4J6C9-T83GX` | `kms.srv.crsoo.com:1688` |

Windows Server images are provisioned from the free Evaluation ISO and converted to a retail edition via `DISM /Set-Edition` **before** activation — the full sequence (edition check, edition conversion, connectivity test, activation, verification) is documented in [`docs/04-golden-baseline-windows-server-2025.md`](./docs/04-golden-baseline-windows-server-2025.md).

> The KMS host is external to this lab's VM inventory. All Windows VMs must be able to **resolve** the KMS hostname and reach it on **TCP 1688**. Once DC01's DNS is in use as the client resolver, ensure a DNS forwarder to a public resolver is configured so KMS activation continues to work after domain join.

---

## Network architecture summary

All VMs share a single unified subnet on VMware Workstation's **NAT** network (VMnet8), re-addressed from its default range to:

```
192.168.10.0/24
```

Using NAT (instead of a separate Host-only network) keeps every VM able to reach the internet (package updates, source downloads, KMS activation) while still being fully routable to each other on the same subnet — no dual-NIC configuration is required.

Full instructions for repointing VMnet8 to this subnet: [`docs/02-network-architecture-planning.md`](./docs/02-network-architecture-planning.md)

> Feel free to substitute your own subnet — `192.168.10.0/24` is simply what this guide uses throughout for consistency between documents. If you change it, update every static IP reference accordingly.

### IP allocation

| Host | IP address | Role |
|---|---|---|
| DC01 | `192.168.10.10` | Active Directory, DNS, DHCP, NTP |
| WINAPP01 | `192.168.10.15` | IIS, SQL Server, WSUS |
| WEB01 | `192.168.10.21` | Nginx (reverse proxy / load balancer), Apache × 2, MariaDB, Suricata |
| MON01 | `192.168.10.40` | Zabbix, Wazuh |
| ELK01 | `192.168.10.50` | Elasticsearch, Logstash, Kibana |
| LOG02 | `192.168.10.51` | OpenSearch, OpenSearch Dashboards |
| CLIENT01 | DHCP (`192.168.10.100`–`200` pool) | Domain-joined test client |

---

## Virtual machine inventory

| VM | vCPU | RAM | Disk 1 (OS + swap) | Disk 2 (data) | Total disk | Role |
|---|---|---|---|---|---|---|
| DC01 | 2 | 4–6 GB | 60 GB | 40 GB *(Storage Spaces demo)* | 100 GB | AD DS, DNS, DHCP, NTP |
| WINAPP01 | 4 | 8–10 GB | 60 GB | 40 GB *(SQL Server data)* | 100 GB | IIS, SQL Server, WSUS |
| CLIENT01 | 2 | 2–4 GB | 40 GB | — | 40 GB | Domain-joined test client |
| WEB01 | 4–6 | 8 GB | 50 GB *(30 OS + 20 swap)* | 20 GB *(LVM demo)* | 70 GB | Nginx LB, Apache × 2, MariaDB, Suricata |
| MON01 | 4 | 8 GB | 60 GB *(40 OS + 20 swap)* | 20 GB *(Zabbix/Wazuh data)* | 80 GB | Zabbix, Wazuh |
| ELK01 | 4 | 8 GB | 60 GB *(40 OS + 20 swap)* | 20 GB *(Elasticsearch index data)* | 80 GB | Elasticsearch, Logstash, Kibana |
| LOG02 | 4 | 8 GB | 60 GB *(40 OS + 20 swap)* | 20 GB *(OpenSearch data)* | 80 GB | OpenSearch, OpenSearch Dashboards |
| **Total** | **24–28** | **46–52 GB** | | | **~550 GB** | |

**Notes:**
- Swap size follows a fixed rule of **2× RAM**, rounded up to the nearest 10 GB.
- Any VM with a second virtual disk uses it for a specific hands-on exercise (Storage Spaces, LVM, or dedicated data storage) rather than general capacity — this is called out explicitly in the relevant build document.
- For JVM-heavy services (MON01, ELK01, LOG02), `vm.swappiness` is tuned low despite the swap partition being present, to avoid JVM garbage-collection stalls caused by aggressive swapping.

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
| 14 | [`elk01-elasticsearch-logstash-kibana.md`](./docs/14-elk01-elasticsearch-logstash-kibana.md) | Log analytics |
| 15 | [`log02-opensearch-deployment.md`](./docs/15-log02-opensearch-deployment.md) | Log analytics |
| 16 | [`agent-deployment-all-vms.md`](./docs/16-agent-deployment-all-vms.md) | Monitoring rollout |
| 17 | [`secops-firewall-selinux-hardening.md`](./docs/17-secops-firewall-selinux-hardening.md) | Security hardening |
| 18 | [`automation-backup-scheduling.md`](./docs/18-automation-backup-scheduling.md) | Automation |

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
        ├──► MON01 (12–13)
        ├──► ELK01 (14)
        └──► LOG02 (15)
                │
                ▼
        Agent deployment (16)
                │
                ▼
        SecOps hardening (17)
                │
                ▼
        Automation & backup (18)
```

---

## Documentation conventions

- All documents are written in **English**, formatted as Markdown.
- All shell/PowerShell commands are in **English**, using fenced code blocks.
- Every piece of software built or installed **from source** includes a direct download link at the point it is first referenced — no undocumented dependencies.
- File naming follows `NN-topic-name.md`, where `NN` reflects build order, not importance.
- Any VM provisioned with more than one virtual disk has that disk's purpose explicitly labeled in its build document (never a silent capacity-only disk).

---

## Contributing

Found an error, an outdated link, or a step that doesn't work on a newer release? Issues and pull requests are welcome — this guide is meant to stay useful as software versions change.

---

## Disclaimer

This repository is a **free, community-facing learning guide** and is not affiliated with, nor endorsed by, Microsoft, Rocky Enterprise Software Foundation, or any other vendor referenced. All IP ranges, hostnames, and KMS endpoints shown are illustrative examples from the author's own environment — substitute your own values when following along. GVLK keys are Microsoft's publicly published generic volume license keys and require a valid, properly licensed KMS host to activate against; this guide does not provide or imply a free Windows license.
