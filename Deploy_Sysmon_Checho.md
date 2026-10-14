# 6. Deploying Sysmon

[⬅️ Previous: Security GPOs](SecurityBaselineGPOs_Checho.md) · [Back to README](README.md) · [Next: Wazuh ➡️](WazuhServer_Checho.md)

**Sysmon** (*System Monitor*, from Sysinternals) logs detailed Windows activity —process creation, network connections, DNS queries, created files, registry changes— which is then sent to Wazuh to build detections.

It's first installed manually on `ADDC01` and `WIN11-01`, then automated for every new machine through a GPO.

## Contents

- [Downloads](#downloads)
- [Manual installation](#manual-installation)
- [Key Event IDs](#key-event-ids)
- [Automatic deployment through Group Policy](#automatic-deployment-through-group-policy)

## Downloads

| Resource | Link |
| -------- | ---- |
| Sysmon | <https://learn.microsoft.com/sysinternals/downloads/sysmon> |
| Config used: SwiftOnSecurity | <https://github.com/SwiftOnSecurity/sysmon-config> |
| Alternative: sysmon-modular (Olaf Hartong) | <https://github.com/olafhartong/sysmon-modular> |

Both configurations are recommended in [Microsoft's documentation](https://learn.microsoft.com/windows/security/operating-system-security/sysmon/how-to-enable-sysmon).

Sysmon comes as a `.zip` with `Sysmon.exe`, `Sysmon64.exe`, `Sysmon64a.exe` and `Eula.txt`. The SwiftOnSecurity configuration is a single `sysmonconfig-export.xml` file.

---

## Manual installation

### 1. Prepare the folder

Create `C:\Sysmon` and copy `Sysmon64.exe` and `sysmonconfig-export.xml` into it.

![C:\Sysmon folder](images/sysmon/01_carpeta_sysmon.png)

### 2. Install

In PowerShell as administrator:

```powershell
cd C:\Sysmon
.\Sysmon64.exe -accepteula -i .\sysmonconfig-export.xml
```

Expected output:

```text
Configuration file validated.
Sysmon64 installed.
SysmonDrv installed.
Starting SysmonDrv.
SysmonDrv started.
Starting Sysmon64..
Sysmon64 started.
```

![Installing Sysmon](images/sysmon/02_instalacion.png)

### 3. Check the service and configuration

```powershell
Get-Service Sysmon64      # Status: Running
.\Sysmon64.exe -c         # shows the active configuration
```

![Service and configuration](images/sysmon/03_servicio_y_config.png)

### 4. Check that logs are being generated

- **Event Viewer → Applications and Services Logs → Microsoft → Windows → Sysmon → Operational**, or
- PowerShell:

  ```powershell
  Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10
  ```

![Events in PowerShell](images/sysmon/04_eventos_powershell.png)

![Events in Event Viewer](images/sysmon/05_event_viewer.png)

### 5. Generate test events

```powershell
notepad.exe
Get-Date
ping 8.8.8.8

Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 20 |
  Select-Object Id, TimeCreated, Message
```

![Generating activity](images/sysmon/06_generar_eventos.png)

![Generated events (Process Create, DNS query, File created)](images/sysmon/07_eventos_generados.png)

Repeat the same process on `WIN11-01`:

![Installing on WIN11-01](images/sysmon/08_instalacion_win11.png)

---

## Key Event IDs

You'll use these constantly when building detections in Wazuh:

| Event ID | Description | Why it matters |
| -------- | ----------- | -------------- |
| 1 | Process Creation | Malware execution, PowerShell, LOLBins |
| 3 | Network Connection | Suspicious outbound connections and C2 |
| 5 | Process Terminated | Process lifecycle analysis |
| 7 | Image Loaded | DLL injection and suspicious modules |
| 8 | CreateRemoteThread | Code injection techniques |
| 10 | Process Access | Credential theft (e.g. LSASS access) |
| 11 | File Create | Malware dropping files |
| 12–14 | Registry Events | Persistence mechanisms |
| 22 | DNS Query | Domains looked up by each process |

---

## Automatic deployment through Group Policy

The goal is to simulate how companies automatically install Sysmon on every new machine that joins the domain:

```text
ADDC01
│
├── SYSVOL
│   └── scripts\Sysmon
│       ├── Sysmon64.exe
│       ├── sysmonconfig-export.xml
│       └── install_sysmon.cmd
│
└── GPO "Workstation Baseline"
      │
      └── Startup Script ──► runs at boot on every computer in the Workstations OU
```

Steps:

1. [Create the shared folder in SYSVOL](#1-create-the-folder-in-sysvol)
2. [Create the install script](#2-create-the-install-script)
3. [Add the script to the Workstation Baseline GPO](#3-add-the-script-to-the-gpo)
4. [Update Group Policy](#4-update-group-policy)
5. [Verify the installation](#5-verify-the-installation)
6. [Test with a new workstation](#6-test-with-a-new-workstation)

### 1. Create the folder in SYSVOL

On `ADDC01`, create a `Sysmon` folder in:

```text
C:\Windows\SYSVOL\sysvol\corp.lab\scripts\
```

and copy `Sysmon64.exe` and `sysmonconfig-export.xml` into it.

SYSVOL is used because every computer in the domain can read it by default.

![Sysmon folder in SYSVOL](images/sysmon/09_carpeta_sysvol.png)

![Files in the folder](images/sysmon/10_archivos_sysvol.png)

### 2. Create the install script

Use Notepad to create `install_sysmon.cmd` (the extension **must** be `.cmd`) **in the same** `scripts\Sysmon` folder. `%~dp0` expands to the folder the script is in, which is why the `.exe` and `.xml` need to sit next to it.

**Basic version:** installs Sysmon only if it isn't already installed.

```bat
@echo off

if exist "C:\Windows\Sysmon64.exe" (
    echo Sysmon already installed.
    exit /b
)

copy "%~dp0Sysmon64.exe" "C:\Windows\Sysmon64.exe"

"C:\Windows\Sysmon64.exe" -accepteula -i "%~dp0sysmonconfig-export.xml"
```

![Basic script](images/sysmon/11_script_basico.png)

**Version that also updates the configuration** if Sysmon is already installed:

```bat
@echo off

if exist "C:\Windows\Sysmon64.exe" (
    "C:\Windows\Sysmon64.exe" -c "%~dp0sysmonconfig-export.xml"
) else (
    copy "%~dp0Sysmon64.exe" "C:\Windows\Sysmon64.exe" /Y
    "C:\Windows\Sysmon64.exe" -accepteula -i "%~dp0sysmonconfig-export.xml"
)
```

**Improved version** (the one used in the lab):

```bat
@echo off
setlocal

:: Define paths once to keep the code clean
set "SRC_EXE=%~dp0Sysmon64.exe"
set "SRC_XML=%~dp0sysmonconfig-export.xml"
set "DEST_EXE=C:\Windows\Sysmon64.exe"

:: 1. Check whether Sysmon is already installed
if exist "%DEST_EXE%" (
    echo [INFO] Sysmon found. Updating executable and configuration...
    copy "%SRC_EXE%" "%DEST_EXE%" /Y
    "%DEST_EXE%" -c "%SRC_XML%"
) else (
    echo [INFO] Sysmon not found. Starting a clean install...
    copy "%SRC_EXE%" "%DEST_EXE%" /Y
    "%DEST_EXE%" -accepteula -i "%SRC_XML%"
)

endlocal
```

What this version improves:

1. **Variables:** paths are defined once at the top, so the code is shorter and easier to change.
2. **Messages (`echo`):** show which branch the script took.
3. **`setlocal` / `endlocal`:** variables don't leak into other CMD windows.

![Improved script](images/sysmon/12_script_mejorado.png)

> ⚠️ **Don't use `pause` in a startup script.** GPO startup scripts run with no visible window: a `pause` leaves the script waiting for a keypress nobody will make and delays boot until the timeout. Only use it when testing the script by hand.
>
> 💡 `copy ... /Y` over `C:\Windows\Sysmon64.exe` can fail while the service is running (the file is in use). To upgrade the Sysmon **version** you need to uninstall (`Sysmon64.exe -u`) and reinstall; to change only the **configuration**, `-c` is enough.

### 3. Add the script to the GPO

1. In **Group Policy Management**, edit the `Workstation Baseline` GPO.
2. Go to **Computer Configuration → Policies → Windows Settings → Scripts (Startup/Shutdown) → Startup** and double-click it.
3. Click **Add...** → **Browse...**

   ![Add a script](images/sysmon/13_agregar_script.png)

4. Select the script from the SYSVOL folder. The path should be:

   ```text
   \\corp.lab\SYSVOL\corp.lab\scripts\Sysmon\install_sysmon.cmd
   ```

   ![Script path](images/sysmon/15_ruta_script.png)

> 💡 The **Show Files** button opens the GPO's own script folder (`...\Machine\Scripts\Startup`). You don't need to copy the script there: if it ran from that folder, `%~dp0` would point to it and wouldn't find `Sysmon64.exe` or the `.xml`. What matters is that the configured path is `scripts\Sysmon`.
>
> ![The GPO's Startup folder](images/sysmon/14_carpeta_startup_gpo.png)

### 4. Update Group Policy

On `WIN11-01`:

```cmd
gpupdate /force
shutdown /r /t 0
```

Startup scripts run at boot, which is why a reboot is needed.

### 5. Verify the installation

```powershell
Get-Service Sysmon64          # Status: Running
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10
```

![Service on WIN11-01](images/sysmon/16_servicio_win11_gpo.png)

![Events on WIN11-01](images/sysmon/17_eventos_win11_gpo.png)

### 6. Test with a new workstation

1. Build another workstation (`WIN11-02`) and join it to the domain.
2. Move it from `Computers` to the `Workstations` OU.

   ![WIN11-02 in Computers](images/sysmon/18_win11_02_computers.png)

   ![WIN11-02 in Workstations](images/sysmon/19_win11_02_workstations.png)

3. Reboot it.
4. Verify as in the previous step.

   ![Sysmon installed automatically on WIN11-02](images/sysmon/20_win11_02_sysmon.png)

✅ The startup script installs Sysmon automatically, without logging on or copying files by hand.

---

## Next steps

- [Build Wazuh and install the agents](WazuhServer_Checho.md)
- Verify telemetry
- Harden the DC further
- Build the attacker VM and generate detections
- Integrate Linux with AD
- Add AD CS

[⬅️ Previous: Security GPOs](SecurityBaselineGPOs_Checho.md) · [Back to README](README.md) · [Next: Wazuh ➡️](WazuhServer_Checho.md)
