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

Step 10: Repeat on ADDC01

Once you've verified everything on WIN11-01:

Copy the same C:\Sysmon folder to ADDC01.
Run the same installation command.
Verify the service and event log.

Keeping both systems on the same configuration makes future analysis much easier.

Next step: Deploy through Group Policy

After you've confirmed Sysmon works on both machines, we can automate deployment so that every future workstation or server that joins the domain installs Sysmon automatically at startup. This is much closer to how Sysmon is managed in enterprise environments and will make your lab easier to expand.
