# 3. Active Directory (Windows Server 2022)

[⬅️ Previous: Bastion Host](DebianBastionHost_Checho.md) · [Back to README](README.md) · [Next: Windows 11 Workstation ➡️](Win11_User_Checho.md)

Installing the `ADDC01` domain controller with the AD DS and DNS roles, creating the `corp.lab` forest, the Organizational Units (OUs) and the first users.

> 🎥 Reference video: <https://www.youtube.com/watch?v=1XeDht_B-bA>

## Contents

- [Specs](#specs)
- [Installing Windows Server](#installing-windows-server)
- [Network and Internet access](#network-and-internet-access)
- [1. Install updates](#1-install-updates)
- [2. Install AD DS and DNS](#2-install-ad-ds-and-dns)
- [3. Promote the server to a domain controller](#3-promote-the-server-to-a-domain-controller)
- [4. Verify AD is working](#4-verify-ad-is-working)
- [5. Fix DNS](#5-fix-dns)
- [6. Create the Organizational Units](#6-create-the-organizational-units)
- [7. Create users](#7-create-users)

## Specs

| Parameter | Value |
| --------- | ----- |
| Hostname | `ADDC01` |
| Operating system | Windows Server 2022 Standard (Desktop Experience) |
| RAM / Disk | 8 GB / 80 GB |
| Network | Servers – `10.0.20.0/24` (OPT1 interface in pfSense) |
| IP / Gateway | `10.0.20.10` / `10.0.20.1` |
| Domain | `corp.lab` (NetBIOS: `CORP`) |

---

## Installing Windows Server

1. Download the ISO from Microsoft's official site.
2. Create the VM **without** attaching the ISO in the wizard ("I will install the operating system later"), to avoid VMware's *Easy Install*, which sometimes causes problems with this version.
3. Assign 8 GB of RAM and 80 GB of disk, mount the ISO in the CD drive and connect the network to the Servers subnet.
4. Install the **Windows Server 2022 Standard (Desktop Experience)** edition to get a desktop rather than just a console.
5. For the installation type choose **Custom: Install Microsoft Server Operating System only (advanced)**: a domain controller should start from a clean system (the other option is for upgrading an existing Windows install).

### Rename the computer

**Search → This PC → Properties → Rename this PC** → `ADDC01`. Reboot.

---

## Network and Internet access

Set a static IP `10.0.20.10`, gateway `10.0.20.1` and, **temporarily**, DNS `8.8.8.8`:

![Static IP with temporary DNS](images/ad/01_ip_estatica_dns_temporal.png)

In pfSense (**Firewall → Rules → OPT1**), create rules so the Servers subnet can reach the Internet to install roles and updates:

| Action | Protocol | Source | Destination | Port |
| ------ | -------- | ------ | ----------- | ---- |
| Pass | ICMP | OPT1 net | any | — (to test with `ping`) |
| Pass | TCP | OPT1 net | any | 80 (HTTP) |
| Pass | TCP | OPT1 net | any | 443 (HTTPS) |
| Pass | TCP/UDP | OPT1 net | any | 53 (DNS) |

![Outbound rules on OPT1](images/ad/02_reglas_opt1_salida.png)

Test from `ADDC01`:

```powershell
ping 10.0.20.1
ping 8.8.8.8
nslookup microsoft.com
```

---

## 1. Install updates

**Settings → Windows Update → Check for updates**. Repeat (rebooting as many times as needed) until you see *You're up to date*.

![Windows Update up to date](images/ad/03_windows_update.png)

---

## 2. Install AD DS and DNS

1. In **Server Manager → Manage → Add Roles and Features**, click *Next*.
2. Choose **Role-based or feature-based installation**.

   ![Installation type](images/ad/04_tipo_instalacion.png)

3. Select the `ADDC01` server.

   ![Destination server](images/ad/05_servidor_destino.png)

4. Check **Active Directory Domain Services** and **DNS Server**. Each time you check a role, a window pops up: click **Add Features**.

   ![AD DS role](images/ad/06_rol_adds.png)

   ![Features required by AD DS](images/ad/07_features_adds.png)

   ![Features required by DNS](images/ad/08_features_dns.png)

   ![Both roles selected](images/ad/09_roles_seleccionados.png)

5. Click *Next* through to the confirmation page and click **Install**.

   ![Installation confirmation](images/ad/10_confirmar_instalacion.png)

   ![Installation complete](images/ad/11_instalacion_completa.png)

---

## 3. Promote the server to a domain controller

1. Click the notifications flag in Server Manager → **Promote this server to a domain controller**.

   ![Promote to domain controller](images/ad/12_promover_dc.png)

2. Choose **Add a new forest** with *Root domain name* `corp.lab`.

   ![New corp.lab forest](images/ad/13_nuevo_bosque.png)

3. Keep **DNS Server** and **Global Catalog (GC)** checked, and set a strong **DSRM** (*Directory Services Restore Mode*) password.
4. Leave *DNS Options*, *NetBIOS Name* and *Paths* at their defaults.

   ![Database, log and SYSVOL paths](images/ad/14_rutas_ntds_sysvol.png)

   > 🔎 **Security note:** *Paths* shows where `NTDS.dit` is stored: the Active Directory database, which includes the password hashes for the whole domain. It's a top target for attackers: any suspicious access to or copy of this file means the entire domain is probably compromised.

5. Review the prerequisites and click **Install**. The server reboots automatically.

   ![Prerequisites check](images/ad/15_prerrequisitos.png)

---

## 4. Verify AD is working

After the reboot, the login screen shows `CORP\Administrator`, meaning the machine is now in the `CORP` domain:

![Logging in as CORP\Administrator](images/ad/16_login_corp_administrator.png)

In PowerShell:

```powershell
Get-ADDomain
```

If it returns information about `corp.lab`, AD is running.

![Get-ADDomain output](images/ad/17_get_addomain.png)

---

## 5. Fix DNS

Change the preferred DNS to `10.0.20.10` (if it didn't change automatically). Windows may have set it to `127.0.0.1`; in this lab the explicit IP is better because it makes troubleshooting DNS, AD replication and client errors easier, since you know exactly which server is being used.

![Preferred DNS 10.0.20.10](images/ad/18_dns_preferido.png)

Check in PowerShell as administrator:

```powershell
ipconfig /all          # DNS Servers should include 10.0.20.10
```

![ipconfig /all](images/ad/19_ipconfig_all.png)

```powershell
nslookup addc01.corp.lab
```

Expected result:

```text
Name:    addc01.corp.lab
Address: 10.0.20.10
```

![nslookup addc01.corp.lab](images/ad/20_nslookup_addc01.png)

This confirms that DNS is installed, the `corp.lab` zone exists, the DC registered its DNS record, and DNS can resolve the domain controller's hostname.

```powershell
Get-Service DNS,NTDS     # both should be Running
```

![DNS and NTDS services](images/ad/21_servicios_dns_ntds.png)

Check Internet access through pfSense again:

```powershell
ping 10.0.20.1
ping 8.8.8.8
nslookup microsoft.com
```

![Final connectivity](images/ad/22_conectividad_final.png)

📸 Take a snapshot: **"DC01 - Fresh Domain Controller"**.

---

## 6. Create the Organizational Units

OUs are created before joining machines to the domain. In **Server Manager → Tools → Active Directory Users and Computers**:

![Tools → ADUC menu](images/ad/23_menu_aduc.png)

1. On `corp.lab`, create an OU called **Corp** (the lab's root OU).

   ![Create OU button](images/ad/24_nueva_ou.png)

   ![Corp OU](images/ad/25_ou_corp.png)

2. Inside `Corp`, create:

   ```text
   Corp
   ├── Users
   ├── Workstations
   ├── Servers
   ├── Admins
   ├── Service Accounts
   └── Groups
   ```

   ![OU structure](images/ad/26_estructura_ous.png)

**Why not use the default `Users` and `Computers` folders?** They're not OUs but *containers*: GPOs can't be linked to them and they're mostly legacy defaults. OUs, on the other hand, support Group Policy and delegation, and they're how companies actually organize AD. For example:

```text
Workstations OU          Servers OU
      ↓                       ↓
Disable LLMNR            Other security policies
Enable Sysmon
Enable auditing
```

---

## 7. Create users

Two separate accounts are created to practice privilege separation: a regular user (`sergio`) and an admin user (`sergio.admin`).

### Regular user

In the **Corp → Users** OU → *Create a new user*:

![Create a new user](images/ad/27_nuevo_usuario.png)

![User sergio](images/ad/28_usuario_sergio.png)

Check **User must change password at next logon**.

![User created](images/ad/29_usuario_creado.png)

### Admin user

In the **Corp → Admins** OU, create `sergio.admin`:

![User sergio.admin](images/ad/30_usuario_sergio_admin.png)

Then: right-click the user → **Add to a group** → type `Domain Admins` → **Check Names** → **OK**.

![Add to Domain Admins](images/ad/31_grupo_domain_admins.png)

![User added to the group](images/ad/32_grupo_agregado.png)

---

## Next steps

1. ~~Finish AD deployment~~ ✅
2. [Join a workstation to the domain](Win11_User_Checho.md)
3. Verify DNS
4. [Verify Group Policy](SecurityBaselineGPOs_Checho.md)
5. Create administrative users and groups
6. [Deploy Wazuh](WazuhServer_Checho.md)
7. Harden the domain controller

[⬅️ Previous: Bastion Host](DebianBastionHost_Checho.md) · [Back to README](README.md) · [Next: Windows 11 Workstation ➡️](Win11_User_Checho.md)
