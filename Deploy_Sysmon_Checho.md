El siguiente paso es desplegar Sysmon para tener mejor telemetria y poder incluirla en Wazuh cuando lo montemos

Para esto abrimos ADDC01 y descargamos Sysmon y el archivo de configuracion de nuestra preferencia

En mi caso usaré el archivo SwiftOnSecurity Sysmon Config

Sysmon: https://learn.microsoft.com/es-es/sysinternals/downloads/sysmon
Config: https://github.com/SwiftOnSecurity/sysmon-config

Otra configuracion recomendada por Microsoft es la config Sysmon-modular: https://github.com/olafhartong/sysmon-modular
Estas configuraciones se sacaron de: https://learn.microsoft.com/es-es/windows/security/operating-system-security/sysmon/how-to-enable-sysmon

Sysmon es un zip con 4 archivos (Sysmon.exe, Sysmon64.exe, Sysmon64a.exe, Eula.txt) que descomprimimos y la configuracion es un archivo .xml (sysmonconfig-export.xml)

Ahora creamos una carpeta C:\Sysmon y copiamos los archivos Sysmon64.exe (de la carpeta descomprimida) y sysmonconfig-export.xml

Abrimos Powershell como administrador y entramos a la carpeta C:\Sysmon (cd C:\Sysmon)

Aqui corremos el comando: .\Sysmon64.exe -accepteula -i .\sysmonconfig-export.xml
Deberia aparecer un texto asi:
  Loading configuration file...
  Configuration validated.
  Sysmon64 installed.
  SysmonDrv installed.
  Starting Sysmon...
  Sysmon started.

Ahora, para verificar que se haya instalado correctamente corremos el comando: Get-Service Sysmon64

Deberia aparecer: Status : Running

Para verificar la configuracion corremos: .\Sysmon64.exe -c
Esto muestra la configuracion activa

Verificamos que se esten generando Logs
  1. En el "Event Viewer" vamos a Applications and Services Logs > Microsoft > Windows > Sysmon > Operational
  2. Correr el siguiente comando en Powershell: Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10


Por ultimo generamos eventos que se deberian loggear y verificamos los ultimos eventos de Sysmon

  notepad.exe
  Get-Date
  ping 8.8.8.8

Y para verificar en powershell:
  Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 20 | Select-Object Id, TimeCreated, Message

Este mismo proceso lo usamos para desplegar sysmon en WIN11-01

------------------------------------------------------------------------------------------------------------------
Learn the important Event IDs
You'll use these frequently when creating detections in Wazuh.

Event ID	Description	Why it matters
1	Process Creation	Detect malware execution, PowerShell, LOLBins
3	Network Connection	Detect outbound C2 and suspicious connections
5	Process Terminated	Useful for process lifecycle analysis
7	Image Loaded	DLL injection and suspicious modules
8	CreateRemoteThread	Code injection techniques
10	Process Access	Credential theft (e.g., LSASS access)
11	File Create	Malware dropping files
12–14	Registry Events	Persistence mechanisms
22	DNS Query	Domain lookups by processes


------------------------------------------------------------------------------------------------------------------
Siguiente paso, Desplegar por medio de Group Policy

La idea es simular el despliegue automatico de las empresas para sysmon en nuevos equipos que entren al dominio

El deployment funcionara asi

ADDC01
│
├── SYSVOL
│   └── Sysmon
│       ├── Sysmon64.exe
│       ├── sysmonconfig.xml
│       └── install_sysmon.cmd
│
└── GPO
      │
      └── Startup Script
              │
              ▼
Every domain computer


Los pasos a seguir son:
  1. Crear una carpeta compartida en SYSVOL
  2. Crear un script de instalacion
  3. Editar la GPO Workstation Baseline
  4. Seleccionar el script
  5. Actualizar Group Policy
  6. Verificar instalacion
  7. Testear con nueva workstation


------------------------------------------------------------------------------------------------------------------
1. Crear una carpeta compartida en SYSVOL

En ADDC01 vamos a C:\Windows\SYSVOL\sysvol\corp.lab\scripts y creamos una carpeta "Sysmon"

Aqui copiamos los archivos Sysmon64.exe, sysmonconfig-export.xml

Usamos la carpeta SYSVOL porque esta es visible y legible para todos los equipos dentro del dominio por default

------------------------------------------------------------------------------------------------------------------
2. Crear un script de instalacion

Abrimos notepad y escribimos el script

´´´
@echo off

if exist "C:\Windows\Sysmon64.exe" (
    echo Sysmon already installed.
    exit /b
)

copy "%~dp0Sysmon64.exe" "C:\Windows\Sysmon64.exe"

"C:\Windows\Sysmon64.exe" -accepteula -i "%~dp0sysmonconfig.xml"
´´´
