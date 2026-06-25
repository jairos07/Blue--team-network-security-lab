# Documentación Técnica — Blue Team Network Security Lab

---

## Índice

1. [Introducción](#1-introducción)
2. [Topología de Red](#2-topología-de-red)
3. [pfSense — Firewall Perimetral](#3-pfsense--firewall-perimetral)
   - 3.1 [Especificaciones de la VM](#31-especificaciones-de-la-vm)
   - 3.2 [Configuración inicial (Setup Wizard)](#32-configuración-inicial-setup-wizard)
   - 3.3 [Reglas de Firewall](#33-reglas-de-firewall)
4. [Ubuntu Server — DHCP y DNS](#4-ubuntu-server--dhcp-y-dns)
   - 4.1 [Especificaciones de la VM](#41-especificaciones-de-la-vm)
   - 4.2 [Instalación del SO](#42-instalación-del-so)
   - 4.3 [Configuración de Red](#43-configuración-de-red)
   - 4.4 [Servidor DHCP (isc-dhcp-server)](#44-servidor-dhcp-isc-dhcp-server)
   - 4.5 [Servidor DNS (BIND9)](#45-servidor-dns-bind9)
5. [Wazuh — SIEM](#5-wazuh--siem)
   - 5.1 [Especificaciones de la VM](#51-especificaciones-de-la-vm)
   - 5.2 [Instalación del SO](#52-instalación-del-so)
   - 5.3 [Despliegue con Docker Compose](#53-despliegue-con-docker-compose)
   - 5.4 [Registro de Agentes](#54-registro-de-agentes)
6. [Windows Server 2022 — Active Directory](#6-windows-server-2022--active-directory)
   - 6.1 [Especificaciones de la VM](#61-especificaciones-de-la-vm)
   - 6.2 [Instalación del SO](#62-instalación-del-so)
   - 6.3 [Configuración de Red](#63-configuración-de-red)
   - 6.4 [Instalación de AD DS](#64-instalación-de-ad-ds)
   - 6.5 [Promoción a Controlador de Dominio](#65-promoción-a-controlador-de-dominio)
   - 6.6 [Creación de Usuarios](#66-creación-de-usuarios)
   - 6.7 [Políticas de Grupo (GPO)](#67-políticas-de-grupo-gpo)
7. [Windows 10 Pro — Cliente del Dominio](#7-windows-10-pro--cliente-del-dominio)
   - 7.1 [Especificaciones de la VM](#71-especificaciones-de-la-vm)
   - 7.2 [Instalación del SO](#72-instalación-del-so)
   - 7.3 [Verificación de Red](#73-verificación-de-red)
   - 7.4 [Unión al Dominio](#74-unión-al-dominio)
   - 7.5 [Aplicación de Directivas de Grupo](#75-aplicación-de-directivas-de-grupo)
8. [Verificación del Laboratorio](#8-verificación-del-laboratorio)

---

## 1. Introducción

Este documento describe la configuración técnica paso a paso de cada componente del laboratorio Blue Team, acompañada de las capturas de pantalla correspondientes. El laboratorio simula una infraestructura empresarial real con medidas de seguridad defensiva: firewall perimetral, servicios DHCP/DNS, Active Directory con GPO y un SIEM para detección de eventos.

**Dominio:** `red.local` | **Red interna:** `192.168.100.0/24` | **Hipervisor:** Proxmox VE

---

## 2. Topología de Red

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

| VM | Rol | IP |
|---|---|---|
| pfSense | Firewall / Gateway | 192.168.100.1 (LAN) / 192.168.100.2 (WAN) |
| ubuntu-server | DHCP + DNS + Wazuh Agent | 192.168.100.1 |
| wazuh | SIEM (Docker) | 192.168.100.113 |
| win-server | Active Directory DC | 192.168.100.3 |
| win-client | Endpoint de usuario | DHCP (rango .10 – .50) |

---

## 3. pfSense — Firewall Perimetral

### 3.1 Especificaciones de la VM

![pfSense specs cap1](img/specs/pfsense/cap1.png)
*Especificaciones de hardware de la VM pfSense en Proxmox: configuración de CPU y memoria.*

![pfSense specs cap2](img/specs/pfsense/cap2.png)
*Configuración de almacenamiento asignado a la VM pfSense.*

![pfSense specs cap3](img/specs/pfsense/cap3.png)
*Configuración de interfaces de red de pfSense en Proxmox (adaptadores WAN y LAN).*

![pfSense specs cap4](img/specs/pfsense/cap4.png)
*Vista general de la VM pfSense en el panel de Proxmox.*

![pfSense specs cap5](img/specs/pfsense/cap5.png)
*Detalles adicionales de la VM pfSense: opciones de arranque y hardware.*

![pfSense specs cap6](img/specs/pfsense/cap6.png)
*Resumen de recursos asignados a pfSense en Proxmox.*

![pfSense specs cap7](img/specs/pfsense/cap7.png)
*Configuración final de la VM pfSense antes del primer arranque.*

---

### 3.2 Configuración inicial (Setup Wizard)

![pfSense cap1](img/pfsense/cap1.png)
*Pantalla de bienvenida de pfSense durante el primer arranque. Se inicia el asistente de configuración vía consola.*

![pfSense cap2](img/pfsense/cap2.png)
*Selección de las interfaces WAN y LAN durante la configuración inicial por consola.*

![pfSense cap3](img/pfsense/cap3.png)
*Asignación de interfaces completada. pfSense detecta los adaptadores de red disponibles.*

![pfSense cap4](img/pfsense/cap4.png)
*Configuración de la IP estática de la interfaz LAN vía consola antes de acceder al WebGUI.*

![pfSense cap5](img/pfsense/cap5.png)
*Confirmación de la IP de la interfaz LAN y URL de acceso al panel web de pfSense.*

![pfSense cap6](img/pfsense/cap6.png)
*Primer acceso al WebGUI de pfSense. Pantalla de login con las credenciales por defecto (admin/pfsense).*

![pfSense cap7](img/pfsense/cap7.png)
*Dashboard de pfSense tras el login. Se aprecia el estado de las interfaces y la versión de pfSense CE instalada.*

![pfSense cap8](img/pfsense/cap8.png)
*Inicio del asistente de configuración (Setup Wizard) paso 1 de 9: pantalla de bienvenida.*

![pfSense cap9](img/pfsense/cap9.png)
*Setup Wizard — Paso 2/9: Información General. Se configura el hostname `pfSense`, el dominio `red.local` y el DNS primario `192.168.100.1`. Se activa Override DNS para permitir que el DHCP/PPP sobreescriba el DNS.*

![pfSense cap10](img/pfsense/cap10.png)
*Setup Wizard — Paso 4/9: Configuración de la interfaz WAN. Tipo estático con IP `192.168.100.2/24` y gateway `192.168.100.1`. Los campos MAC, MTU y MSS se dejan en blanco (valores por defecto).*

![pfSense cap11](img/pfsense/cap11.png)
*Setup Wizard — Paso 5/9: Configuración de la interfaz LAN. Se asigna la IP `192.168.100.1/24`, que actuará como gateway por defecto de toda la red interna del laboratorio.*

![pfSense cap12](img/pfsense/cap12.png)
*Setup Wizard — Paso 6/9: Establecimiento de la contraseña del administrador del WebGUI. Esta contraseña también se utiliza para el acceso SSH si está habilitado.*

---

### 3.3 Reglas de Firewall

Las reglas se aplican sobre la interfaz **WAN** con política base de **denegación por defecto**. Se añaden reglas `Pass` explícitas únicamente para el tráfico necesario. El logging está habilitado en todas las reglas para su correlación en Wazuh.

![pfSense cap13](img/pfsense/cap13.png)
*Edición de la regla de firewall para **DNS** (puerto 53 TCP/UDP). Acción: Pass. Interfaz: WAN. Origen: red `192.168.100.0`. Destino: `192.168.100.1` (ubuntu-server, servidor DNS). Permite la resolución de nombres desde la red interna.*

![pfSense cap14](img/pfsense/cap14.png)
*Opciones adicionales de la regla DNS: logging activado (`Log packets that are handled by this rule`) y descripción `DNS`. El logging permite auditar las consultas DNS en el SIEM.*

![pfSense cap15](img/pfsense/cap15.png)
*Edición de la regla para **DHCP** (puerto 67 TCP/UDP). Acción: Pass. Interfaz: WAN. Origen: `192.168.100.0/24`. Destino: `192.168.100.1/24` (ubuntu-server). Logging habilitado. Permite que los clientes soliciten asignación de IP al servidor DHCP.*

![pfSense cap16](img/pfsense/cap16.png)
*Edición de la regla para **SSH** (puerto 22 TCP). Acción: Pass. Interfaz: WAN. Origen: `192.168.100.0/24`. Destino: Any. Logging habilitado. Permite el acceso SSH desde cualquier equipo de la red interna.*

![pfSense cap17](img/pfsense/cap17.png)
*Regla de **denegación por defecto**. Acción: Block. Interfaz: WAN. Protocolo TCP. Origen: Any. Destino: Any. Descripción: "Denegacion por defecto, bloquear todo el otro trafico". Logging habilitado. Esta regla bloquea silenciosamente todo el tráfico no explícitamente permitido.*

![pfSense cap18](img/pfsense/cap18.png)
*Vista completa del conjunto de **reglas WAN** aplicadas. De arriba abajo: (1) Block bogon networks, (2) Block default — denegar todo, (3) Pass SSH puerto 22 desde 192.168.100.0/24, (4) Pass DHCP puerto 67 hacia 192.168.100.1/24, (5) Pass DNS puerto 53 hacia 192.168.100.1. Hay cambios pendientes de aplicar (botón "Apply Changes").*

![pfSense cap19](img/pfsense/cap19.png)
*Regla para el puerto **1514 de Wazuh** (envío de eventos del agente). Acción: Pass. TCP. Origen: `192.168.100.0/24`. Destino: `192.168.100.113` (servidor Wazuh). Puerto: 1514. Logging habilitado. Permite que los agentes Wazuh envíen eventos al SIEM.*

![pfSense cap20](img/pfsense/cap20.png)
*Regla para el puerto **1515 de Wazuh** (registro de agentes). Acción: Pass. TCP. Origen: `192.168.100.0/24`. Destino: `192.168.100.113`. Puerto: 1515. Logging habilitado. Permite que los agentes se registren contra el manager de Wazuh.*

![pfSense cap21](img/pfsense/cap21.png)
*Regla para el puerto **55000 de Wazuh** (API REST). Acción: Pass. TCP. Origen: `192.168.100.0/24`. Destino: `192.168.100.113`. Puerto: 55000. Logging habilitado. Permite el acceso a la API de Wazuh para gestión de agentes y consultas.*

---

## 4. Ubuntu Server — DHCP y DNS

### 4.1 Especificaciones de la VM

![ubuntu-server specs cap1](img/specs/ubuntu-server/cap1.png)
*Especificaciones de hardware de la VM ubuntu-server en Proxmox: CPU y RAM asignados.*

![ubuntu-server specs cap2](img/specs/ubuntu-server/cap2.png)
*Configuración de disco asignado a ubuntu-server.*

![ubuntu-server specs cap3](img/specs/ubuntu-server/cap3.png)
*Primera interfaz de red de ubuntu-server (ens18) conectada al bridge de gestión de Proxmox.*

![ubuntu-server specs cap4](img/specs/ubuntu-server/cap4.png)
*Segunda interfaz de red (ens19) conectada al bridge de la red interna del laboratorio (vmbr1).*

![ubuntu-server specs cap5](img/specs/ubuntu-server/cap5.png)
*Vista general de la VM ubuntu-server en el panel de Proxmox.*

![ubuntu-server specs cap6](img/specs/ubuntu-server/cap6.png)
*Resumen de recursos asignados a ubuntu-server.*

![ubuntu-server specs cap7](img/specs/ubuntu-server/cap7.png)
*Configuración final de la VM ubuntu-server antes del primer arranque.*

---

### 4.2 Instalación del SO

![ubuntu-server cap1](img/ubuntu-server/cap1.png)
*Pantalla de selección de idioma del instalador de Ubuntu Server 24.04 LTS. Se selecciona **Español** como idioma de instalación para el sistema.*

![ubuntu-server cap2](img/ubuntu-server/cap2.png)
*Configuración del perfil de usuario. Se establece: nombre completo `administrador`, hostname del servidor `ubuntu-server`, nombre de usuario `administrador` y contraseña. Este es el usuario principal con acceso sudo.*

![ubuntu-server cap3](img/ubuntu-server/cap3.png)
*Configuración SSH durante la instalación. Se selecciona **Instalar servidor OpenSSH** y se habilita la **autenticación por contraseña**. No se importan claves SSH autorizadas en este paso.*

---

### 4.3 Configuración de Red

![ubuntu-server cap4](img/ubuntu-server/cap4.png)
*Diálogo de Proxmox para añadir un **segundo dispositivo de red** a la VM ubuntu-server. Se selecciona el bridge `vmbr1` (red interna del laboratorio), modelo VirtIO paravirtualizado y se activa el cortafuegos de Proxmox para esta interfaz.*

![ubuntu-server cap5](img/ubuntu-server/cap5.png)
*Salida de `ip a` mostrando el estado inicial de las interfaces. `ens18` está UP con IP `192.168.1.226/24` (asignada por DHCP del host Proxmox). `ens19` está en estado DOWN — aún sin configuración estática.*

![ubuntu-server cap6](img/ubuntu-server/cap6.png)
*Edición del archivo `/etc/netplan/50-cloud-init.yaml` con `nano`. Se configura la IP estática de ambas interfaces: `ens18` con `192.168.1.200/24`, gateway `192.168.1.1` y DNS `8.8.8.8`/`8.8.4.4` (acceso de gestión); `ens19` con `192.168.100.1/24` (red interna del laboratorio, sin gateway).*

![ubuntu-server cap7](img/ubuntu-server/cap7.png)
*Aplicación de la configuración de red con `sudo netplan apply` y verificación con `ip a`. Ambas interfaces están ahora UP: `ens18` con `192.168.1.200/24` y `ens19` con `192.168.100.1/24`. La red interna del laboratorio está operativa.*

---

### 4.4 Servidor DHCP (isc-dhcp-server)

![ubuntu-server cap8](img/ubuntu-server/cap8.png)
*Instalación del paquete `isc-dhcp-server` con `sudo apt install isc-dhcp-server`. El gestor de paquetes descarga e instala el servidor DHCP junto con `isc-dhcp-common` (1.281 KB, 4.281 KB en disco).*

![ubuntu-server cap9](img/ubuntu-server/cap9.png)
*Edición de `/etc/default/isc-dhcp-server` con `nano`. Se especifica `INTERFACESv4="ens19"` para que el servicio DHCP escuche únicamente en la interfaz de la red interna del laboratorio. `INTERFACESv6` se deja vacío.*

![ubuntu-server cap10](img/ubuntu-server/cap10.png)
*Edición de `/etc/dhcp/dhcpd.conf` con `nano`. Se configura el pool DHCP para la subred `192.168.100.0/255.255.255.0`: rango de IPs `192.168.100.10` – `192.168.100.50`, gateway `192.168.100.1`, máscara `255.255.255.0` y DNS `8.8.8.8`/`8.8.4.4`. Lease por defecto 602s, máximo 7200s. Directiva `authoritative` activa.*

![ubuntu-server cap11](img/ubuntu-server/cap11.png)
*Reinicio, habilitación y verificación del estado del servicio DHCP. `systemctl status isc-dhcp-server` muestra el servicio **active (running)** desde el 27/05/2026. El daemon `dhcpd` escucha en la interfaz `ens19/192.168.100.0/24`. La base de datos de leases está en `/var/lib/dhcp/dhcpd.leases`.*

---

### 4.5 Servidor DNS (BIND9)

![ubuntu-server cap12](img/ubuntu-server/cap12.png)
*Instalación de BIND9 con `sudo apt install bind9 bind9utils bind9-doc`. Se instalan los paquetes `bind9`, `bind9-doc`, `bind9utils`, `bind9utils` y `dns-root-data` (3.691 KB descargados, 9.371 KB en disco).*

![ubuntu-server cap13](img/ubuntu-server/cap13.png)
*Edición de `/etc/bind/named.conf.local` con `nano`. Se declaran dos zonas DNS: zona directa `red.local` (tipo master, archivo `/etc/bind/db.red.local`) y zona inversa `100.168.192.in-addr.arpa` (tipo master, archivo `/etc/bind/db.192.168.100`). Estas zonas hacen a este servidor autoritativo para el dominio del laboratorio.*

![ubuntu-server cap14](img/ubuntu-server/cap14.png)
*Creación del archivo de zona directa copiando la plantilla por defecto: `sudo cp /etc/bind/db.local /etc/bind/db.red.local`. Este archivo se editará a continuación para definir los registros DNS de `red.local`.*

![ubuntu-server cap15](img/ubuntu-server/cap15.png)
*Edición de `/etc/bind/db.red.local`. Se configura el registro SOA con `ns.red.local.` como nameserver primario y `admin.red.local.` como contacto. Serial: 2. Se añaden registros: `NS` apuntando a `localhost`, `A` para `ns` → `192.168.100.1` y `A` para `server` → `192.168.100.1`.*

![ubuntu-server cap16](img/ubuntu-server/cap16.png)
*Creación del archivo de zona inversa copiando la plantilla loopback: `sudo cp /etc/bind/db.127 /etc/bind/db.192.168.100`. Este archivo se editará para la resolución inversa (PTR) de la subred `192.168.100.0/24`.*

![ubuntu-server cap17](img/ubuntu-server/cap17.png)
*Edición de `/etc/bind/db.192.168.100`. Zona inversa con registro SOA (`ns.red.local.`/`admin.red.local.`), registro `NS` apuntando a `ns.red.local.` y registro `PTR` `1.0.0 → ns.red.local.` para la resolución inversa de `192.168.100.1`.*

![ubuntu-server cap18](img/ubuntu-server/cap18.png)
*Inicio del servicio BIND9 y verificación con `systemctl status bind9`. El servicio muestra estado **active (running)** desde el 27/05/2026. El proceso `named` carga la zona `red.local` y envía notificaciones. Algunos mensajes de "network unreachable" para IPv6 son esperados en un entorno sin conectividad IPv6.*

---

## 5. Wazuh — SIEM

### 5.1 Especificaciones de la VM

![wazuh specs cap1](img/specs/wazuh/cap1.png)
*Especificaciones de hardware de la VM wazuh en Proxmox: CPU y RAM asignados.*

![wazuh specs cap2](img/specs/wazuh/cap2.png)
*Configuración de almacenamiento de la VM wazuh en Proxmox.*

![wazuh specs cap3](img/specs/wazuh/cap3.png)
*Interfaz de red de la VM wazuh conectada a la red interna del laboratorio.*

![wazuh specs cap4](img/specs/wazuh/cap4.png)
*Vista general de la VM wazuh en el panel de Proxmox.*

![wazuh specs cap5](img/specs/wazuh/cap5.png)
*Resumen de recursos asignados a la VM wazuh.*

![wazuh specs cap6](img/specs/wazuh/cap6.png)
*Opciones de arranque y configuración adicional de la VM wazuh.*

![wazuh specs cap7](img/specs/wazuh/cap7.png)
*Configuración final de la VM wazuh antes del primer arranque.*

---

### 5.2 Instalación del SO

![wazuh cap1](img/wazuh/cap1.png)
*Selección de idioma del instalador de Ubuntu Server 24.04 LTS para la VM wazuh. Se selecciona **Español** como idioma del sistema.*

![wazuh cap2](img/wazuh/cap2.png)
*Configuración del perfil de usuario de la VM wazuh. Hostname: `wazuh`. Usuario: `administrador`. Contraseña configurada para acceso local y SSH.*

![wazuh cap3](img/wazuh/cap3.png)
*Configuración SSH durante la instalación de la VM wazuh. Se instala OpenSSH Server y se habilita la autenticación por contraseña para acceso remoto.*

---

### 5.3 Despliegue con Docker Compose

![wazuh cap4](img/wazuh/cap4.png)
*Instalación de `docker-compose` con `apt install docker-compose`. Se instalan los paquetes necesarios incluyendo `python3-docker`, `python3-dockerpty` y dependencias de Python. Tamaño de descarga: 297 KB.*

![wazuh cap5](img/wazuh/cap5.png)
*Contenido del archivo `docker-compose.yml`. Define el servicio `wazuh` usando la imagen `ghcr.io/wazuh/wazuh:latest` con nombre de contenedor `wazuh`. Puertos expuestos: `443:443` (WebGUI HTTPS), `1514:1514/udp` (eventos de agentes), `1515:1515` (registro de agentes). Variables de entorno: `INDEXER_USERNAME=admin` y `INDEXER_PASSWORD=SecretPassword`. Política de reinicio: `always`.*

![wazuh cap6](img/wazuh/cap6.png)
*Ejecución de `docker-compose up -d` para levantar el contenedor Wazuh en segundo plano. Docker crea la red `administrador_default` y comienza a descargar la imagen `ghcr.io/wazuh/wazuh:latest`.*

![wazuh cap7](img/wazuh/cap7.png)
*Pantalla de login de la interfaz web de Wazuh accesible en `https://192.168.100.113`. Se introduce el usuario `admin` y la contraseña configurada. Wazuh se identifica como "The Open Source Security Platform".*

---

### 5.4 Registro de Agentes

![wazuh cap8](img/wazuh/cap8.png)
*Asistente **Deploy new agent** de la interfaz web de Wazuh. Permite seleccionar la plataforma del agente (Linux, Windows, macOS), la dirección del manager y el nombre del agente. Formulario en blanco antes de configurar el primer agente.*

![wazuh cap9](img/wazuh/cap9.png)
*Configuración del agente para **ubuntu-server**. Plataforma: Linux RPM amd64. Dirección del manager: `192.168.100.1`. Nombre del agente: `Ubuntu-server`. Wazuh genera automáticamente el comando de instalación en el paso siguiente.*

![wazuh cap10](img/wazuh/cap10.png)
*Paso 4 del asistente: comando de instalación del agente para ubuntu-server. El comando usa `curl` para descargar el paquete RPM de Wazuh 4.14.5 y lo instala con las variables `WAZUH_MANAGER='192.168.100.1'` y `WAZUH_AGENT_NAME='Ubuntu-server'`. Paso 5: comandos para habilitar e iniciar el agente con systemctl.*

![wazuh cap12](img/wazuh/cap12.png)
*Instalación del agente Wazuh en **ubuntu-server** (paquete DEB para Ubuntu). Se descarga `wazuh-agent_4.14.5-1_amd64.deb` con `wget` y se instala con `dpkg` pasando las variables `WAZUH_MANAGER` y `WAZUH_AGENT_NAME='ubuntu-server'`.*

![wazuh cap13](img/wazuh/cap13.png)
*Activación del agente Wazuh en ubuntu-server. Se ejecutan `systemctl daemon-reload`, `systemctl enable wazuh-agent` y `systemctl start wazuh-agent`. El estado muestra **active (running)** con todos los subprocesos activos: `wazuh-execd`, `wazuh-agentd`, `wazuh-syscheckd`, `wazuh-logcollector` y `wazuh-modulesd`.*

![wazuh cap15](img/wazuh/cap15.png)
*Configuración del agente para **Windows Server (AD)**. Plataforma: Windows MSI 32/64-bit. Dirección del manager: `192.168.100.3`. Nombre del agente: `Windows-AD`. Wazuh genera el comando PowerShell de instalación para ejecutar en el Windows Server.*

![wazuh cap16](img/wazuh/cap16.png)
*Instalación del agente Wazuh en **Windows Server** desde PowerShell (Administrador). Se descarga el instalador MSI con `Invoke-WebRequest` y se instala con los parámetros `/q WAZUH_MANAGER='192.168.100.3' WAZUH_AGENT_NAME='Windows-AD'`. A continuación `NET START Wazuh` inicia el servicio: "The Wazuh service was started successfully."*

---

## 6. Windows Server 2022 — Active Directory

### 6.1 Especificaciones de la VM

![win-server specs cap1](img/specs/win-server/cap1.png)
*Especificaciones de hardware de la VM Windows Server en Proxmox: CPU y RAM asignados.*

![win-server specs cap2](img/specs/win-server/cap2.png)
*Configuración de almacenamiento de la VM Windows Server (60 GB).*

![win-server specs cap3](img/specs/win-server/cap3.png)
*Interfaz de red de la VM Windows Server conectada a la red interna del laboratorio.*

![win-server specs cap4](img/specs/win-server/cap4.png)
*Vista general de la VM Windows Server 2022 en el panel de Proxmox.*

![win-server specs cap5](img/specs/win-server/cap5.png)
*Opciones avanzadas de la VM Windows Server en Proxmox.*

![win-server specs cap6](img/specs/win-server/cap6.png)
*Resumen de recursos de la VM Windows Server.*

![win-server specs cap7](img/specs/win-server/cap7.png)
*Configuración de opciones de arranque de la VM Windows Server.*

![win-server specs cap8](img/specs/win-server/cap8.png)
*Configuración final de la VM Windows Server antes del primer arranque.*

---

### 6.2 Instalación del SO

![win-server cap1](img/win-server/cap1.png)
*Pantalla inicial del instalador de Windows Server 2022. Configuración regional: idioma `English (United States)`, formato de hora y moneda `Spanish (Spain, International Sort)`, teclado `Spanish`. Se mantiene el idioma del sistema en inglés para mayor compatibilidad con herramientas de administración.*

![win-server cap2](img/win-server/cap2.png)
*Selección de la edición del sistema operativo. Se elige **Windows Server 2022 Standard Evaluation (Desktop Experience)** (x64, 3/3/2022) para disponer de interfaz gráfica completa, necesaria para la configuración del Active Directory.*

![win-server cap3](img/win-server/cap3.png)
*Selección del disco de instalación. Se muestra `Drive 0` con `60.0 GB` de espacio sin asignar. Se selecciona como destino de instalación.*

![win-server cap4](img/win-server/cap4.png)
*Progreso de instalación de Windows Server 2022 al 55%. El instalador copia archivos, instala características y actualizaciones antes de finalizar.*

![win-server cap5](img/win-server/cap5.png)
*Pantalla de personalización post-instalación. Se establece la contraseña para la cuenta `Administrator` incorporada del sistema. Esta cuenta es el acceso inicial al servidor.*

---

### 6.3 Configuración de Red

![win-server cap6](img/win-server/cap6.png)
*Configuración de la IP estática del servidor. En **Configuración → Red e Internet → Ethernet → Propiedades → IPv4**: IP `192.168.100.3`, máscara `255.255.255.0`, gateway `192.168.100.1` y DNS preferido `192.168.100.1` (ubuntu-server). Esta IP será la del Controlador de Dominio.*

---

### 6.4 Instalación de AD DS

![win-server cap7](img/win-server/cap7.png)
*Asistente **Add Roles and Features** en Server Manager → Server Roles. Se visualizan los roles disponibles. Se seleccionará **Active Directory Domain Services** para convertir el servidor en un controlador de dominio.*

![win-server cap8](img/win-server/cap8.png)
*Página de confirmación de instalación del rol AD DS. Se instalarán: `Active Directory Domain Services`, `Group Policy Management`, `Remote Server Administration Tools`, `AD DS and AD LDS Tools`, `Active Directory Administrative Center` y `AD DS Snap-Ins and Command-Line Tools`. Se marca la opción de reinicio automático si es necesario.*

![win-server cap9](img/win-server/cap9.png)
*Dashboard de Server Manager tras completar la instalación del rol AD DS. Aparece la notificación de configuración post-despliegue: *"Configuration required for Active Directory Domain Services at WIN-HOGMAS1NJHI"* con el enlace **Promote this server to a domain controller**.*

---

### 6.5 Promoción a Controlador de Dominio

![win-server cap10](img/win-server/cap10.png)
*Asistente **AD DS Configuration Wizard** — Deployment Configuration. Se selecciona **Add a new forest** para crear un nuevo bosque de Active Directory. Root domain name: `red.local`. Servidor destino: `WIN-HOGMAS1NJHI`.*

![win-server cap11](img/win-server/cap11.png)
*Domain Controller Options. Forest y Domain functional level: **Windows Server 2016**. Capacidades del DC: **Domain Name System (DNS) server** y **Global Catalog (GC)** activados. No se configura como RODC (Read-only). Se establece la contraseña DSRM (Directory Services Restore Mode).*

![win-server cap12](img/win-server/cap12.png)
*Additional Options. El asistente asigna automáticamente el nombre NetBIOS del dominio: **RED**. Este nombre se usará para el inicio de sesión en formato `RED\usuario`.*

![win-server cap13](img/win-server/cap13.png)
*Paths — Ubicaciones de los archivos de AD DS. Base de datos NTDS: `C:\Windows\NTDS`. Archivos de log: `C:\Windows\NTDS`. Carpeta SYSVOL: `C:\Windows\SYSVOL`. Se mantienen las rutas por defecto.*

![win-server cap14](img/win-server/cap14.png)
*Prerequisites Check — verificación de requisitos previos en curso. El asistente valida que el servidor cumple todos los requisitos antes de proceder con la instalación del controlador de dominio.*

![win-server cap15](img/win-server/cap15.png)
*Prerequisites Check completado: **"All prerequisite checks passed successfully. Click 'Install' to begin installation."** Se muestran avisos informativos sobre algoritmos de criptografía y adaptadores de red sin IP estática IPv6, que no bloquean la instalación. Se puede proceder con el botón **Install**.*

![win-server cap16](img/win-server/cap16.png)
*El servidor se reinicia automáticamente tras la promoción a Controlador de Dominio. Pantalla de "Restarting". Tras el reinicio, el servidor habrá adoptado el rol de DC del bosque `red.local`.*

---

### 6.6 Creación de Usuarios

![win-server cap17](img/win-server/cap17.png)
*Creación de un nuevo usuario del dominio desde **Server Manager → Tools → Active Directory Users and Computers → red.local/Users → New Object - User**. Se completan: First name `usuario`, Last name `1`, Full name `usuario 1`, User logon name `u1@red.local`, logon pre-Windows 2000 `RED\u1`.*

![win-server cap18](img/win-server/cap18.png)
*Pantalla de confirmación de creación del usuario. Se muestra el resumen: usuario `usuario 1` con logon `u1@red.local` en la OU `red.local/Users`. La opción **"El usuario debe cambiar la contraseña en el próximo inicio de sesión"** está activa, forzando el cambio de credencial en el primer login.*

---

### 6.7 Políticas de Grupo (GPO)

![win-server cap19](img/win-server/cap19.png)
*Group Policy Management Editor — Default Domain Policy. Vista del árbol de configuración navegando a `Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy`. Se visualiza el panel de configuración de la política de contraseñas del dominio.*

![win-server cap20](img/win-server/cap20.png)
*Configuración detallada de la **Password Policy** del dominio `red.local`. Valores aplicados: historial de contraseñas 5, edad máxima 42 días, edad mínima 1 día, longitud mínima 12 caracteres, complejidad habilitada, cifrado reversible deshabilitado. Estas políticas se aplican a todos los usuarios del dominio.*

---

## 7. Windows 10 Pro — Cliente del Dominio

### 7.1 Especificaciones de la VM

![win-client specs cap1](img/specs/win-client/cap1.png)
*Especificaciones de hardware de la VM Windows 10 en Proxmox: CPU y RAM asignados.*

![win-client specs cap2](img/specs/win-client/cap2.png)
*Configuración de almacenamiento de la VM Windows 10 (40 GB).*

![win-client specs cap3](img/specs/win-client/cap3.png)
*Interfaz de red de la VM Windows 10 conectada a la red interna del laboratorio.*

![win-client specs cap4](img/specs/win-client/cap4.png)
*Vista general de la VM Windows 10 en Proxmox.*

![win-client specs cap5](img/specs/win-client/cap5.png)
*Opciones avanzadas de la VM Windows 10.*

![win-client specs cap6](img/specs/win-client/cap6.png)
*Resumen de recursos de la VM Windows 10.*

![win-client specs cap7](img/specs/win-client/cap7.png)
*Configuración de arranque de la VM Windows 10.*

![win-client specs cap8](img/specs/win-client/cap8.png)
*Configuración final de la VM Windows 10 antes del primer arranque.*

---

### 7.2 Instalación del SO

![win-client cap1](img/win-client/cap1.png)
*Pantalla inicial del instalador de Windows 10. Idioma: `Español (España, internacional)`, formato de hora y moneda: `Español (España, internacional)`, teclado: `Español`. A diferencia del servidor, el cliente se instala completamente en español.*

![win-client cap2](img/win-client/cap2.png)
*Selección de la edición del SO. Se elige **Windows 10 Pro** (x64, 05/05/2023). La edición Pro es necesaria para poder unir el equipo a un dominio Active Directory.*

![win-client cap3](img/win-client/cap3.png)
*Selección del disco de instalación. `Espacio sin asignar en la unidad 0` con `40.0 GB` disponibles. Se selecciona para la instalación.*

![win-client cap4](img/win-client/cap4.png)
*Progreso de instalación de Windows 10 al 3%: preparando archivos para la instalación. El proceso continúa con la instalación de características, actualizaciones y finalización.*

![win-client cap5](img/win-client/cap5.png)
*OOBE (Out-of-Box Experience) — selección de región. Se selecciona **España** como región del equipo.*

![win-client cap6](img/win-client/cap6.png)
*OOBE — creación del usuario local. Se introduce el nombre `usuario` como cuenta local inicial del equipo. Esta cuenta se usará hasta completar la unión al dominio.*

![win-client cap7](img/win-client/cap7.png)
*Escritorio de Windows 10 Pro recién instalado. El sistema está listo. Se pueden ver los iconos de Papelera de reciclaje y Microsoft Edge en el escritorio.*

---

### 7.3 Verificación de Red

![win-client cap8](img/win-client/cap8.png)
*Verificación de conectividad de red. `ipconfig` muestra que el cliente recibió correctamente IP por DHCP: `192.168.100.32`, máscara `255.255.255.0` y gateway `192.168.100.1`. El servidor DHCP de ubuntu-server está operativo. El sufijo DNS `example.org` será actualizado al unirse al dominio.*

---

### 7.4 Unión al Dominio

![win-client cap9](img/win-client/cap9.png)
*Proceso de unión al dominio. En **Configuración → Sistema → Acerca de → Cambiar nombre o dominio**, se introduce el equipo como `equipo1` y se selecciona **Dominio**: `red.local`. El cuadro de diálogo de Windows solicita las credenciales de una cuenta con permisos para unir equipos al dominio.*

![win-client cap10](img/win-client/cap10.png)
*Confirmación exitosa de la unión al dominio. El cuadro de diálogo muestra: **"Se unió correctamente al dominio red.local."** El equipo aparece ahora en la OU `Computers` del AD. Es necesario reiniciar para aplicar los cambios.*

![win-client cap11](img/win-client/cap11.png)
*El equipo se reinicia para completar la unión al dominio `red.local`. Pantalla de "Reiniciando". Tras el reinicio, el equipo reconocerá el dominio y permitirá el inicio de sesión con cuentas del directorio activo.*

---

### 7.5 Aplicación de Directivas de Grupo

![win-client cap12](img/win-client/cap12.png)
*Pantalla de login tras el reinicio. La pantalla muestra **"Otro usuario"** con el mensaje "Se debe cambiar la contraseña del usuario antes de iniciar sesión", indicando que el usuario del dominio (`u1`) debe cambiar su contraseña en el primer acceso según la GPO configurada.*

![win-client cap13](img/win-client/cap13.png)
*Inicio de sesión con el usuario del dominio. Se introducen las credenciales del usuario `u1` y la contraseña temporal. El campo inferior muestra **"Iniciar sesión en RED"**, confirmando que el equipo está autenticando contra el controlador de dominio `red.local` (NetBIOS: RED).*

![win-client cap14](img/win-client/cap14.png)
*Ejecución de `gpupdate /force` desde PowerShell como Administrador. El comando fuerza la descarga y aplicación inmediata de todas las directivas de grupo desde el DC. Resultado: **"La actualización de la directiva de equipo se completó correctamente"** y **"Se completó correctamente la actualización de directiva de usuario."***

![win-client cap15](img/win-client/cap15.png)
*Segunda ejecución de `gpupdate /force` confirmando la correcta aplicación de las GPO. Ambas actualizaciones (directiva de equipo y de usuario) se completan correctamente. Las políticas de contraseñas y seguridad definidas en el servidor están activas en el cliente.*

---

## 8. Verificación del Laboratorio

### Checklist de validación final

| Componente | Verificación | Estado |
|---|---|---|
| pfSense | Reglas WAN configuradas y aplicadas | ✓ |
| pfSense | Denegación por defecto activa | ✓ |
| pfSense | Puertos Wazuh (1514, 1515, 55000) abiertos hacia 192.168.100.113 | ✓ |
| ubuntu-server | `isc-dhcp-server` activo en ens19 | ✓ |
| ubuntu-server | `bind9` resolviendo zona `red.local` | ✓ |
| ubuntu-server | Wazuh agent activo y conectado | ✓ |
| wazuh | Contenedor Docker activo (443, 1514, 1515) | ✓ |
| wazuh | WebGUI accesible en `https://192.168.100.113` | ✓ |
| win-server | AD DS instalado, promovido a DC de `red.local` | ✓ |
| win-server | NetBIOS `RED` configurado | ✓ |
| win-server | Usuario `u1@red.local` creado | ✓ |
| win-server | GPO de contraseñas aplicada | ✓ |
| win-server | Wazuh agent `Windows-AD` activo | ✓ |
| win-client | IP asignada por DHCP (rango .10–.50) | ✓ |
| win-client | Unido al dominio `red.local` | ✓ |
| win-client | Login como `u1@RED` exitoso | ✓ |
| win-client | GPO aplicadas con `gpupdate /force` | ✓ |
| Wazuh | Agentes `Ubuntu-server` y `Windows-AD` registrados | ✓ |

### Comandos de verificación rápida

**Ubuntu Server:**
```bash
sudo systemctl status isc-dhcp-server   # DHCP activo
sudo systemctl status bind9             # DNS activo
sudo systemctl status wazuh-agent       # Agente Wazuh activo
ip a                                    # Verificar IPs de ens18 y ens19
```

**Windows Server / Cliente (PowerShell Administrador):**
```powershell
gpupdate /force          # Forzar aplicación de GPO
ipconfig                 # Verificar IP (cliente: rango DHCP .10-.50)
nltest /dsgetdc:red.local  # Verificar conectividad con el DC
```
