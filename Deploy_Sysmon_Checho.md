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

```
@echo off

if exist "C:\Windows\Sysmon64.exe" (
    echo Sysmon already installed.
    exit /b
)

copy "%~dp0Sysmon64.exe" "C:\Windows\Sysmon64.exe"

"C:\Windows\Sysmon64.exe" -accepteula -i "%~dp0sysmonconfig-export.xml"
```
y lo guardamos como install_sysmon.cmd

Importante que la extension sea .cmd

Este script revisa si Sysmon.exe ya está instalado, y en caso de que no lo este, lo instala con la configuracion de la carpeta compartida.

Se puede usar un script parecido para actualizar la configuracion en caso de que ya este instalado.

```
@echo off

if exist "C:\Windows\Sysmon64.exe" (
    "C:\Windows\Sysmon64.exe" -c "%~dp0sysmonconfig-export.xml"
) else (
    copy "%~dp0Sysmon64.exe" "C:\Windows\Sysmon64.exe" /Y
    "C:\Windows\Sysmon64.exe" -accepteula -i "%~dp0sysmonconfig-export.xml"
)
```

O asi

```
@echo off
setlocal

:: Definir rutas para mantener el código limpio
set "ORIGEN_EXE=%~dp0Sysmon64.exe"
set "ORIGEN_XML=%~dp0sysmonconfig-export.xml"
set "DESTINO_EXE=C:\Windows\Sysmon64.exe"

:: 1. Comprobar si Sysmon ya existe en el sistema
if exist "%DESTINO_EXE%" (
    echo [INFO] Sysmon detectado. Actualizando ejecutable y configuracion...
    copy "%ORIGEN_EXE%" "%DESTINO_EXE%" /Y
    "%DESTINO_EXE%" -c "%ORIGEN_XML%"
) else (
    echo [INFO] Sysmon no detectado. Iniciando instalacion limpia...
    copy "%ORIGEN_EXE%" "%DESTINO_EXE%" /Y
    "%DESTINO_EXE%" -accepteula -i "%ORIGEN_XML%"
)

endlocal
pause

```
Este codigo mejora en estos aspectos:
  1. Uso de variables: Centraliza las rutas al inicio (ORIGEN_EXE, DESTINO_EXE, etc.) para que el código sea más corto, limpio y fácil de modificar en el futuro.
  2. Mensajes en pantalla (echo): Agrega textos informativos para que sepas exactamente qué camino tomó el script al ejecutarse.
  3. Comandos de control: setlocal y endlocal aseguran que las variables creadas no afecten a otras ventanas de CMD de tu sistema. pause evita que la ventana se cierre de golpe al terminar para que puedas ver el resultado.

------------------------------------------------------------------------------------------------------------------
3. Editar la GPO Workstation Baseline

En el Group Policy Management buscamos la Workstation Baseline que creamos anteriormente.

La editamos y vamos a Computer Configuration > Policies > Windows Settings > Scripts (Startup/Shutdown) > Startup

Doble click y damos en "Show Files"

En esta ventana se abre la carpeta de scripts de la GPO. Aqui pegamos el script que acabamos de crear

Despues de pegar el archivo, cerramos la ventana y damos click en "Add"

------------------------------------------------------------------------------------------------------------------
4. Seleccionar el script

Damos en "Browse" y nos llevara a la carpeta de scripts donde pegamos el script, aqui seleccionamos nuestro scipt y aparecera en el script name algo asi 

```
\\corp.lab\SYSVOL\corp.lab\scripts\Sysmon\install_sysmon.cmd
```
------------------------------------------------------------------------------------------------------------------
5. Actualizar Group Policy

En el equipo WIN11-01 corremos los siguientes comandos

```
gpupdate /force
shutdown /r /t 0
```

Esto reincia el computador despues de actualizar las politicas

------------------------------------------------------------------------------------------------------------------
6. Verificar instalacion

Despues de reinciar corremos los comandos en powershell

```
Get-Service Sysmon64
```
Y deberia mostrar:
```
Status : Running
```

Y el comando:
```
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 10
```
Que deberia mostrar eventos

------------------------------------------------------------------------------------------------------------------
7. Testear con nueva workstation

Finalmente montamos otra workstation y
  1. La unimos al dominio
  2. Movemos la workstation a la OU de Workstations
  3. Reiniciamos la maquina
  4. Verificamos la instalacion (como en el punto anterior)

The startup script should automatically install Sysmon without you logging on or manually copying files.

------------------------------------------------------------------------------------------------------------------
Siguientes pasos:
  - Build Wazuh.
  - Install Wazuh agents.
  - Verify telemetry.
  - Harden DC further.
  - Build attack VM.
  - Generate detections.
  - Add Linux AD integration.
  - Add AD CS.
