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
