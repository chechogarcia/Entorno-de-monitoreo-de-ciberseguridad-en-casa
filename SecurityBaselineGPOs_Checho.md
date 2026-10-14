# 5. GPOs de seguridad (Security Baselines)

[⬅️ Anterior: Workstation Windows 11](Win11_User_Checho.md) · [Volver al README](README.md) · [Siguiente: Sysmon ➡️](Deploy_Sysmon_Checho.md)

Una **GPO** (*Group Policy Object*, Directiva de Grupo) permite a los administradores gestionar de forma centralizada la configuración del sistema operativo, aplicaciones y seguridad de muchos usuarios y equipos del dominio.

En este laboratorio se crean dos líneas base orientadas a **detección**:

| GPO | Vinculada a | Objetivo |
| --- | ----------- | -------- |
| **Workstation Baseline** | OU `Corp → Workstations` | Desactivar LLMNR y generar telemetría en los equipos de usuario |
| **DC Baseline** | OU `Domain Controllers` | Lo mismo, más auditoría específica de Active Directory |

## Contenido

- [Workstation Baseline](#workstation-baseline)
  - [1. Desactivar LLMNR](#1-desactivar-llmnr)
  - [2. Registro de PowerShell](#2-registro-de-powershell)
  - [3. Políticas de auditoría](#3-políticas-de-auditoría)
- [DC Baseline](#dc-baseline)
- [Aplicar y verificar](#aplicar-y-verificar)

---

## Workstation Baseline

1. **Server Manager → Tools → Group Policy Management**.
2. Navegar a **Domains → corp.lab → Corp → Workstations**.
3. Clic derecho sobre la OU → **Create a GPO in this domain, and Link it here...** → nombre: `Workstation Baseline`.

   ![Nueva GPO Workstation Baseline](images/gpo/01_nueva_gpo_workstation.png)

4. Clic derecho sobre la GPO → **Edit**.

### 1. Desactivar LLMNR

**Computer Configuration → Policies → Administrative Templates → Network → DNS Client** → **Turn off multicast name resolution** → **Enabled**.

![Desactivar LLMNR](images/gpo/02_desactivar_llmnr.png)

> **¿Por qué desactivar LLMNR?** Cuando falla el DNS, Windows pregunta el nombre por *multicast* a toda la red local mediante LLMNR (*Link-Local Multicast Name Resolution*). Un atacante en la misma red puede responder haciéndose pasar por el servidor buscado (**LLMNR poisoning**, por ejemplo con *Responder*), lograr que la víctima se autentique contra él y capturar su hash NetNTLMv2 (**robo de credenciales**). Ese hash puede crackearse o retransmitirse a otros servidores (**relay**) sin necesidad de conocer la contraseña.

### 2. Registro de PowerShell

**Computer Configuration → Policies → Administrative Templates → Windows Components → Windows PowerShell**

![Políticas de Windows PowerShell](images/gpo/03_powershell_politicas.png)

- **Turn on Module Logging** → **Enabled**. En *Module Names → Show...* escribir `*` para registrar todos los módulos.

  ![Module Logging habilitado](images/gpo/04_module_logging.png)

  ![Module Names = *](images/gpo/05_module_names.png)

- **Turn on PowerShell Script Block Logging** → **Enabled**.

  ![Script Block Logging habilitado](images/gpo/06_script_block_logging.png)

### 3. Políticas de auditoría

**Computer Configuration → Policies → Windows Settings → Security Settings → Advanced Audit Policy Configuration → Audit Policies**

![Advanced Audit Policy](images/gpo/07_advanced_audit.png)

En cada subcategoría de la tabla, marcar **Configure the following audit events** con **Success** y **Failure**:

![Success y Failure](images/gpo/08_success_failure.png)

| Categoría | Subcategorías | Qué permite ver |
| --------- | ------------- | --------------- |
| **Logon/Logoff** | Audit Logon · Audit Logoff · Audit Special Logon · Audit Other Logon/Logoff Events | Inicios de sesión exitosos y fallidos, RDP, logins de administradores |
| **Account Logon** | Audit Credential Validation · Audit Kerberos Authentication Service · Audit Kerberos Service Ticket Operations · Audit Other Account Logon Events | Monitoreo de la autenticación en AD |
| **Account Management** | Audit User Account Management · Audit Security Group Management · Audit Computer Account Management · Audit Distribution Group Management · Audit Other Account Management Events | Usuarios creados o borrados, cambios de contraseña, usuarios agregados a Domain Admins |
| **Detailed Tracking** | Audit Process Creation | Creación de procesos (Event ID 4688) |
| **Policy Change** | Audit Audit Policy Change · Audit Authentication Policy Change · Audit Authorization Policy Change | Manipulación de la configuración de seguridad |
| **Privilege Use** | Audit Sensitive Privilege Use | Abuso de privilegios poderosos |
| **Object Access** (mínimo) | Audit File Share · Audit Detailed File Share | Acceso a recursos compartidos (útil cuando haya file servers) |

![Logon/Logoff](images/gpo/09_logon_logoff.png)

![Account Logon](images/gpo/10_account_logon.png)

![Account Management](images/gpo/11_account_management.png)

![Detailed Tracking](images/gpo/12_detailed_tracking.png)

![Policy Change](images/gpo/15_policy_change.png)

![Object Access](images/gpo/16_object_access.png)

#### Incluir la línea de comandos en los eventos de creación de procesos

**Computer Configuration → Policies → Administrative Templates → System → Audit Process Creation** → **Include command line in process creation events** → **Enabled**.

Así el Event ID 4688 incluye los argumentos (por ejemplo `powershell.exe -enc ...` en lugar de solo `powershell.exe`).

![Audit Process Creation](images/gpo/13_audit_process_creation.png)

![Include command line habilitado](images/gpo/14_command_line_enabled.png)

> ⚠️ Con esta opción, cualquier usuario que pueda leer el log de seguridad verá los argumentos de los comandos, que a veces incluyen datos sensibles.

#### Evitar por ahora

Estas subcategorías generan un volumen de logs muy alto; es mejor introducirlas más adelante y de forma controlada:

- Audit File System
- Audit Registry
- Audit Kernel Object
- Audit Handle Manipulation
- Audit Removable Storage
- Audit Filtering Platform Packet Drop
- Audit Filtering Platform Connection

---

## DC Baseline

Para el controlador de dominio se usa la OU **Domain Controllers** que Windows crea por defecto en la raíz de `corp.lab`.

1. **Tools → Group Policy Management → Forest → Domains → corp.lab → Domain Controllers**.
2. Clic derecho → **Create a GPO in this domain, and Link it here...** → nombre: `DC Baseline`.

   La GPO queda vinculada a la OU `Domain Controllers` de la **raíz del dominio** (donde está el objeto `ADDC01`), junto a la `Default Domain Controllers Policy`:

   ![DC Baseline vinculada a la OU Domain Controllers](images/gpo/17b_dc_baseline_vinculada.png)

   > 💡 Vincularla a la OU por defecto es lo correcto: Microsoft no recomienda mover los controladores de dominio fuera de ella. Si se vinculara a otra OU con el mismo nombre (por ejemplo, dentro de `Corp`) y sin el DC adentro, la política no se aplicaría.

3. Clic derecho → **Edit** y configurar:

| # | Configuración | Detalle |
| - | ------------- | ------- |
| 1 | Desactivar LLMNR | Igual que en la [Workstation Baseline](#1-desactivar-llmnr) |
| 2 | Auditoría específica de DC | Las mismas políticas de la Workstation Baseline, más las de abajo |
| 3 | Registro de PowerShell | Igual que en la [Workstation Baseline](#2-registro-de-powershell) |
| 4 | Línea de comandos en eventos de procesos | **Administrative Templates → System → Audit Process Creation → Include command line in process creation events** |

**Auditoría adicional para el DC** (Success y Failure):

| Categoría | Subcategoría |
| --------- | ------------ |
| DS Access | Directory Service Changes |
| DS Access | Directory Service Access |
| Account Management | User Account Management |
| Account Management | Security Group Management |

---

## Aplicar y verificar

### Forzar la actualización de políticas

En **ADDC01** (PowerShell como administrador) y en **WIN11-01**:

```powershell
gpupdate /force
```

Resultado esperado: `Computer Policy update has completed successfully.`

![gpupdate en ADDC01](images/gpo/18_gpupdate_dc.png)

![gpupdate en WIN11-01](images/gpo/19_gpupdate_win11.png)

### Confirmar qué GPO se aplicaron

```powershell
gpresult /scope computer /r
```

| Equipo | *Applied Group Policy Objects* esperados |
| ------ | ---------------------------------------- |
| WIN11-01 | `Workstation Baseline`, `Default Domain Policy` |
| ADDC01 | `DC Baseline`, `Default Domain Controllers Policy` |

### Pruebas puntuales

**LLMNR desactivado**

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient"
```

Buscar `EnableMulticast : 0`.

**Registro de PowerShell**

```powershell
Get-Process
Get-WinEvent -LogName "Microsoft-Windows-PowerShell/Operational" -MaxEvents 20
```

Si aparecen eventos, el Module Logging / Script Block Logging funciona.

**Auditoría de creación de procesos**

```cmd
notepad.exe
```

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4688} -MaxEvents 10
```

Si aparece el Event ID 4688, la auditoría de procesos funciona.

---

## Siguientes pasos

- 📸 Snapshot de pfSense, ADDC01, WIN11-01 y Bastion01: **"VM__Pre_Sysmon_Wazuh"**
- [Desplegar Sysmon](Deploy_Sysmon_Checho.md)
- [Montar Wazuh e instalar los agentes](WazuhServer_Checho.md)
- Verificar la telemetría
- Endurecer más el DC
- Montar la VM de ataque y generar detecciones
- Integrar Linux con AD
- Agregar AD CS

[⬅️ Anterior: Workstation Windows 11](Win11_User_Checho.md) · [Volver al README](README.md) · [Siguiente: Sysmon ➡️](Deploy_Sysmon_Checho.md)
