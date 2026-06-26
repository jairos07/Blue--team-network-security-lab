# Documentación Técnica — Blue Team Network Security Lab

<img src="img/portada.webp">

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

El presente documento recoge la configuración técnica detallada de cada componente del laboratorio Blue Team Network Security Lab. El objetivo principal es documentar, paso a paso y con soporte visual mediante capturas de pantalla, el proceso de despliegue de una infraestructura de red empresarial simulada orientada a la seguridad defensiva.

El laboratorio integra los siguientes elementos:

- **pfSense** como firewall perimetral con política de denegación por defecto y registro de tráfico.
- **Ubuntu Server** como servidor de servicios de red centrales: DHCP y DNS.
- **Wazuh** como plataforma SIEM para la recolección, correlación y análisis de eventos de seguridad.
- **Windows Server 2022** como controlador de dominio Active Directory con gestión de usuarios y políticas de grupo.
- **Windows 10 Pro** como estación de trabajo cliente unida al dominio.

Toda la infraestructura se despliega sobre el hipervisor **Proxmox VE**, con las máquinas virtuales interconectadas a través de la red interna `192.168.100.0/24` bajo el dominio `red.local`.

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

*La captura muestra la configuración de CPU y memoria asignada a la máquina virtual de pfSense dentro del panel de administración de Proxmox VE. Se pueden observar el número de núcleos virtuales y la cantidad de RAM reservada para garantizar un rendimiento estable del firewall.*

---

![pfSense specs cap2](img/specs/pfsense/cap2.png)

*Se muestra la configuración del almacenamiento asignado a la VM de pfSense. En el panel de hardware de Proxmox se aprecia el disco virtual con su tamaño y el tipo de controlador utilizado, parámetros que determinan la capacidad de almacenamiento para los logs y la configuración del sistema.*

---

![pfSense specs cap3](img/specs/pfsense/cap3.png)

*Vista de la configuración de interfaces de red de la VM en Proxmox. Se observan los dos adaptadores de red virtuales asignados: uno destinado a la interfaz WAN y otro a la interfaz LAN, cada uno asociado a su bridge de red correspondiente dentro del hipervisor.*

---

![pfSense specs cap4](img/specs/pfsense/cap4.png)

*Panel de resumen general de la VM pfSense en Proxmox VE. Se visualiza el estado de la máquina virtual, junto con un listado consolidado de los recursos de hardware asignados: CPU, memoria, disco y red.*

---

![pfSense specs cap5](img/specs/pfsense/cap5.png)

*Detalle de las opciones de arranque y configuración avanzada de hardware de la VM. Se aprecia la configuración del firmware de arranque (BIOS/UEFI), el orden de dispositivos de arranque y otras opciones específicas del hardware virtualizado.*

---

![pfSense specs cap6](img/specs/pfsense/cap6.png)

*Resumen de la totalidad de recursos asignados a la VM pfSense. La captura muestra de forma consolidada el inventario de hardware virtual: procesadores, memoria RAM, almacenamiento e interfaces de red configuradas.*

---

![pfSense specs cap7](img/specs/pfsense/cap7.png)

*Vista final de la configuración de la VM pfSense previa al primer arranque. El panel de Proxmox confirma que todos los parámetros de hardware están correctamente definidos y que la máquina virtual está lista para ser iniciada.*

---

### 3.2 Configuración inicial (Setup Wizard)

![pfSense cap1](img/pfsense/cap1.png)

*Pantalla de bienvenida de pfSense durante el primer arranque desde la consola del hipervisor. Se muestra el menú principal de gestión de pfSense por línea de comandos, donde se presentan las opciones disponibles para la configuración inicial del sistema, entre ellas la asignación de interfaces de red.*

---

![pfSense cap2](img/pfsense/cap2.png)

*Proceso de selección y asignación de interfaces de red WAN y LAN a través de la consola de pfSense. El sistema detecta los adaptadores de red disponibles y solicita al administrador que especifique cuál corresponde a cada interfaz, paso fundamental para el correcto enrutamiento del tráfico.*

---

![pfSense cap3](img/pfsense/cap3.png)

*Confirmación de la asignación de interfaces completada. La consola muestra el resultado del proceso de detección y asignación, indicando qué adaptador físico ha quedado vinculado a cada interfaz lógica (WAN y LAN) del firewall.*

---

![pfSense cap4](img/pfsense/cap4.png)

*Configuración de la dirección IP estática de la interfaz LAN desde la consola de pfSense, paso previo al acceso al panel de administración web. Se introduce la dirección IP que actuará como gateway de la red interna del laboratorio, permitiendo así el acceso al WebGUI desde un navegador.*

---

![pfSense cap5](img/pfsense/cap5.png)

*Confirmación de la configuración de la interfaz LAN. La consola muestra la dirección IP asignada y la URL de acceso al panel de administración web de pfSense, indicando que el sistema está listo para ser gestionado a través del navegador.*

---

![pfSense cap6](img/pfsense/cap6.png)

*Página de autenticación del panel de administración web (WebGUI) de pfSense accedida desde el navegador. Se muestra el formulario de inicio de sesión donde se introducen las credenciales de administrador para acceder al entorno gráfico de configuración del firewall.*

---

![pfSense cap7](img/pfsense/cap7.png)

*Dashboard principal de pfSense tras el inicio de sesión exitoso. El panel de control muestra el estado general del sistema: versión de pfSense CE instalada, estado de las interfaces WAN y LAN, tiempo de actividad del sistema y un resumen de los recursos del servidor.*

---

![pfSense cap8](img/pfsense/cap8.png)

*Primer paso del asistente de configuración inicial (Setup Wizard) de pfSense. La pantalla de bienvenida del wizard informa al administrador sobre el proceso guiado de 9 pasos que permite establecer la configuración básica del firewall antes de su puesta en producción.*

---

![pfSense cap9](img/pfsense/cap9.png)

*Setup Wizard — Paso 2 de 9: Información General del sistema. Se configura el nombre de host del firewall como `pfSense`, el dominio del laboratorio como `red.local` y el servidor DNS primario como `192.168.100.1` (ubuntu-server). Adicionalmente, se activa la opción de sobrescritura de DNS para permitir que el servidor DHCP o las conexiones PPP actualicen dinámicamente la configuración DNS del sistema.*

---

![pfSense cap10](img/pfsense/cap10.png)

*Setup Wizard — Paso 4 de 9: Configuración de la interfaz WAN. Se establece el tipo de conexión como estático, asignando la dirección IP `192.168.100.2` con máscara `/24` y el gateway `192.168.100.1`. Los campos de dirección MAC personalizada, MTU y MSS se dejan en blanco para utilizar los valores por defecto del sistema.*

---

![pfSense cap11](img/pfsense/cap11.png)

*Setup Wizard — Paso 5 de 9: Configuración de la interfaz LAN. Se asigna la dirección IP `192.168.100.1` con prefijo `/24`, que funcionará como puerta de enlace predeterminada para todos los dispositivos de la red interna del laboratorio.*

---

![pfSense cap12](img/pfsense/cap12.png)

*Setup Wizard — Paso 6 de 9: Establecimiento de la contraseña de administrador del WebGUI. La interfaz solicita la nueva contraseña para la cuenta `admin`, que también será utilizada para el acceso SSH al sistema en caso de que dicho servicio esté habilitado. Se requiere confirmación de la contraseña para evitar errores de escritura.*

---

### 3.3 Reglas de Firewall

Las reglas se aplican sobre la interfaz **WAN** bajo una política base de **denegación implícita**. Únicamente se definen reglas de tipo `Pass` para el tráfico estrictamente necesario. El registro de eventos (logging) se habilita en todas las reglas para permitir su posterior correlación en el SIEM Wazuh.

---

![pfSense cap13](img/pfsense/cap13.png)

*Editor de reglas de firewall para el servicio **DNS** (puerto 53, TCP/UDP). La regla se configura con acción `Pass` sobre la interfaz WAN. El origen queda definido como la red `192.168.100.0/24` y el destino como `192.168.100.1`, correspondiente al servidor ubuntu-server que actúa como servidor DNS del laboratorio. Esta regla permite que todos los dispositivos de la red interna realicen consultas de resolución de nombres.*

---

![pfSense cap14](img/pfsense/cap14.png)

*Opciones avanzadas de la regla de DNS. Se muestra la sección inferior del formulario de edición, donde se aprecia la casilla de logging activada (`Log packets that are handled by this rule`) junto al campo de descripción con el valor `DNS`. La habilitación del registro permite auditar las consultas DNS a través del panel de logs de pfSense y correlacionar los eventos en Wazuh.*

---

![pfSense cap15](img/pfsense/cap15.png)

*Configuración de la regla para el servicio **DHCP** (puerto 67, TCP/UDP). La regla establece acción `Pass` sobre la interfaz WAN, con origen en la subred `192.168.100.0/24` y destino `192.168.100.1/24` (ubuntu-server). El logging está habilitado. Esta regla garantiza que los clientes de la red interna puedan enviar solicitudes de asignación de dirección IP al servidor DHCP.*

---

![pfSense cap16](img/pfsense/cap16.png)

*Configuración de la regla para el acceso **SSH** (puerto 22, TCP). La regla define acción `Pass` sobre la interfaz WAN, con origen en `192.168.100.0/24` y destino `Any`. El logging está activado. Esta regla permite el acceso por SSH a cualquier destino desde cualquier equipo de la red interna, facilitando la administración remota de los servidores del laboratorio.*

---

![pfSense cap17](img/pfsense/cap17.png)

*Regla de **denegación por defecto**. Se configura con acción `Block` sobre la interfaz WAN, protocolo TCP, con origen `Any` y destino `Any`. La descripción indica explícitamente su función: `Denegacion por defecto, bloquear todo el otro trafico`. El logging está habilitado para registrar todos los intentos de conexión bloqueados. Esta regla implementa el principio de mínimo privilegio, rechazando silenciosamente cualquier tráfico no autorizado por las reglas anteriores.*

---

![pfSense cap18](img/pfsense/cap18.png)

*Vista completa del conjunto de **reglas aplicadas sobre la interfaz WAN**. De arriba hacia abajo se pueden identificar las siguientes reglas activas: bloqueo de redes bogon, bloqueo por defecto de todo el tráfico, permiso de SSH (puerto 22) desde `192.168.100.0/24`, permiso de DHCP (puerto 67) hacia `192.168.100.1/24` y permiso de DNS (puerto 53) hacia `192.168.100.1`. En la parte superior de la pantalla se aprecia el aviso de cambios pendientes de aplicar, con el botón `Apply Changes` disponible para confirmar la política.*

---

![pfSense cap19](img/pfsense/cap19.png)

*Regla para el puerto **1514 de Wazuh** (canal de comunicación de eventos del agente). La regla establece acción `Pass`, protocolo TCP, con origen `192.168.100.0/24` y destino `192.168.100.113` (servidor Wazuh) en el puerto 1514. El logging está habilitado. Este puerto es el canal principal por el que los agentes Wazuh instalados en los equipos del laboratorio envían sus eventos de seguridad al manager.*

---

![pfSense cap20](img/pfsense/cap20.png)

*Regla para el puerto **1515 de Wazuh** (registro y autenticación de agentes). Misma configuración que la regla anterior en cuanto a origen y destino, pero apuntando al puerto 1515. Logging habilitado. Este puerto es utilizado durante el proceso de alta y registro de un nuevo agente contra el manager de Wazuh, intercambiando las claves de autenticación necesarias para el establecimiento del canal seguro.*

---

![pfSense cap21](img/pfsense/cap21.png)

*Regla para el puerto **55000 de Wazuh** (API REST). Acción `Pass`, protocolo TCP, origen `192.168.100.0/24`, destino `192.168.100.113`, puerto 55000. Logging habilitado. Este puerto expone la API RESTful de Wazuh, necesaria para la gestión programática del manager, la consulta de eventos y el control de agentes a través de herramientas de automatización o desde la propia interfaz web.*

---

## 4. Ubuntu Server — DHCP y DNS

### 4.1 Especificaciones de la VM

![ubuntu-server specs cap1](img/specs/ubuntu-server/cap1.png)

*Configuración de CPU y memoria RAM asignada a la VM ubuntu-server en el panel de hardware de Proxmox VE. Los recursos mostrados determinan la capacidad de procesamiento disponible para los servicios DHCP, DNS y el agente Wazuh que correrán simultáneamente en esta máquina.*

---

![ubuntu-server specs cap2](img/specs/ubuntu-server/cap2.png)

*Vista de la configuración de almacenamiento de la VM ubuntu-server. El panel muestra el disco virtual asignado, indicando su capacidad total y el tipo de controlador de almacenamiento utilizado, parámetros que condicionan el rendimiento de las operaciones de lectura y escritura del sistema.*

---

![ubuntu-server specs cap3](img/specs/ubuntu-server/cap3.png)

*Configuración de la primera interfaz de red de la VM (ens18) en Proxmox. Se observa el bridge de red al que está conectada, que corresponde a la interfaz de gestión del hipervisor, y el modelo de adaptador de red virtualizado seleccionado.*

---

![ubuntu-server specs cap4](img/specs/ubuntu-server/cap4.png)

*Configuración de la segunda interfaz de red de la VM (ens19) en Proxmox. Esta segunda tarjeta está conectada al bridge de la red interna del laboratorio (vmbr1), siendo la interfaz a través de la cual ubuntu-server prestará los servicios DHCP y DNS al resto de máquinas del entorno.*

---

![ubuntu-server specs cap5](img/specs/ubuntu-server/cap5.png)

*Panel de resumen general de la VM ubuntu-server en Proxmox VE. Se visualiza el estado actual de la máquina y un listado consolidado de todos los componentes de hardware virtual asignados.*

---

![ubuntu-server specs cap6](img/specs/ubuntu-server/cap6.png)

*Resumen completo de los recursos hardware de ubuntu-server. La vista muestra de forma unificada la configuración de CPU, memoria, almacenamiento e interfaces de red que conforman la especificación final de la máquina virtual.*

---

![ubuntu-server specs cap7](img/specs/ubuntu-server/cap7.png)

*Estado de la VM ubuntu-server previo al primer arranque. La configuración de hardware queda completamente definida, confirmando que la máquina virtual está preparada para iniciar el proceso de instalación del sistema operativo.*

---

### 4.2 Instalación del SO

![ubuntu-server cap1](img/ubuntu-server/cap1.png)

*Pantalla inicial del instalador de Ubuntu Server 24.04 LTS mostrando la selección de idioma del sistema. Se ha seleccionado **Español** como idioma de instalación, configuración que afecta tanto al propio proceso de instalación como a los mensajes del sistema operativo una vez desplegado.*

---

![ubuntu-server cap2](img/ubuntu-server/cap2.png)

*Paso de configuración del perfil de usuario durante la instalación. Se establece el nombre completo del usuario como `administrador`, el nombre de host de la máquina como `ubuntu-server`, el nombre de usuario para el inicio de sesión como `administrador` y se define la contraseña de acceso. Este usuario dispondrá de privilegios sudo para la administración del sistema.*

---

![ubuntu-server cap3](img/ubuntu-server/cap3.png)

*Configuración del servicio SSH durante el proceso de instalación. El instalador ofrece la opción de instalar el servidor OpenSSH directamente en este paso. Se selecciona dicha opción y se habilita la autenticación por contraseña, lo que permitirá el acceso remoto al servidor una vez completada la instalación sin necesidad de configuración adicional.*

---

### 4.3 Configuración de Red

![ubuntu-server cap4](img/ubuntu-server/cap4.png)

*Diálogo de Proxmox VE para agregar un segundo dispositivo de red a la VM ubuntu-server. Se selecciona el bridge `vmbr1`, que corresponde a la red interna del laboratorio, se elige el modelo de adaptador VirtIO paravirtualizado para un mayor rendimiento y se activa el cortafuegos de Proxmox sobre esta interfaz como medida de seguridad adicional a nivel del hipervisor.*

---

![ubuntu-server cap5](img/ubuntu-server/cap5.png)

*Salida del comando `ip a` que refleja el estado inicial de las interfaces de red del servidor recién instalado. La interfaz `ens18` aparece en estado UP con la dirección `192.168.1.226/24` asignada dinámicamente por el DHCP del host Proxmox. La interfaz `ens19`, correspondiente a la red interna del laboratorio, aparece en estado DOWN al no tener aún configuración estática asignada.*

---

![ubuntu-server cap6](img/ubuntu-server/cap6.png)

*Edición del archivo de configuración de red `/etc/netplan/50-cloud-init.yaml` mediante el editor `nano`. El archivo muestra la configuración estática definida para ambas interfaces: `ens18` con la dirección `192.168.1.200/24`, gateway `192.168.1.1` y servidores DNS `8.8.8.8` y `8.8.4.4` para el acceso de gestión; `ens19` con la dirección `192.168.100.1/24` y sin gateway, al tratarse de la interfaz de la red interna aislada del laboratorio.*

---

![ubuntu-server cap7](img/ubuntu-server/cap7.png)

*Aplicación de la configuración Netplan mediante `sudo netplan apply` y verificación del resultado con `ip a`. Ambas interfaces muestran ahora estado UP con sus respectivas direcciones IP correctamente asignadas: `ens18` con `192.168.1.200/24` y `ens19` con `192.168.100.1/24`. La red interna del laboratorio queda operativa a partir de este punto.*

---

### 4.4 Servidor DHCP (isc-dhcp-server)

![ubuntu-server cap8](img/ubuntu-server/cap8.png)

*Instalación del paquete `isc-dhcp-server` mediante el gestor de paquetes `apt`. La terminal muestra el proceso de descarga e instalación del daemon DHCP junto con su dependencia `isc-dhcp-common`. El gestor de paquetes indica el tamaño de los archivos a descargar y el espacio en disco que ocupará la instalación una vez completada.*

---

![ubuntu-server cap9](img/ubuntu-server/cap9.png)

*Edición del archivo de configuración `/etc/default/isc-dhcp-server` con `nano`. En este archivo se define la directiva `INTERFACESv4="ens19"`, instruyendo al daemon DHCP para que escuche solicitudes únicamente en la interfaz de la red interna del laboratorio. El campo `INTERFACESv6` se deja vacío al no requerir soporte para IPv6 en este entorno.*

---

![ubuntu-server cap10](img/ubuntu-server/cap10.png)

*Edición del archivo de configuración principal `/etc/dhcp/dhcpd.conf`. Se define el ámbito DHCP para la subred `192.168.100.0` con máscara `255.255.255.0`, especificando el rango de direcciones a distribuir entre `192.168.100.10` y `192.168.100.50`. Se configuran también el gateway por defecto (`192.168.100.1`), los servidores DNS (`8.8.8.8` y `8.8.4.4`), el tiempo de concesión por defecto (602 segundos) y el máximo (7200 segundos). La directiva `authoritative` indica que este servidor es el DHCP autoritativo de la red.*

---

![ubuntu-server cap11](img/ubuntu-server/cap11.png)

*Reinicio, habilitación en el arranque y verificación del estado del servicio DHCP. La salida de `systemctl status isc-dhcp-server` confirma que el servicio se encuentra en estado **active (running)**. Se puede observar también la fecha de inicio del servicio, el proceso daemon `dhcpd` escuchando en la interfaz `ens19` sobre la subred `192.168.100.0/24` y la ruta a la base de datos de concesiones activas `/var/lib/dhcp/dhcpd.leases`.*

---

### 4.5 Servidor DNS (BIND9)

![ubuntu-server cap12](img/ubuntu-server/cap12.png)

*Instalación del servidor DNS BIND9 y sus utilidades de administración mediante `apt`. La terminal registra la descarga e instalación de los paquetes `bind9`, `bind9-doc`, `bind9utils` y `dns-root-data`, indicando el volumen total de datos descargados y el espacio en disco requerido para la instalación completa.*

---

![ubuntu-server cap13](img/ubuntu-server/cap13.png)

*Edición del archivo `/etc/bind/named.conf.local` con `nano`. Se declaran las dos zonas DNS que gestionará este servidor: la zona de resolución directa `red.local`, de tipo `master`, cuya definición de registros reside en el archivo `/etc/bind/db.red.local`; y la zona de resolución inversa `100.168.192.in-addr.arpa`, también de tipo `master`, con sus registros en `/etc/bind/db.192.168.100`. Esta configuración convierte a ubuntu-server en el servidor DNS autoritativo del dominio del laboratorio.*

---

![ubuntu-server cap14](img/ubuntu-server/cap14.png)

*Creación del archivo de zona de resolución directa copiando la plantilla de zona local por defecto de BIND9: `sudo cp /etc/bind/db.local /etc/bind/db.red.local`. Esta acción proporciona una base de configuración válida que será modificada en el siguiente paso para definir los registros de recursos específicos del dominio `red.local`.*

---

![ubuntu-server cap15](img/ubuntu-server/cap15.png)

*Edición del archivo de zona directa `/etc/bind/db.red.local`. El archivo muestra la configuración del registro SOA (Start of Authority) con `ns.red.local.` como servidor de nombres primario y `admin.red.local.` como dirección de contacto del administrador. El número de serie se establece en `2`. Se incluyen un registro `NS` apuntando a `localhost`, un registro `A` que resuelve `ns` a `192.168.100.1` y un registro `A` que resuelve `server` a `192.168.100.1`.*

---

![ubuntu-server cap16](img/ubuntu-server/cap16.png)

*Creación del archivo de zona de resolución inversa copiando la plantilla loopback de BIND9: `sudo cp /etc/bind/db.127 /etc/bind/db.192.168.100`. Al igual que en el caso anterior, este archivo servirá como base para la configuración de los registros PTR necesarios para la resolución inversa de direcciones de la subred `192.168.100.0/24`.*

---

![ubuntu-server cap17](img/ubuntu-server/cap17.png)

*Edición del archivo de zona inversa `/etc/bind/db.192.168.100`. Se configura el registro SOA con los mismos valores de nameserver y contacto que la zona directa. Se añade un registro `NS` apuntando a `ns.red.local.` y un registro `PTR` que asocia el último octeto `1` con el nombre completo `ns.red.local.`, permitiendo la resolución inversa de la dirección `192.168.100.1`.*

---

![ubuntu-server cap18](img/ubuntu-server/cap18.png)

*Arranque del servicio BIND9 y verificación de su estado con `systemctl status bind9`. El servicio aparece en estado **active (running)**, indicando que el daemon `named` se ha iniciado correctamente. Los mensajes del log muestran la carga satisfactoria de la zona `red.local` y el envío de notificaciones. Los mensajes de `network unreachable` para direcciones IPv6 son esperados y no afectan al funcionamiento del servicio en un entorno sin conectividad IPv6 configurada.*

---

## 5. Wazuh — SIEM

### 5.1 Especificaciones de la VM

![wazuh specs cap1](img/specs/wazuh/cap1.png)

*Configuración de CPU y memoria de la VM wazuh en Proxmox VE. Dado que Wazuh se desplegará mediante contenedores Docker y debe procesar eventos en tiempo real de todos los agentes, esta VM requiere una asignación de recursos considerablemente mayor que el resto de máquinas del laboratorio.*

---

![wazuh specs cap2](img/specs/wazuh/cap2.png)

*Configuración de almacenamiento de la VM wazuh. El disco virtual muestra la capacidad asignada, que debe ser suficiente para albergar las imágenes Docker del stack de Wazuh (manager, indexer y dashboard) así como los índices de eventos generados por los agentes a lo largo del tiempo.*

---

![wazuh specs cap3](img/specs/wazuh/cap3.png)

*Configuración de la interfaz de red de la VM wazuh. Se muestra el bridge al que está conectada la tarjeta de red virtual, correspondiente a la red interna del laboratorio, a través de la cual los agentes enviarán sus eventos y los administradores accederán al panel web.*

---

![wazuh specs cap4](img/specs/wazuh/cap4.png)

*Panel de resumen general de la VM wazuh en el panel de administración de Proxmox VE. Se visualiza el estado de la máquina y el conjunto de recursos hardware asignados de forma unificada.*

---

![wazuh specs cap5](img/specs/wazuh/cap5.png)

*Vista de resumen de recursos de la VM wazuh, mostrando de forma consolidada la especificación completa de hardware virtual: CPU, memoria, almacenamiento e interfaces de red.*

---

![wazuh specs cap6](img/specs/wazuh/cap6.png)

*Opciones de arranque y parámetros adicionales de configuración de la VM wazuh en Proxmox. Se aprecian los ajustes de firmware, orden de dispositivos de arranque y otras opciones avanzadas de virtualización.*

---

![wazuh specs cap7](img/specs/wazuh/cap7.png)

*Configuración final de la VM wazuh antes del primer arranque. El panel de hardware confirma que todos los parámetros están correctamente definidos y que la máquina está lista para iniciar el proceso de instalación del sistema operativo.*

---

### 5.2 Instalación del SO

![wazuh cap1](img/wazuh/cap1.png)

*Pantalla de selección de idioma del instalador de Ubuntu Server 24.04 LTS para la VM wazuh. Se selecciona **Español** como idioma del sistema, manteniendo coherencia con la configuración del resto de servidores Linux del laboratorio.*

---

![wazuh cap2](img/wazuh/cap2.png)

*Configuración del perfil de usuario de la VM wazuh durante la instalación. Se establece el nombre de host como `wazuh` para identificar claramente esta máquina en la red, el nombre de usuario como `administrador` y se define la contraseña de acceso. El usuario dispondrá de privilegios sudo para la gestión del sistema.*

---

![wazuh cap3](img/wazuh/cap3.png)

*Configuración del acceso SSH durante la instalación de la VM wazuh. Se selecciona la instalación del servidor OpenSSH y se habilita la autenticación por contraseña, permitiendo el acceso remoto a la máquina desde el equipo de administración una vez completada la instalación.*

---

### 5.3 Despliegue con Docker Compose

![wazuh cap4](img/wazuh/cap4.png)

*Instalación del paquete `docker-compose` mediante `apt install docker-compose`. La terminal muestra el proceso de descarga e instalación de Docker Compose junto con sus dependencias Python (`python3-docker`, `python3-dockerpty` y otras librerías relacionadas). Este componente es la herramienta que orquestará el despliegue del stack de Wazuh mediante la definición declarativa de sus servicios y configuraciones.*

---

![wazuh cap5](img/wazuh/cap5.png)

*Contenido del archivo `docker-compose.yml` que define el despliegue del stack de Wazuh. El archivo especifica el servicio `wazuh` utilizando la imagen `ghcr.io/wazuh/wazuh:latest` con nombre de contenedor `wazuh`. Se mapean los puertos necesarios para su operación: `443:443` para el acceso a la interfaz web mediante HTTPS, `1514:1514/udp` para la recepción de eventos de los agentes y `1515:1515` para el proceso de registro de nuevos agentes. Las variables de entorno `INDEXER_USERNAME` e `INDEXER_PASSWORD` configuran las credenciales de acceso al indexador. La política de reinicio `always` garantiza que el contenedor se recupere automáticamente ante cualquier fallo o reinicio del sistema.*

---

![wazuh cap6](img/wazuh/cap6.png)

*Ejecución del comando `docker-compose up -d` para iniciar el despliegue del contenedor Wazuh en segundo plano. La terminal muestra cómo Docker crea la red virtual `administrador_default` para el stack y comienza la descarga de la imagen `ghcr.io/wazuh/wazuh:latest` desde el registro de contenedores de GitHub.*

---

![wazuh cap7](img/wazuh/cap7.png)

*Interfaz de autenticación del panel web de Wazuh, accesible en la URL `https://192.168.100.113`. La página de login muestra el logotipo de Wazuh con su lema "The Open Source Security Platform". Se introducen las credenciales del usuario administrador para acceder al dashboard de gestión del SIEM.*

---

### 5.4 Registro de Agentes

![wazuh cap8](img/wazuh/cap8.png)

*Asistente **Deploy new agent** de la interfaz web de Wazuh. El formulario permite seleccionar la plataforma de destino del agente (Linux, Windows, macOS o Solaris), la arquitectura del sistema, la dirección IP del manager al que reportará el agente y el nombre identificativo que tendrá en el panel. La captura muestra el formulario en su estado inicial, antes de introducir los datos del primer agente.*

---

![wazuh cap9](img/wazuh/cap9.png)

*Configuración del asistente para el despliegue del agente en **ubuntu-server**. Se ha seleccionado la plataforma Linux RPM amd64, se ha especificado `192.168.100.1` como dirección del manager de Wazuh y se ha asignado el nombre `Ubuntu-server` al agente. Con estos parámetros, Wazuh genera automáticamente el comando de instalación personalizado en el siguiente paso del asistente.*

---

![wazuh cap10](img/wazuh/cap10.png)

*Paso 4 del asistente de despliegue: comando de instalación generado para ubuntu-server. El comando utiliza `curl` para descargar el paquete de Wazuh Agent versión 4.14.5 e instalarlo pasando las variables de entorno `WAZUH_MANAGER='192.168.100.1'` y `WAZUH_AGENT_NAME='Ubuntu-server'`. El paso 5 del asistente muestra los comandos para habilitar el servicio en el arranque del sistema e iniciarlo mediante `systemctl`.*

---

![wazuh cap12](img/wazuh/cap12.png)

*Proceso de instalación del agente Wazuh en **ubuntu-server** mediante el paquete DEB correspondiente a la distribución Ubuntu. Se descarga el paquete `wazuh-agent_4.14.5-1_amd64.deb` con `wget` y se procede a su instalación con `dpkg`, pasando las variables de entorno `WAZUH_MANAGER` y `WAZUH_AGENT_NAME='ubuntu-server'` que configuran automáticamente el agente con los parámetros del manager durante la instalación.*

---

![wazuh cap13](img/wazuh/cap13.png)

*Activación y verificación del agente Wazuh en ubuntu-server. Se ejecutan secuencialmente `systemctl daemon-reload`, `systemctl enable wazuh-agent` y `systemctl start wazuh-agent`. La salida de `systemctl status wazuh-agent` confirma el estado **active (running)** del servicio. Los logs muestran que todos los subprocesos del agente están operativos: `wazuh-execd` (ejecución de respuestas activas), `wazuh-agentd` (comunicación con el manager), `wazuh-syscheckd` (monitorización de integridad de archivos), `wazuh-logcollector` (recolección de logs) y `wazuh-modulesd` (módulos adicionales).*

---

![wazuh cap15](img/wazuh/cap15.png)

*Configuración del asistente de despliegue para el agente del **Windows Server (AD)**. Se ha seleccionado la plataforma Windows MSI de 32/64 bits, se ha especificado `192.168.100.3` como dirección del manager y se ha asignado el nombre `Windows-AD` al agente. Wazuh genera el comando de instalación en formato PowerShell para su ejecución en el Windows Server.*

---

![wazuh cap16](img/wazuh/cap16.png)

*Instalación del agente Wazuh en **Windows Server** desde una consola de PowerShell con privilegios de Administrador. El proceso se realiza en dos pasos: primero se descarga el instalador MSI mediante `Invoke-WebRequest` y a continuación se ejecuta con los parámetros `/q WAZUH_MANAGER='192.168.100.3'` y `WAZUH_AGENT_NAME='Windows-AD'` para una instalación silenciosa con la configuración del manager preestablecida. La ejecución del comando `NET START Wazuh` confirma que el servicio del agente se ha iniciado correctamente con el mensaje `The Wazuh service was started successfully.`*

---

## 6. Windows Server 2022 — Active Directory

### 6.1 Especificaciones de la VM

![win-server specs cap1](img/specs/win-server/cap1.png)

*Configuración de CPU y memoria de la VM Windows Server 2022 en Proxmox VE. Los recursos asignados deben ser suficientes para soportar el sistema operativo Windows Server junto con los servicios de Active Directory Domain Services, DNS integrado y el agente Wazuh.*

---

![win-server specs cap2](img/specs/win-server/cap2.png)

*Configuración del almacenamiento de la VM Windows Server. Se muestra el disco virtual de 60 GB asignado a esta máquina, espacio necesario para albergar el sistema operativo Windows Server 2022, la base de datos del Active Directory (NTDS) y los archivos SYSVOL.*

---

![win-server specs cap3](img/specs/win-server/cap3.png)

*Configuración de la interfaz de red de la VM Windows Server en Proxmox. La tarjeta de red virtual está conectada al bridge de la red interna del laboratorio, a través del cual el controlador de dominio atenderá las solicitudes de autenticación y resolución de nombres de los equipos del dominio.*

---

![win-server specs cap4](img/specs/win-server/cap4.png)

*Panel de resumen general de la VM Windows Server 2022 en la consola de administración de Proxmox VE, mostrando el estado de la máquina y un inventario de su hardware virtual.*

---

![win-server specs cap5](img/specs/win-server/cap5.png)

*Opciones avanzadas de configuración de la VM Windows Server en Proxmox. Se muestran parámetros específicos como el tipo de máquina virtual, opciones del agente QEMU y otros ajustes de virtualización relevantes para el correcto funcionamiento de Windows Server.*

---

![win-server specs cap6](img/specs/win-server/cap6.png)

*Resumen consolidado de los recursos hardware asignados a la VM Windows Server, incluyendo la especificación completa de CPU, memoria, almacenamiento e interfaces de red.*

---

![win-server specs cap7](img/specs/win-server/cap7.png)

*Configuración de las opciones de arranque de la VM Windows Server en Proxmox. Se visualiza el orden de dispositivos de arranque y la configuración de firmware, parámetros necesarios para el correcto arranque del sistema operativo Windows.*

---

![win-server specs cap8](img/specs/win-server/cap8.png)

*Vista final de la configuración de hardware de la VM Windows Server previa al primer arranque. Todos los componentes virtuales están correctamente configurados y la máquina está lista para iniciar la instalación del sistema operativo.*

---

### 6.2 Instalación del SO

![win-server cap1](img/win-server/cap1.png)

*Pantalla inicial del instalador de Windows Server 2022. Se muestra la selección de configuración regional: el idioma del sistema se mantiene como `English (United States)` para mayor compatibilidad con las herramientas de administración, mientras que el formato de fecha y moneda se establece en `Spanish (Spain, International Sort)` y la distribución de teclado en `Spanish`. Esta combinación permite trabajar con la interfaz en inglés utilizando el teclado en español.*

---

![win-server cap2](img/win-server/cap2.png)

*Selección de la edición del sistema operativo a instalar. Se elige **Windows Server 2022 Standard Evaluation (Desktop Experience)** en arquitectura x64, con fecha de compilación 3/3/2022. La variante con Desktop Experience proporciona la interfaz gráfica completa de Windows, necesaria para la configuración interactiva de los servicios de Active Directory mediante sus asistentes gráficos.*

---

![win-server cap3](img/win-server/cap3.png)

*Selección del disco de destino para la instalación. El instalador detecta la unidad `Drive 0` con 60,0 GB de espacio sin particionar, que se selecciona como destino. El instalador creará automáticamente las particiones necesarias para el sistema: partición de sistema EFI, partición reservada para Windows y partición principal donde se instalará el SO.*

---

![win-server cap4](img/win-server/cap4.png)

*Progreso del proceso de instalación de Windows Server 2022, avanzando al 55%. En esta fase el instalador ha completado la copia de archivos y se encuentra expandiendo e instalando las características y actualizaciones del sistema operativo, pasos previos a la fase de finalización y primer arranque.*

---

![win-server cap5](img/win-server/cap5.png)

*Pantalla de personalización post-instalación para la configuración de la cuenta de administrador. El instalador solicita el establecimiento de la contraseña para la cuenta integrada `Administrator` del sistema, que será el método de acceso inicial al servidor antes de configurar cuentas de dominio.*

---

### 6.3 Configuración de Red

![win-server cap6](img/win-server/cap6.png)

*Configuración de la dirección IP estática del servidor mediante la interfaz gráfica de Windows. Accediendo a través de **Configuración → Red e Internet → Ethernet → Propiedades del adaptador → Protocolo de Internet versión 4 (TCP/IPv4)**, se establecen los siguientes valores: dirección IP `192.168.100.3`, máscara de subred `255.255.255.0`, puerta de enlace predeterminada `192.168.100.1` y servidor DNS preferido `192.168.100.1` (ubuntu-server). La dirección `192.168.100.3` será la IP permanente del Controlador de Dominio a la que los clientes del dominio se conectarán para autenticación y resolución de nombres.*

---

### 6.4 Instalación de AD DS

![win-server cap7](img/win-server/cap7.png)

*Asistente **Agregar roles y características** de Server Manager, en la fase de selección de roles del servidor. La lista muestra todos los roles disponibles en Windows Server 2022. En este paso se seleccionará **Active Directory Domain Services (AD DS)** para habilitar las capacidades de controlador de dominio en este servidor, junto con las herramientas de administración asociadas.*

---

![win-server cap8](img/win-server/cap8.png)

*Página de confirmación de la instalación del rol AD DS. Se muestra el listado completo de componentes que serán instalados: `Active Directory Domain Services`, `Group Policy Management`, `Remote Server Administration Tools`, `AD DS and AD LDS Tools`, `Active Directory Administrative Center` y `AD DS Snap-Ins and Command-Line Tools`. La opción de reinicio automático en caso necesario aparece marcada para agilizar el proceso de instalación.*

---

![win-server cap9](img/win-server/cap9.png)

*Dashboard de Server Manager tras completar satisfactoriamente la instalación del rol AD DS. Se visualiza el aviso de acción requerida en el panel de notificaciones: *"Configuration required for Active Directory Domain Services at WIN-HOGMAS1NJHI"*, indicando que el rol ha sido instalado pero aún es necesario realizar la configuración de post-despliegue para promover el servidor a controlador de dominio mediante el enlace **Promote this server to a domain controller**.*

---

### 6.5 Promoción a Controlador de Dominio

![win-server cap10](img/win-server/cap10.png)

*Asistente de Configuración de AD DS — fase de Configuración de Implementación. Se selecciona la opción **Agregar un nuevo bosque** para crear una nueva infraestructura de Active Directory desde cero. Se especifica `red.local` como nombre del dominio raíz del bosque. El servidor de destino sobre el que se realizará la promoción aparece identificado como `WIN-HOGMAS1NJHI`.*

---

![win-server cap11](img/win-server/cap11.png)

*Opciones del Controlador de Dominio. Se establecen los niveles funcionales del bosque y del dominio en **Windows Server 2016**, que ofrece compatibilidad con versiones anteriores manteniendo las características más modernas. Se activan las capacidades de **Servidor DNS** y **Catálogo Global (GC)** en este controlador. No se configura como RODC (Controlador de Dominio de Solo Lectura). Se define también la contraseña del Modo de Restauración de Servicios de Directorio (DSRM), necesaria para operaciones de recuperación del Active Directory.*

---

![win-server cap12](img/win-server/cap12.png)

*Opciones adicionales de la configuración. El asistente ha determinado automáticamente el nombre NetBIOS del dominio como **RED**, derivado del componente de primer nivel del nombre DNS `red.local`. Este nombre NetBIOS se utilizará para el inicio de sesión en el formato heredado `RED\usuario`, siendo necesario para la compatibilidad con sistemas y aplicaciones que no soporten el formato UPN.*

---

![win-server cap13](img/win-server/cap13.png)

*Configuración de rutas para los archivos de Active Directory. Se muestran las ubicaciones predeterminadas de los tres componentes principales: la base de datos NTDS (`C:\Windows\NTDS`), los archivos de registro del directorio (`C:\Windows\NTDS`) y la carpeta compartida SYSVOL (`C:\Windows\SYSVOL`). Se mantienen las rutas por defecto al tratarse de una instalación en un entorno de laboratorio sin requisitos especiales de rendimiento o disponibilidad.*

---

![win-server cap14](img/win-server/cap14.png)

*Verificación de requisitos previos en curso. El asistente de configuración ejecuta una serie de comprobaciones automáticas para garantizar que el servidor cumple todas las condiciones necesarias para su promoción a controlador de dominio, validando aspectos como la conectividad de red, la disponibilidad de los servicios requeridos y la configuración del sistema.*

---

![win-server cap15](img/win-server/cap15.png)

*Resultado de la verificación de requisitos previos: **"All prerequisite checks passed successfully. Click 'Install' to begin installation."** Se presentan algunos avisos informativos sobre el uso de algoritmos de cifrado considerados débiles y sobre adaptadores de red sin dirección IPv6 estática, ninguno de los cuales bloquea el proceso de instalación en un entorno de laboratorio. El botón **Install** queda habilitado para proceder con la promoción.*

---

![win-server cap16](img/win-server/cap16.png)

*El servidor inicia el proceso de reinicio automático tras completar la promoción a Controlador de Dominio. La pantalla muestra el mensaje de "Restarting" de Windows. Una vez completado el reinicio, el servidor habrá asumido plenamente su rol como Controlador de Dominio del bosque `red.local`, con todos los servicios de AD DS, DNS y Kerberos activos.*

---

### 6.6 Creación de Usuarios

![win-server cap17](img/win-server/cap17.png)

*Formulario de creación de un nuevo objeto de usuario del dominio en **Active Directory Users and Computers**, accedido desde Server Manager → Tools. Navegando hasta el contenedor `red.local/Users` y seleccionando `New Object - User`, se completan los campos del asistente: nombre `usuario`, apellido `1`, nombre completo `usuario 1`, nombre de inicio de sesión de usuario (UPN) `u1@red.local` y nombre de inicio de sesión pre-Windows 2000 `RED\u1`.*

---

![win-server cap18](img/win-server/cap18.png)

*Pantalla de confirmación y resumen del nuevo usuario creado. Se muestra el nombre completo `usuario 1`, el UPN `u1@red.local` y la unidad organizativa de destino `red.local/Users`. Destaca la opción **"El usuario debe cambiar la contraseña en el próximo inicio de sesión"** marcada como activa, lo que obligará al usuario a establecer una nueva contraseña en su primer acceso al sistema, en conformidad con las buenas prácticas de seguridad.*

---

### 6.7 Políticas de Grupo (GPO)

![win-server cap19](img/win-server/cap19.png)

*Editor de administración de directivas de grupo (Group Policy Management Editor) mostrando la **Default Domain Policy**. El árbol de navegación muestra la ruta seguida: `Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy`. El panel de la derecha muestra los parámetros de política de contraseñas del dominio disponibles para su configuración.*

---

![win-server cap20](img/win-server/cap20.png)

*Configuración detallada de la **Directiva de contraseñas** aplicada al dominio `red.local`. Los valores establecidos son: historial de contraseñas recordadas de 5 contraseñas anteriores, vigencia máxima de 42 días, vigencia mínima de 1 día, longitud mínima de 12 caracteres, requisitos de complejidad habilitados y almacenamiento con cifrado reversible deshabilitado. Esta configuración establece una política de contraseñas robusta que se aplicará automáticamente a todos los usuarios del dominio al actualizar las directivas de grupo.*

---

## 7. Windows 10 Pro — Cliente del Dominio

### 7.1 Especificaciones de la VM

![win-client specs cap1](img/specs/win-client/cap1.png)

*Configuración de CPU y memoria de la VM Windows 10 Pro en Proxmox VE. Los recursos asignados son suficientes para soportar el sistema operativo Windows 10 y su uso como estación de trabajo cliente del dominio `red.local`.*

---

![win-client specs cap2](img/specs/win-client/cap2.png)

*Configuración de almacenamiento de la VM Windows 10. Se muestra el disco virtual de 40 GB asignado a esta máquina, dimensionado para el sistema operativo y las aplicaciones propias de un equipo cliente.*

---

![win-client specs cap3](img/specs/win-client/cap3.png)

*Configuración de la interfaz de red de la VM Windows 10 en Proxmox. La tarjeta de red virtual está conectada al bridge de la red interna del laboratorio, a través del cual el cliente recibirá su dirección IP por DHCP y se comunicará con el resto de servicios del entorno.*

---

![win-client specs cap4](img/specs/win-client/cap4.png)

*Panel de resumen general de la VM Windows 10 Pro en Proxmox VE, con el listado del hardware virtual asignado y el estado de la máquina.*

---

![win-client specs cap5](img/specs/win-client/cap5.png)

*Opciones avanzadas de configuración de la VM Windows 10 en Proxmox, incluyendo el tipo de máquina virtual, la configuración del agente QEMU y otros parámetros de virtualización específicos para sistemas operativos Windows.*

---

![win-client specs cap6](img/specs/win-client/cap6.png)

*Resumen consolidado de la configuración de hardware de la VM Windows 10, mostrando de forma unificada la especificación de CPU, memoria RAM, almacenamiento e interfaz de red.*

---

![win-client specs cap7](img/specs/win-client/cap7.png)

*Configuración de las opciones de arranque de la VM Windows 10 en Proxmox. Se visualiza el orden de dispositivos de arranque y la configuración de firmware requeridos para el arranque del sistema operativo.*

---

![win-client specs cap8](img/specs/win-client/cap8.png)

*Vista final de la configuración de hardware de la VM Windows 10 Pro previa al primer arranque. Todos los parámetros están correctamente definidos y la máquina está preparada para iniciar la instalación del sistema operativo.*

---

### 7.2 Instalación del SO

![win-client cap1](img/win-client/cap1.png)

*Pantalla inicial del instalador de Windows 10. A diferencia del servidor, el cliente se instala completamente en español: el idioma del sistema, el formato de fecha y moneda y la distribución de teclado se configuran todos en `Español (España, internacional)`. Esta configuración refleja el perfil de una estación de trabajo de usuario final en un entorno de empresa española.*

---

![win-client cap2](img/win-client/cap2.png)

*Selección de la edición del sistema operativo a instalar. Se elige **Windows 10 Pro** en arquitectura x64 con fecha de compilación 05/05/2023. La edición Pro es un requisito indispensable para poder unir el equipo a un dominio de Active Directory, funcionalidad no disponible en la edición Home.*

---

![win-client cap3](img/win-client/cap3.png)

*Selección del disco de destino para la instalación. El instalador detecta el espacio sin asignar de 40,0 GB en la unidad 0, que se selecciona como destino. El instalador creará automáticamente las particiones necesarias para el sistema operativo.*

---

![win-client cap4](img/win-client/cap4.png)

*Progreso inicial del proceso de instalación de Windows 10, al 3%, en la fase de preparación de archivos. El proceso continuará con la instalación de características, actualizaciones del sistema y la finalización, incluyendo varios reinicios automáticos hasta completar la instalación.*

---

![win-client cap5](img/win-client/cap5.png)

*Asistente de configuración inicial OOBE (Out-of-Box Experience) — paso de selección de región. Se elige **España** como región del dispositivo, configuración que afecta a los ajustes regionales del sistema como el formato de fecha, hora y las aplicaciones recomendadas por región.*

---

![win-client cap6](img/win-client/cap6.png)

*OOBE — configuración de la cuenta de usuario local inicial. Se introduce `usuario` como nombre para la cuenta local que se utilizará en el equipo antes de completar la unión al dominio. Esta cuenta local servirá de acceso temporal hasta que el equipo quede integrado en el dominio `red.local`.*

---

![win-client cap7](img/win-client/cap7.png)

*Escritorio de Windows 10 Pro recién instalado y listo para su configuración. El escritorio muestra el estado inicial del sistema con los elementos mínimos: el icono de la Papelera de reciclaje y el acceso directo a Microsoft Edge. La instalación se ha completado satisfactoriamente y el sistema está operativo.*

---

### 7.3 Verificación de Red

![win-client cap8](img/win-client/cap8.png)

*Verificación de la conectividad de red y la asignación de dirección IP mediante el comando `ipconfig` en el símbolo del sistema. La salida muestra que el cliente ha recibido correctamente una dirección IP dentro del rango DHCP configurado en ubuntu-server: `192.168.100.32`, con máscara `255.255.255.0` y puerta de enlace `192.168.100.1`. Esto confirma que el servidor DHCP está operativo y que el cliente puede comunicarse con el resto de la red del laboratorio. El sufijo de conexión DNS `example.org` visible en la salida será reemplazado por `red.local` una vez que el equipo se una al dominio.*

---

### 7.4 Unión al Dominio

![win-client cap9](img/win-client/cap9.png)

*Proceso de unión al dominio Active Directory a través de **Configuración → Sistema → Acerca de → Cambiar nombre del equipo o el dominio**. Se establece el nuevo nombre del equipo como `equipo1` y se selecciona la opción **Dominio**, introduciendo `red.local` como dominio de destino. Windows solicita las credenciales de una cuenta con permisos para agregar equipos al dominio, ya que esta operación requiere privilegios administrativos en el Active Directory.*

---

![win-client cap10](img/win-client/cap10.png)

*Confirmación exitosa de la unión al dominio. El cuadro de diálogo del sistema muestra el mensaje **"Se unió correctamente al dominio red.local."**, indicando que el proceso de integración con el Active Directory se ha completado sin errores. El equipo `equipo1` aparecerá automáticamente en el contenedor `Computers` del directorio activo. Se informa de la necesidad de reiniciar el sistema para que los cambios surtan efecto.*

---

![win-client cap11](img/win-client/cap11.png)

*Reinicio del sistema Windows 10 para completar el proceso de unión al dominio. La pantalla muestra el mensaje de "Reiniciando" de Windows. Tras este reinicio, el equipo reconocerá el dominio `red.local`, cargará las directivas de grupo del controlador de dominio y permitirá el inicio de sesión con cualquier cuenta de usuario del Active Directory.*

---

### 7.5 Aplicación de Directivas de Grupo

![win-client cap12](img/win-client/cap12.png)

*Pantalla de inicio de sesión del equipo tras el reinicio y la unión al dominio. La pantalla muestra la opción **"Otro usuario"** y, debajo de los campos de credenciales, el mensaje **"Se debe cambiar la contraseña del usuario antes de iniciar sesión"**. Este aviso confirma que la GPO de política de contraseñas configurada en el controlador de dominio está siendo aplicada correctamente: el usuario `u1` deberá establecer una nueva contraseña en este primer acceso al dominio.*

---

![win-client cap13](img/win-client/cap13.png)

*Inicio de sesión con las credenciales del usuario de dominio `u1`. Se introducen el nombre de usuario y la contraseña temporal en los campos de autenticación. En la parte inferior de la pantalla se puede leer **"Iniciar sesión en RED"**, confirmando que el equipo está procesando la autenticación contra el Controlador de Dominio del bosque `red.local` utilizando el nombre NetBIOS del dominio.*

---

![win-client cap14](img/win-client/cap14.png)

*Ejecución del comando `gpupdate /force` desde una consola de PowerShell con privilegios de Administrador. Este comando fuerza la descarga y aplicación inmediata de todas las directivas de grupo vigentes desde el Controlador de Dominio, sin esperar al ciclo de actualización automática. La salida del comando confirma la correcta aplicación de las directivas: **"La actualización de la directiva de equipo se completó correctamente"** y **"Se completó correctamente la actualización de directiva de usuario."***

---

![win-client cap15](img/win-client/cap15.png)

*Segunda ejecución de `gpupdate /force` que ratifica la correcta y estable aplicación de las directivas de grupo en el equipo cliente. Ambas actualizaciones —de directiva de equipo y de directiva de usuario— se completan sin errores, confirmando que las políticas de seguridad definidas en el servidor (incluyendo la política de contraseñas con longitud mínima de 12 caracteres, historial y complejidad) están plenamente activas y en vigor en la estación de trabajo `equipo1`.*

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
gpupdate /force            # Forzar aplicación de GPO
ipconfig                   # Verificar IP (cliente: rango DHCP .10-.50)
nltest /dsgetdc:red.local  # Verificar conectividad con el DC
```
