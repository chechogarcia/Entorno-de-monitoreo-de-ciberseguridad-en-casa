# Home Cybersecurity Monitoring Lab

A homelab that simulates a small company network to practice **security operations (SOC)**: firewall segmentation, secure administration through a bastion host, hardened Active Directory with GPOs, Sysmon telemetry, and centralized monitoring with Wazuh.

Everything runs as virtual machines on **VMware Workstation**, and every step is documented with screenshots.

## Background

The project started last year as an **Active Directory lab for my Network Security class** at university. After the semester ended I kept building on it: every time I learned something new (firewall segmentation, bastion hosts, MFA, Sysmon, SIEM), I added it to the lab. It has grown into a full monitoring environment, and it's still evolving.

## Contents

- [Tech stack](#tech-stack)
- [Lab status](#lab-status)
- [Step-by-step guides](#step-by-step-guides)
- [Beyond learning: putting the lab to real use](#beyond-learning-putting-the-lab-to-real-use)
- [Architecture](#architecture)
- [Addressing](#addressing)
- [Network segments](#network-segments)
- [Traffic flows](#traffic-flows)
- [pfSense rule philosophy](#pfsense-rule-philosophy)
- [Future expansion](#future-expansion)

## Tech stack

| Area | Tools |
| ---- | ----- |
| Virtualization | VMware Workstation |
| Firewall / segmentation | pfSense CE 2.8 |
| Identity | Windows Server 2022 – Active Directory, DNS, Group Policy |
| Endpoints | Windows 11 Pro |
| Administrative access | Debian 13 (bastion), OpenSSH, Google Authenticator (TOTP), ufw, fail2ban, AppArmor |
| Telemetry | Sysmon (SwiftOnSecurity config), Windows advanced auditing, PowerShell logging |
| SIEM | Wazuh 4.14 on Ubuntu Server 24.04 LTS |

## Lab status

| Component | Status |
| --------- | ------ |
| pfSense with segmented subnets | ✅ |
| Bastion host with SSH key + MFA | ✅ |
| Active Directory (`corp.lab`) with OUs and users | ✅ |
| Windows 11 workstations joined to the domain | ✅ |
| Security GPOs (LLMNR, PowerShell logging, auditing) | ✅ |
| Sysmon deployed automatically through GPO | ✅ |
| Wazuh with agents on ADDC01, WIN11-01 and bastion01 | ✅ |
| Additional DC hardening | ⏳ Planned |
| Attacker VM and custom detections | ⏳ Planned |
| Linux integration with AD | ⏳ Planned |
| AD CS (PKI) | ⏳ Planned |
| DMZ and VPN | ⏳ Planned |
| Physical server with Red Hat OpenShift | 🔄 Learning |

## Step-by-step guides

Recommended reading order:

| # | Guide | What it covers |
| - | ----- | -------------- |
| 1 | [pfSense firewall](Firewall_Checho.md) | Installation on VMware, WAN/LAN interfaces, `vmxnet3` troubleshooting |
| 2 | [Bastion host (Debian 13)](DebianBastionHost_Checho.md) | Minimal install, FQDN aliases in pfSense, AppArmor, ufw, fail2ban, SSH with key + TOTP |
| 3 | [Active Directory](ActiveDirectory_Checho.md) | Windows Server 2022, AD DS + DNS, `corp.lab` forest, OUs and users |
| 4 | [Windows 11 workstation](Win11_User_Checho.md) | Installation, domain join and verification |
| 5 | [Security GPOs](SecurityBaselineGPOs_Checho.md) | Workstation Baseline and DC Baseline: LLMNR, PowerShell logging, advanced auditing |
| 6 | [Sysmon](Deploy_Sysmon_Checho.md) | Manual install and automatic deployment through GPO |
| 7 | [Wazuh](WazuhServer_Checho.md) | Firewall rules, Ubuntu hardening, Wazuh install and agents |

---

## Beyond learning: putting the lab to real use

The goal isn't for this lab to be only an academic exercise. I want it to be infrastructure I actually use, so the security work is tested against real services and real traffic:

- **Practice in a controlled environment** — run attack simulations, test detections and try out new tools safely, without touching any system I don't own.
- **Self-hosted services** — host my own services behind the firewall, such as a **Minecraft server to play with my friends**, published through the DMZ and monitored by Wazuh like any other production service.
- **Personal projects** — use the same segmentation, bastion and monitoring setup as the base for anything else I want to self-host.

Running real services makes the lab more realistic: there's legitimate traffic to separate from suspicious activity, exposed services to protect, and uptime that actually matters.

### Next stage: physical server and OpenShift

We recently got a **physical server at home**, and I'm currently learning **Red Hat OpenShift** on it. The goal is to take what I've built in this VMware lab (segmentation, hardened access, centralized monitoring) and learn how to deploy a similar architecture the way it would run in a **real enterprise environment**: on dedicated hardware, with containers and an orchestration platform instead of standalone VMs.

---

## Architecture

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

The design follows the principle of **separation of duties**: users, production services, management, security operations, internet-facing services and remote access are all isolated from each other. This lets the lab grow (AD CS, Security Onion, more domain controllers) and supports both defensive operations and controlled attack scenarios.

## Addressing

| Segment | Subnet | pfSense interface | Gateway | Hosts |
| ------- | ------ | ----------------- | ------- | ----- |
| Users | `10.0.10.0/24` | LAN | `10.0.10.1` | `WIN11-01` (`10.0.10.110`), `WIN11-02` |
| Servers | `10.0.20.0/24` | OPT1 | `10.0.20.1` | `ADDC01` (`10.0.20.10`) |
| DMZ | `10.0.30.0/27` | — | — | Planned |
| Management | `10.0.40.0/28` | OPT3 | `10.0.40.1` | `bastion01` (`10.0.40.10`), Admin VM (`10.0.40.11`) |
| Security | `10.0.50.0/27` | OPT4 | `10.0.50.1` | `WAZUH01` (`10.0.50.10`) |
| VPN | `10.0.60.0/24` | — | — | Planned |

| Item | Value |
| ---- | ----- |
| AD domain | `corp.lab` (NetBIOS `CORP`) |
| Bastion domain | `lab.internal` |
| Internal DNS | `10.0.20.10` (ADDC01) |

---

## Network segments

### 1. Users – `10.0.10.0/24`

Simulates employee devices.

```text
WIN11-01
WIN11-02
Ubuntu Client         (planned)
Attack VM             (planned)
```

**Purpose:** domain users, daily workstation activity, phishing simulation, malware execution, Red Team testing, Atomic Red Team and lateral movement exercises.

These machines must **not** have administrative privileges over the environment.

### 2. Servers – `10.0.20.0/24`

Infrastructure services: the "production" assets.

```text
ADDC01
File Server                 (planned)
SQL Server                  (planned)
IIS Server                  (planned)
Linux Application Server    (planned)
GitLab                      (planned)
```

**Purpose:** Active Directory, DNS, file sharing, internal applications and databases.

### 3. DMZ – `10.0.30.0/27`

Anything exposed to external users, such as the planned game server.

```text
Public web server
Reverse proxy
Game server (Minecraft)
VPN gateway (optional)
Mail relay
FTP/SFTP
```

These systems must **never** have unrestricted access to internal networks.

### 4. Management – `10.0.40.0/28`

Administrative systems only.

```text
Admin VM
Bastion Host
Ansible Server   (planned)
WSUS             (planned)
vCenter          (planned)
```

**Purpose:** SSH, RDP, GPO management, pfSense and hypervisor management.

No regular user should ever access this subnet.

### 5. Security – `10.0.50.0/27`

The lab's SOC.

```text
Wazuh
TheHive          (planned)
Cortex           (planned)
MISP             (planned)
Velociraptor     (planned)
REMnux           (planned)
Malware sandbox  (planned)
```

**Purpose:** detection, monitoring, DFIR, threat intelligence and incident response.

Only administrators should be able to reach this subnet.

### 6. VPN – `10.0.60.0/24`

Remote administrators (OpenVPN / WireGuard clients).

VPN users must **not** land directly in Users or Servers; they go through the bastion instead:

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

This models a secure remote administration workflow.

---

## Traffic flows

### User workstation

```text
WIN11
      │
      ├──── DNS ─────────► ADDC01
      ├──── Kerberos ────► ADDC01
      ├──── LDAP ────────► ADDC01
      ├──── SMB ─────────► File Server
      └──── Wazuh Agent ─► Wazuh
```

### Administrator

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

Agents initiate the connection to Wazuh; Wazuh doesn't need to open sessions back to the endpoints.

---

## pfSense rule philosophy

Default policy: **deny everything**, and only allow what is explicitly needed.

| Segment | Allow |
| ------- | ----- |
| **Users** | DNS, Kerberos, LDAP, SMB (if needed), HTTP/HTTPS to the Internet, agent → Wazuh |
| **Servers** | DNS, domain replication (if another DC is added), application traffic, agent → Wazuh |
| **Management** | SSH, RDP, WinRM, HTTPS, pfSense management, Wazuh dashboard |
| **Security** | Incoming Wazuh agents, incoming syslog (if configured), Internet for updates, admin access from Management |
| **VPN** | VPN → Management, VPN → Bastion, VPN → Wazuh dashboard (optional) |

VPN clients must **not** get unrestricted access to the Users network.

---

## Future expansion

| Segment | Ideas |
| ------- | ----- |
| **Servers** | PKI / AD CS, Exchange (or a mail server), Linux web application, test SIEM server |
| **DMZ** | Minecraft server for friends, reverse proxy, personal web services |
| **Security** | Suricata, Zeek, Security Onion, YARA scanning, Sigma rule testing, MITRE ATT&CK Navigator, OSQuery Fleet |
| **Management** | Ansible, Windows Admin Center, patch management, automation server |
| **Platform** | Move to the physical server: rebuild the architecture on Red Hat OpenShift, containerized services, enterprise-style deployment |
