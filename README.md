# Entorno-de-monitoreo-de-ciberseguridad-en-casa


```
                                    Internet
                                        │
                                    ISP / WAN
                                        │
                                    pfSense FW
                                        │
  ───────────────────────────────────────────────────────────────────
  │          │             │            │             │             │
  │          │             │            │             │             │
  Users     Servers      DMZ       Management     Security        VPN
  10.0.10   10.0.20      10.0.30   10.0.40        10.0.50         10.0.60
```
--------------------------------------------------------------------------
# 1. Users Network (10.0.10.0/24)

This subnet simulates normal employee devices.

## Systems
```
WIN11-01
Ubuntu Client
Attack-VM
Future Windows 10/11 Client
Future Linux Client
```

Purpose:

Domain users
Daily workstation activity
Phishing simulation
Malware execution
Red Team testing
Atomic Red Team
Lateral movement exercises

These machines should have no administrative privileges over the environment.

--------------------------------------------------------------------------
# 2. Servers Network (10.0.20.0/24)

Infrastructure services live here.
```
ADDC01
Future File Server
Future SQL Server
Future IIS Server
Future Linux Application Server
Future GitLab
```
## Purpose:

- Active Directory
- DNS
- File Sharing
- Internal applications
- Databases

These are the production assets.

--------------------------------------------------------------------------
# 3. DMZ (10.0.30.0/24)

Anything exposed to external users belongs here.
```
Public Web Server
Reverse Proxy
VPN Gateway (optional)
Mail Relay
FTP/SFTP
```
These systems should never have unrestricted access to internal networks.

--------------------------------------------------------------------------
# 4. Management Network (10.0.40.0/24)

Administrative systems only.
```
Admin VM
Bastion Host
Future Ansible Server
Future WSUS
Future vCenter
```
Purpose:

- SSH
- RDP
- GPO management
- pfSense management
- Hypervisor management

No regular user should ever access this subnet.

--------------------------------------------------------------------------
# 5. Security Network (10.0.50.0/24)

Your SOC.
```
Wazuh
TheHive
Cortex
MISP
Velociraptor
REMnux
Malware Sandbox
```
Purpose:

Detection
Monitoring
DFIR
Threat Intelligence
Incident Response

This subnet should only be reachable by administrators.

--------------------------------------------------------------------------
# 6. VPN Network (10.0.60.0/24)

Remote administrators.
```
OpenVPN Clients
WireGuard Clients
Future Remote Laptop
```
These users should not land directly inside Users or Servers.

Instead:
```
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

--------------------------------------------------------------------------
--------------------------------------------------------------------------
# Traffic Flow
## User workstation
```
WIN11
      │
      ├──── DNS ─────────► ADDC01
      ├──── Kerberos ────► ADDC01
      ├──── LDAP ────────► ADDC01
      ├──── SMB ─────────► File Server
      └──── Wazuh Agent ─► Wazuh
```
## Administrator
```
Admin VM
      │
      ├──── SSH ─────► Bastion
      ├──── HTTPS ───► Wazuh
      ├──── RDP ─────► Servers
      ├──── RSAT ────► AD
      └──── HTTPS ───► pfSense
```

## Bastion
```
Bastion
      │
      ├──── SSH ─────► Linux Servers
      ├──── RDP ─────► Windows Servers
      ├──── WinRM ───► Windows
      └──── HTTPS ───► Wazuh
```

## Wazuh
```
WIN11 Agent
        │
ADDC01 Agent
        │
Bastion Agent
        │
Ubuntu Agent
        │
────────▼────────
     Wazuh
```
Notice that agents initiate the connection to Wazuh. Wazuh generally does not need to initiate sessions back to endpoints.

--------------------------------------------------------------------------
--------------------------------------------------------------------------
# pfSense Philosophy

Default policy:
```
Deny Everything
```
Only explicitly allow:

## Users
- DNS
- Kerberos
- LDAP
- SMB (if needed)
- HTTP/HTTPS to Internet
- Wazuh Agent → Wazuh

## Servers
Allow:
- Domain replication (if you later add another DC)
- DNS
- Required application traffic
- Wazuh Agent

## Management
Allow:
- SSH
- RDP
- WinRM
- HTTPS
- pfSense management
- Wazuh Dashboard

## Security

Allow:
- Receive Wazuh agents
- Receive syslog (if configured)
- Internet access for updates
- Admin access from the Management network

VPN
Allow:
- VPN → Management
- VPN → Bastion
- VPN → Wazuh Dashboard (optional)

Do not allow VPN clients unrestricted access to the Users network.

--------------------------------------------------------------------------
--------------------------------------------------------------------------
# Future Expansion

As your lab grows, consider adding:

## Servers
- PKI / Active Directory Certificate Services (AD CS)
- Exchange (or an email server)
- Linux web application
- SIEM test server

## Security
- Suricata sensor
- Zeek sensor
- Security Onion
- YARA scanning
- Sigma rule testing
- MITRE ATT&CK Navigator
- OSQuery Fleet

## Management
- Ansible
- Windows Admin Center
- Patch management
- Automation server


--------------------------------------------------------------------------
--------------------------------------------------------------------------
# Final Recommendation

I think this design gives you an excellent balance between realism and maintainability:
```
Internet
    │
pfSense
    │
├── Users (10.0.10.0/24)
│      └── Endpoints and attack simulations
│
├── Servers (10.0.20.0/24)
│      └── AD, DNS, file servers, applications
│
├── DMZ (10.0.30.0/24)
│      └── Public-facing services
│
├── Management (10.0.40.0/24)
│      └── Bastion, Admin VM, administrative tooling
│
├── Security (10.0.50.0/24)
│      └── Wazuh, TheHive, Cortex, MISP, Velociraptor
│
└── VPN (10.0.60.0/24)
       └── Remote administrators connecting through the Bastion
```
This architecture follows the principle of separation of duties: users, production services, management, security operations, internet-facing services, and remote access are all isolated. It's a design that will scale well as you add technologies like AD CS, Security Onion, or additional domain controllers, and it provides a strong foundation for practicing both defensive operations and controlled attack scenarios.
