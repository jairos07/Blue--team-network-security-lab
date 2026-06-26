# pfSense Firewall Rules — WAN Interface

## Policy: Default Deny

All traffic entering or traversing the WAN interface is **blocked by default**.
Only the services explicitly listed below are permitted.
Every rule has logging enabled so that all allowed (and blocked) traffic is forwarded to Wazuh for correlation.

---

## Ruleset (evaluated top to bottom)

| # | Action | Protocol | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Block | * | Bogon networks | * | * | — | Block bogon/reserved address space (pfSense built-in) |
| 2 | Block | TCP | Any | Any | * | ✓ | **Default deny** — drop all traffic not matched by rules below |
| 3 | Pass | TCP | 192.168.100.0/24 | Any | 22 | ✓ | SSH — admin access from internal network to any host |
| 4 | Pass | TCP/UDP | 192.168.100.0/24 | 192.168.100.1/24 | 67 | ✓ | DHCP — clients request IP assignment from ubuntu-server |
| 5 | Pass | TCP/UDP | 192.168.100.0/24 | 192.168.100.1 | 53 | ✓ | DNS — name resolution queries to ubuntu-server (BIND9) |
| 6 | Pass | TCP | 192.168.100.0/24 | 192.168.100.113 | 1514 | ✓ | Wazuh — agent event forwarding to SIEM manager |
| 7 | Pass | TCP | 192.168.100.0/24 | 192.168.100.113 | 1515 | ✓ | Wazuh — agent registration and key exchange |
| 8 | Pass | TCP | 192.168.100.0/24 | 192.168.100.113 | 55000 | ✓ | Wazuh — REST API access for management and queries |

---

## Rule notes

### Rule 2 — Default deny
Placed above the explicit `Pass` rules to establish the baseline deny posture. The logging on this rule means every rejected connection attempt is captured and available in Wazuh for analysis. In pfSense, this is configured as:
- **Action:** Block
- **Interface:** WAN
- **Protocol:** TCP
- **Source/Destination:** Any / Any
- **Description:** `Denegacion por defecto, bloquear todo el otro trafico`

### Rules 6–8 — Wazuh ports
These three rules permit the full Wazuh communication stack:
- **1514/UDP** — agents send security events (syslogs, file integrity, process monitoring)
- **1515/TCP** — new agents authenticate and exchange keys with the manager during enrollment
- **55000/TCP** — REST API used by the Wazuh dashboard and automation tooling

All three rules target `192.168.100.113` (the Wazuh VM) specifically, not `Any`, following the principle of least privilege.

### DNS rule (Rule 5)
Permits DNS queries only toward `192.168.100.1` (ubuntu-server running BIND9). External DNS resolution is not permitted through the firewall from internal hosts; all queries must go through the internal authoritative server.

---

## Wazuh agents registered

| Agent name | Platform | Manager address | Communication port |
|---|---|---|---|
| `Ubuntu-server` | Linux DEB amd64 | 192.168.100.113 | 1514 |
| `Windows-AD` | Windows MSI 64-bit | 192.168.100.113 | 1514 |
