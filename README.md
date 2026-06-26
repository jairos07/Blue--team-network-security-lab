# Blue Team Network Security Lab

> A fully functional defensive security lab deployed on Proxmox VE — simulating a real enterprise network with perimeter firewall, centralized logging, Active Directory, and SIEM-based threat detection.

![pfSense](https://img.shields.io/badge/pfSense-CE-003366?logo=pfsense&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh-4.14.5-blue?logo=wazuh&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_Server-24.04_LTS-E95420?logo=ubuntu&logoColor=white)
![Windows Server](https://img.shields.io/badge/Windows_Server-2022-0078D4?logo=windows&logoColor=white)
![Active Directory](https://img.shields.io/badge/Active_Directory-red.local-0078D4?logo=microsoft&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Proxmox](https://img.shields.io/badge/Proxmox-VE-E57000?logo=proxmox&logoColor=white)

---

## What is this?

This lab replicates the kind of internal network infrastructure you'd find in a small-to-medium enterprise: a hardened perimeter firewall, internal DNS/DHCP services, a Windows domain with Group Policy enforcement, and a centralized SIEM collecting logs from every endpoint.

Everything runs on Proxmox VE using isolated virtual machines interconnected through an internal bridge network (`192.168.100.0/24`). The goal is to practice real Blue Team skills — not CTF challenges, but the day-to-day defensive work: network segmentation, log correlation, domain hardening, and policy enforcement.

---

## Network Topology

```
                    ┌─────────────────────────────────────────────────────────────┐
                    │                       Proxmox VE Host                        │
                    │                  Internal Network: 192.168.100.0/24          │
                    │                                                               │
                    │  ┌──────────────┐  ┌───────────────────┐  ┌──────────────┐ │
                    │  │   pfSense    │  │   ubuntu-server   │  │    wazuh     │ │
                    │  │   Firewall   │  │   DHCP + DNS      │  │    SIEM      │ │
                    │  │              │  │   192.168.100.1   │  │ 192.168.     │ │
                    │  │  WAN: .100.2 │  │   isc-dhcp-server │  │   100.113    │ │
                    │  │              │  │   BIND9 + Agent   │  │   Docker     │ │
                    │  └──────────────┘  └───────────────────┘  └──────────────┘ │
                    │                                                               │
                    │  ┌─────────────────────────┐  ┌────────────────────────────┐│
                    │  │   win-server (WS-AD)    │  │       win-client           ││
                    │  │   Windows Server 2022   │  │    Windows 10 Pro          ││
                    │  │   Active Directory DC   │  │    Domain: red.local       ││
                    │  │   192.168.100.3         │  │    DHCP range: .10 – .50   ││
                    │  │   Domain: red.local     │  │    Wazuh Agent installed   ││
                    │  │   NetBIOS: RED          │  │                            ││
                    │  └─────────────────────────┘  └────────────────────────────┘│
                    └─────────────────────────────────────────────────────────────-┘
```

| VM | OS | Role | IP |
|---|---|---|---|
| `pfSense` | pfSense CE | Perimeter firewall / gateway | 192.168.100.1 (LAN) / 192.168.100.2 (WAN) |
| `ubuntu-server` | Ubuntu Server 24.04 LTS | DHCP + DNS (BIND9) + Wazuh Agent | 192.168.100.1 |
| `wazuh` | Ubuntu Server 24.04 LTS | SIEM — Wazuh stack via Docker | 192.168.100.113 |
| `win-server` | Windows Server 2022 Std | Active Directory Domain Controller | 192.168.100.3 |
| `win-client` | Windows 10 Pro | Domain-joined endpoint | DHCP (.10 – .50) |

---

## What this lab actually does

### Perimeter security (pfSense)
The firewall enforces a **default-deny policy** on the WAN interface. Only explicitly permitted traffic gets through: DNS queries toward the internal DNS server, DHCP requests, SSH for administration, and the three Wazuh ports (agent events, agent registration, REST API). Every rule has logging enabled, feeding events into Wazuh for correlation.

### Internal network services (Ubuntu Server)
A dedicated Ubuntu Server VM runs both `isc-dhcp-server` and BIND9 as an authoritative DNS for the `red.local` zone. It also runs a Wazuh agent, meaning the SIEM sees what happens on the infrastructure node itself — not just the Windows machines.

### SIEM (Wazuh)
Wazuh is deployed via Docker Compose on its own VM at `192.168.100.113`. Two agents report to it:
- `Ubuntu-server` — monitoring the Linux infrastructure node
- `Windows-AD` — monitoring the Active Directory domain controller

The SIEM receives events through the pfSense firewall (ports 1514/1515/55000), so any network disruption between agents and the manager is itself a detectable event.

### Identity and access management (Windows Server 2022 + AD DS)
The Windows Server VM is promoted to a Domain Controller for `red.local` (NetBIOS: `RED`). It runs AD DS with a functional level of Windows Server 2016. User accounts, Organizational Units, and Group Policy Objects are managed from here.

### Endpoint (Windows 10 Pro)
The Windows 10 client is joined to the domain and receives Group Policy from the DC. `gpupdate /force` is used to verify that policies apply correctly. Authentication flows through Kerberos against the DC at `192.168.100.3`.

---

## Tech stack

| Component | Version | Purpose |
|---|---|---|
| Proxmox VE | 8.x | Type-1 hypervisor hosting all VMs |
| pfSense CE | Latest stable | Stateful firewall, default-deny WAN policy |
| isc-dhcp-server | — | DHCP for the 192.168.100.0/24 subnet |
| BIND9 | — | Authoritative DNS for `red.local` and reverse zone |
| Wazuh | 4.14.5 | SIEM / HIDS — manager + agents via Docker |
| Windows Server 2022 | Standard Evaluation | AD DS, DNS, GPO |
| Windows 10 Pro | — | Domain-joined client endpoint |

---

## Deployed services

| Service | Host | Port(s) | Protocol |
|---|---|---|---|
| DHCP | ubuntu-server (ens19) | 67 | UDP |
| DNS (authoritative) | ubuntu-server | 53 | TCP/UDP |
| Wazuh WebGUI | wazuh | 443 | HTTPS |
| Wazuh agent events | wazuh | 1514 | UDP |
| Wazuh agent registration | wazuh | 1515 | TCP |
| Wazuh REST API | wazuh | 55000 | TCP |
| LDAP / Kerberos | win-server | 389, 636, 88 | TCP/UDP |
| SSH (admin) | ubuntu-server, wazuh | 22 | TCP |

---

## Firewall policy summary (pfSense — WAN interface)

Default policy: **Block all**. Only the following traffic is explicitly permitted:

| Action | Protocol | Source | Destination | Port | Description |
|---|---|---|---|---|---|
| Pass | TCP/UDP | 192.168.100.0/24 | 192.168.100.1 | 53 | DNS queries to internal server |
| Pass | TCP/UDP | 192.168.100.0/24 | 192.168.100.1/24 | 67 | DHCP requests |
| Pass | TCP | 192.168.100.0/24 | Any | 22 | SSH administration |
| Pass | TCP | 192.168.100.0/24 | 192.168.100.113 | 1514 | Wazuh — agent events |
| Pass | TCP | 192.168.100.0/24 | 192.168.100.113 | 1515 | Wazuh — agent registration |
| Pass | TCP | 192.168.100.0/24 | 192.168.100.113 | 55000 | Wazuh REST API |
| Block | TCP | Any | Any | * | Default deny — all other traffic |

All rules have logging enabled. See [`configs/pfsense-rules.md`](configs/pfsense-rules.md) for the full annotated ruleset.

---

## Active Directory configuration

| Parameter | Value |
|---|---|
| Domain name | `red.local` |
| NetBIOS name | `RED` |
| Forest functional level | Windows Server 2016 |
| Domain functional level | Windows Server 2016 |
| Domain Controller hostname | `WIN-HOGMAS1NJHI` |
| DC IP | `192.168.100.3` |
| Preferred DNS (clients) | `192.168.100.1` |

### Password policy (Default Domain Policy)

| Setting | Value |
|---|---|
| Password history | 5 passwords remembered |
| Maximum password age | 42 days |
| Minimum password age | 1 day |
| Minimum password length | 12 characters |
| Complexity requirements | Enabled |
| Reversible encryption | Disabled |

---

## Skills demonstrated

- **Network segmentation** — isolated internal subnet with controlled perimeter via pfSense
- **Firewall hardening** — default-deny policy with explicit allow rules and full logging
- **DNS/DHCP administration** — ISC DHCP + BIND9 authoritative zones on Linux
- **SIEM deployment** — Wazuh manager via Docker Compose, multi-agent setup
- **Active Directory** — forest creation, DC promotion, user management, OU structure
- **Group Policy** — domain-wide password policy enforced via GPO
- **Domain join workflow** — Windows client joined to AD, Kerberos authentication validated
- **Log correlation** — all firewall events, agent events and system logs forwarded to Wazuh
- **Linux server administration** — Netplan, systemctl, BIND zone files, Netplan configuration

---

## Repository structure

```
Blue--team-network-security-lab/
├── README.md
├── .env.example               # Template for Wazuh credentials
├── .gitignore
├── configs/
│   ├── docker-compose.yml     # Wazuh stack deployment
│   ├── dhcpd.conf             # ISC DHCP server configuration
│   ├── named.conf.local       # BIND9 zone declarations
│   └── pfsense-rules.md       # Annotated pfSense WAN ruleset
├── docs/
│   └── DOCUMENTACION_TECNICA.pdf   # Full step-by-step technical documentation
└── img/
    ├── pfsense/               # Firewall configuration screenshots (21)
    ├── ubuntu-server/         # DHCP/DNS setup screenshots (18)
    ├── wazuh/                 # SIEM deployment and agent registration (16)
    ├── win-server/            # AD DS, DC promotion, GPO (20)
    ├── win-client/            # Windows 10 install, domain join (15)
    └── specs/                 # VM hardware specs in Proxmox
```

---

## Full documentation

Step-by-step technical documentation with screenshots for every configuration step is available in PDF format:

**[docs/DOCUMENTACION_TECNICA.pdf](docs/DOCUMENTACION_TECNICA.pdf)**

Covers: VM specs on Proxmox → OS installation → network configuration → service deployment → AD DS setup → GPO configuration → Wazuh agent registration → final validation checklist.

---

## Disclaimer

This is an **isolated lab environment** running on a private Proxmox VE host. No services are exposed to the internet. All IP addresses, domain names, and credentials shown in documentation are for the internal lab network only and have no external exposure. This project is for educational and portfolio purposes.
