# Blue Team Network Security Lab

Laboratorio de seguridad de red orientado al equipo azul (Blue Team), construido sobre Proxmox VE. Simula una infraestructura empresarial real con firewall perimetral, servicios de red, Active Directory, cliente Windows y un SIEM para monitoreo y detección de amenazas.

---

## Arquitectura del Laboratorio

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                     Proxmox VE                          │
                    │              Red interna: 192.168.100.0/24              │
                    │                                                         │
                    │  ┌────────────┐  ┌─────────────────┐  ┌─────────────┐ │
                    │  │  pfSense   │  │  Ubuntu Server  │  │    Wazuh    │ │
                    │  │  Firewall  │  │  DHCP + DNS     │  │    SIEM     │ │
                    │  │  .1 / .2   │  │  192.168.100.1  │  │ .100.113    │ │
                    │  └────────────┘  └─────────────────┘  └─────────────┘ │
                    │                                                         │
                    │  ┌──────────────────────┐  ┌──────────────────────────┐│
                    │  │  Windows Server 2022 │  │   Windows 10 Pro Client  ││
                    │  │  Active Directory DC │  │   Unido a red.local      ││
                    │  │  192.168.100.3       │  │   DHCP (.10 – .50)       ││
                    │  └──────────────────────┘  └──────────────────────────┘│
                    └─────────────────────────────────────────────────────────┘
```

---

## Componentes

| VM | Sistema Operativo | Rol | IP |
|---|---|---|---|
| `pfSense` | pfSense CE | Firewall perimetral | 192.168.100.1 (LAN) / .2 (WAN) |
| `ubuntu-server` | Ubuntu Server 24.04 LTS | DHCP + DNS (BIND9) + Wazuh Agent | 192.168.100.1 |
| `wazuh` | Ubuntu Server 24.04 LTS | SIEM Wazuh (Docker) | 192.168.100.113 |
| `win-server` | Windows Server 2022 Std | Active Directory DC | 192.168.100.3 |
| `win-client` | Windows 10 Pro | Endpoint unido al dominio | DHCP (rango .10 – .50) |

---

## Tecnologías Utilizadas

- **pfSense Community Edition** — Firewall con reglas de seguridad por capas
- **isc-dhcp-server** — Servidor DHCP para la red interna
- **BIND9** — DNS autoritativo para la zona `red.local`
- **Wazuh 4.14.5** — SIEM / IDS open source desplegado vía Docker
- **Windows Server 2022** — Active Directory Domain Services (AD DS)
- **Group Policy Objects (GPO)** — Políticas de contraseñas y seguridad del dominio
- **Proxmox VE** — Hipervisor de virtualización tipo 1

---

## Dominio Active Directory

| Parámetro | Valor |
|---|---|
| Nombre del dominio | `red.local` |
| Nombre NetBIOS | `RED` |
| Nivel funcional del bosque | Windows Server 2016 |
| Nivel funcional del dominio | Windows Server 2016 |
| Controlador de dominio | `WIN-HOGMAS1NJHI` |
| IP del DC | `192.168.100.3` |
| DNS preferido (clientes) | `192.168.100.1` |

---

## Reglas de Firewall (pfSense — Interfaz WAN)

| Acción | Protocolo | Origen | Destino | Puerto | Descripción |
|---|---|---|---|---|---|
| Block | * | Bogon networks | * | * | Block bogon networks (default) |
| Block | TCP | Any | Any | * | Denegación por defecto |
| Pass | TCP | 192.168.100.0/24 | Any | 22 | SSH |
| Pass | TCP/UDP | 192.168.100.0/24 | 192.168.100.1/24 | 67 | DHCP |
| Pass | TCP/UDP | 192.168.100.0 | 192.168.100.1 | 53 | DNS |
| Pass | TCP | 192.168.100.0/24 | 192.168.100.113 | 1514 | Wazuh — eventos del agente |
| Pass | TCP | 192.168.100.0/24 | 192.168.100.113 | 1515 | Wazuh — registro de agentes |
| Pass | TCP | 192.168.100.0/24 | 192.168.100.113 | 55000 | Wazuh API |

---

## Políticas de Seguridad (GPO — Default Domain Policy)

| Política | Valor |
|---|---|
| Historial de contraseñas | 5 contraseñas recordadas |
| Antigüedad máxima | 42 días |
| Antigüedad mínima | 1 día |
| Longitud mínima | 12 caracteres |
| Complejidad requerida | Habilitada |
| Cifrado reversible | Deshabilitado |

---

## Agentes Wazuh Desplegados

| Nombre del agente | Plataforma | Versión | Manager |
|---|---|---|---|
| `Ubuntu-server` | Linux DEB amd64 | 4.14.5 | 192.168.100.113 |
| `Windows-AD` | Windows MSI 64-bit | 4.14.5 | 192.168.100.113 |

---

## Estructura del Repositorio

```
Blue--team-network-security-lab/
└── img/
    ├── pfsense/          # Configuración del firewall (21 capturas)
    ├── ubuntu-server/    # Instalación y configuración DHCP/DNS (18 capturas)
    ├── wazuh/            # Despliegue SIEM y registro de agentes (16 capturas)
    ├── win-server/       # Instalación AD DS, DC y políticas (20 capturas)
    ├── win-client/       # Instalación Windows 10 y unión al dominio (15 capturas)
    └── specs/            # Especificaciones de hardware de cada VM
        ├── pfsense/
        ├── ubuntu-server/
        ├── wazuh/
        ├── win-server/
        └── win-client/
```

---

## Requisitos Previos

- Proxmox VE 7.x o superior
- ISOs necesarios:
  - pfSense CE (última versión estable)
  - Ubuntu Server 24.04 LTS
  - Windows Server 2022 Evaluation (x64)
  - Windows 10 Pro (x64)
- Mínimo **16 GB RAM** en el host Proxmox
- Red virtual interna configurada (`vmbr1` o equivalente en 192.168.100.0/24)
