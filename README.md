# Entorno de monitoreo de ciberseguridad en casa

Laboratorio casero (*homelab*) que simula la red de una pequeña empresa para practicar **operaciones de seguridad (SOC)**: segmentación con firewall, administración segura mediante un bastion host, Active Directory endurecido con GPOs, telemetría con Sysmon y monitoreo centralizado con Wazuh.

Todo corre en máquinas virtuales sobre **VMware Workstation**, y cada paso está documentado con capturas de pantalla.

## Contenido

- [Tecnologías](#tecnologías)
- [Estado del laboratorio](#estado-del-laboratorio)
- [Guías paso a paso](#guías-paso-a-paso)
- [Arquitectura](#arquitectura)
- [Direccionamiento](#direccionamiento)
- [Segmentos de red](#segmentos-de-red)
- [Flujos de tráfico](#flujos-de-tráfico)
- [Filosofía de reglas en pfSense](#filosofía-de-reglas-en-pfsense)
- [Expansión futura](#expansión-futura)

## Tecnologías

| Área | Herramientas |
| ---- | ------------ |
| Virtualización | VMware Workstation |
| Firewall / segmentación | pfSense CE 2.8 |
| Identidad | Windows Server 2022 – Active Directory, DNS, Group Policy |
| Endpoints | Windows 11 Pro |
| Acceso administrativo | Debian 13 (bastion), OpenSSH, Google Authenticator (TOTP), ufw, fail2ban, AppArmor |
| Telemetría | Sysmon (configuración SwiftOnSecurity), auditoría avanzada de Windows, registro de PowerShell |
| SIEM | Wazuh 4.14 sobre Ubuntu Server 24.04 LTS |

## Estado del laboratorio

| Componente | Estado |
| ---------- | ------ |
| pfSense con subredes segmentadas | ✅ |
| Bastion host con llave SSH + MFA | ✅ |
| Active Directory (`corp.lab`) con OUs y usuarios | ✅ |
| Workstations Windows 11 unidas al dominio | ✅ |
| GPOs de seguridad (LLMNR, PowerShell logging, auditoría) | ✅ |
| Sysmon desplegado automáticamente por GPO | ✅ |
| Wazuh con agentes en ADDC01, WIN11-01 y bastion01 | ✅ |
| Endurecimiento adicional del DC | ⏳ Pendiente |
| VM de ataque y creación de detecciones | ⏳ Pendiente |
| Integración de Linux con AD | ⏳ Pendiente |
| AD CS (PKI) | ⏳ Pendiente |
| DMZ y VPN | ⏳ Pendiente |

## Guías paso a paso

Se recomienda seguirlas en este orden:

| # | Guía | Contenido |
| - | ---- | --------- |
| 1 | [Firewall pfSense](Firewall_Checho.md) | Instalación en VMware, interfaces WAN/LAN, solución de problemas con `vmxnet3` |
| 2 | [Bastion Host (Debian 13)](DebianBastionHost_Checho.md) | Instalación mínima, alias FQDN en pfSense, AppArmor, ufw, fail2ban, SSH con llave + TOTP |
| 3 | [Active Directory](ActiveDirectory_Checho.md) | Windows Server 2022, AD DS + DNS, bosque `corp.lab`, OUs y usuarios |
| 4 | [Workstation Windows 11](Win11_User_Checho.md) | Instalación, unión al dominio y verificación |
| 5 | [GPOs de seguridad](SecurityBaselineGPOs_Checho.md) | Workstation Baseline y DC Baseline: LLMNR, PowerShell logging, auditoría avanzada |
| 6 | [Sysmon](Deploy_Sysmon_Checho.md) | Instalación manual y despliegue automático por GPO |
| 7 | [Wazuh](WazuhServer_Checho.md) | Reglas de firewall, hardening de Ubuntu, instalación de Wazuh y agentes |

---

## Arquitectura

```text
                                    Internet
                                        │
                                    ISP / WAN
                                        │
                                    pfSense FW
                                        │
  ───────────────────────────────────────────────────────────────────
  │          │             │            │             │             │
  Users     Servers      DMZ       Management     Security        VPN
  10.0.10   10.0.20      10.0.30   10.0.40        10.0.50         10.0.60
```

El diseño aplica el principio de **separación de funciones**: usuarios, servicios de producción, administración, operaciones de seguridad, servicios expuestos a Internet y acceso remoto están aislados entre sí. Así el laboratorio puede crecer (AD CS, Security Onion, más controladores de dominio) y sirve tanto para operaciones defensivas como para escenarios de ataque controlados.

## Direccionamiento

| Segmento | Subred | Interfaz pfSense | Gateway | Hosts |
| -------- | ------ | ---------------- | ------- | ----- |
| Users | `10.0.10.0/24` | LAN | `10.0.10.1` | `WIN11-01` (`10.0.10.110`), `WIN11-02` |
| Servers | `10.0.20.0/24` | OPT1 | `10.0.20.1` | `ADDC01` (`10.0.20.10`) |
| DMZ | `10.0.30.0/27` | — | — | Pendiente |
| Management | `10.0.40.0/28` | OPT3 | `10.0.40.1` | `bastion01` (`10.0.40.10`), Admin VM (`10.0.40.11`) |
| Security | `10.0.50.0/27` | OPT4 | `10.0.50.1` | `WAZUH01` (`10.0.50.10`) |
| VPN | `10.0.60.0/24` | — | — | Pendiente |

| Dato | Valor |
| ---- | ----- |
| Dominio de AD | `corp.lab` (NetBIOS `CORP`) |
| Dominio del bastion | `lab.internal` |
| DNS interno | `10.0.20.10` (ADDC01) |

---

## Segmentos de red

### 1. Users – `10.0.10.0/24`

Simula los equipos de los empleados.

```text
WIN11-01
WIN11-02
Ubuntu Client         (futuro)
Attack VM             (futuro)
```

**Propósito:** usuarios del dominio, actividad diaria, simulación de phishing, ejecución de malware, pruebas de Red Team, Atomic Red Team y ejercicios de movimiento lateral.

Estos equipos **no** deben tener privilegios administrativos sobre el entorno.

### 2. Servers – `10.0.20.0/24`

Servicios de infraestructura: los activos de "producción".

```text
ADDC01
File Server                 (futuro)
SQL Server                  (futuro)
IIS Server                  (futuro)
Linux Application Server    (futuro)
GitLab                      (futuro)
```

**Propósito:** Active Directory, DNS, archivos compartidos, aplicaciones internas y bases de datos.

### 3. DMZ – `10.0.30.0/27`

Todo lo expuesto a usuarios externos.

```text
Servidor web público
Reverse proxy
VPN gateway (opcional)
Mail relay
FTP/SFTP
```

Estos sistemas **nunca** deben tener acceso libre a las redes internas.

### 4. Management – `10.0.40.0/28`

Solo sistemas administrativos.

```text
Admin VM
Bastion Host
Ansible Server   (futuro)
WSUS             (futuro)
vCenter          (futuro)
```

**Propósito:** SSH, RDP, gestión de GPOs, administración de pfSense y del hipervisor.

Ningún usuario regular debe acceder a esta subred.

### 5. Security – `10.0.50.0/27`

El SOC del laboratorio.

```text
Wazuh
TheHive          (futuro)
Cortex           (futuro)
MISP             (futuro)
Velociraptor     (futuro)
REMnux           (futuro)
Malware sandbox  (futuro)
```

**Propósito:** detección, monitoreo, DFIR, threat intelligence y respuesta a incidentes.

Solo los administradores deben poder alcanzar esta subred.

### 6. VPN – `10.0.60.0/24`

Administradores remotos (clientes OpenVPN / WireGuard).

Los usuarios de VPN **no** deben llegar directamente a Users ni a Servers, sino pasar por el bastion:

```text
VPN Client
      │
      ▼
Management Network
      │
      ▼
Bastion
      │
      ▼
Servers
```

Esto modela un flujo seguro de administración remota.

---

## Flujos de tráfico

### Workstation de usuario

```text
WIN11
      │
      ├──── DNS ─────────► ADDC01
      ├──── Kerberos ────► ADDC01
      ├──── LDAP ────────► ADDC01
      ├──── SMB ─────────► File Server
      └──── Wazuh Agent ─► Wazuh
```

### Administrador

```text
Admin VM
      │
      ├──── SSH ─────► Bastion
      ├──── HTTPS ───► Wazuh
      ├──── RDP ─────► Servers
      ├──── RSAT ────► AD
      └──── HTTPS ───► pfSense
```

### Bastion

```text
Bastion
      │
      ├──── SSH ─────► Linux Servers
      ├──── RDP ─────► Windows Servers
      ├──── WinRM ───► Windows
      └──── HTTPS ───► Wazuh
```

### Wazuh

```text
WIN11 Agent ────┐
ADDC01 Agent ───┤
Bastion Agent ──┼──► Wazuh
Ubuntu Agent ───┘
```

Los agentes inician la conexión hacia Wazuh; Wazuh no necesita abrir sesiones hacia los endpoints.

---

## Filosofía de reglas en pfSense

Política por defecto: **denegar todo** y permitir solo lo explícitamente necesario.

| Segmento | Permitir |
| -------- | -------- |
| **Users** | DNS, Kerberos, LDAP, SMB (si hace falta), HTTP/HTTPS a Internet, agente → Wazuh |
| **Servers** | DNS, replicación de dominio (si se agrega otro DC), tráfico de aplicaciones, agente → Wazuh |
| **Management** | SSH, RDP, WinRM, HTTPS, administración de pfSense, dashboard de Wazuh |
| **Security** | Recibir agentes de Wazuh, recibir syslog (si se configura), Internet para actualizaciones, acceso administrativo desde Management |
| **VPN** | VPN → Management, VPN → Bastion, VPN → dashboard de Wazuh (opcional) |

Los clientes VPN **no** deben tener acceso libre a la red de Users.

---

## Expansión futura

| Segmento | Ideas |
| -------- | ----- |
| **Servers** | PKI / AD CS, Exchange (o servidor de correo), aplicación web Linux, servidor SIEM de pruebas |
| **Security** | Suricata, Zeek, Security Onion, escaneo con YARA, pruebas de reglas Sigma, MITRE ATT&CK Navigator, OSQuery Fleet |
| **Management** | Ansible, Windows Admin Center, gestión de parches, servidor de automatización |
