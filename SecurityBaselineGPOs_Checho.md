<img width="1025" height="768" alt="image" src="https://github.com/user-attachments/assets/3fb158d8-ce19-4f21-b593-b1942316df96" />Una GPO Directiva de Grupo (Group Policy Object en inglés) es una herramienta de Windows Server que permite a los administradores de TI gestionar y configurar de forma centralizada los sistemas operativos, aplicaciones y permisos de seguridad para múltiples usuarios y equipos en una red.

-----------------------------------------------------------------------
WORKSTATIONS BASELINE
-----------------------------------------------------------------------
Vamos a crear una GPO para las Workstations (Workstation Baseline)

Vamos a Server Manager > Tools > Group Policy Management

Aqui, entramos a Domains > corp.lab > Corp > Workstations

Damos click derecho a la carpeta de la OU "Workstations" y le damos click a "Create a GPO in this domain, and Link it here..."

Aqui le ponemos el nombre "Workstation Baseline"

Damos click derecho sobre la nueva GPO y damos en "Edit"

Vamos a crear las siguientes instrucciones:
  1. Disable LLMNR
  2. Enable PowerShell Logging
  3. Enable Audit Policies

-----------------------------------------------------------------------
1. Disable LLMNR

Para esta primera regla debemos estar en el "Group Policy Management Editor"

Aqui vamos a Computer Configuration > Policies > Administrative Templates > Network > DNS Client

Y buscamos la que diga "Turn Off Multicast Name Resolution"

Le damos click derecho y "Edit"

Ahi seleccionamos "Enabled".

Why disable LLMNR?
You should disable LLMNR (Link-Local Multicast Name Resolution) to prevent critical credential-harvesting attacks. When DNS fails, LLMNR exposes your network to attackers who can intercept requests, trick computers into authenticating, and steal sensitive password hashes (like NetNTLMv2).The Main RisksLLMNR Poisoning: Attackers on your local network (using tools like Responder) answer name resolution queries claiming to be the requested server.Credential Theft: The victim's machine unknowingly sends its password hash to the attacker to authenticate.Relay Attacks: Attackers can relay stolen hashes to access other servers, bypassing the need to even crack the password. - Gemini AI

-----------------------------------------------------------------------
2. Enable PowerShell Logging

Aqui vamos a Computer Configuration > Policies > Administrative Templates > Windows Components > Windows Powershell

En esta carpeta activamos "Turn on Module Logging" (En la lista de modulos escribimos * para incluir todos los comandos de powershell)
Tambien activamos "Turn on PowerShell Script Block Logging"

-----------------------------------------------------------------------
3. Enable Audit Policies

Aqui vamos a Computer Configuration > Policies > Windows Settings > Security Settings > Advanced Audit Policy > Audit Policies

Aqui activamos las siguientes politicas

Logon/Logoff
  - Audit Logon
  - Audit Logoff
  - Audit Special Logon
  - Audit Other Logon/Logoff Events
Activamos Success y Failure
These let you see:
  - User logins
  - Failed logins
  - RDP logins
  - Administrator logins


Account Logon
  - Audit Credential Validation
  - Audit Kerberos Authentication Service
  - Audit Kerberos Service Ticket Operations
  - Audit Other Account Logon Events
Activamos Success y Failure
Important for AD authentication monitoring.


Account Management
  - Audit User Account Management
  - Audit Security Group Management
  - Audit Computer Account Management
  - Audit Distribution Group Management
  - Audit Other Account Management Events
Activamos Success y Failure
Lets you detect:
  - New users
  - Deleted users
  - Password resets
  - Users added to Domain Admins


Detailed Tracking:
  - Audit Process Creation
Adicionalmente, vamos a Computer Configuration > Policies > Administrative Templates > System > Audit Process Creation
Aqui damos doble click en "Include command line in process creation events" y lo cambiamos a "enable"
This causes Event ID 4688 to include command-line arguments. (Ex. powershell.exe -enc ... instead of only powershell.exe)


Nos devolvemos a Computer Configuration > Policies > Windows Settings > Security Settings > Advanced Audit Policy > Audit Policies y activamos

Policy Change
  - Audit Audit Policy Change
  - Audit Authentication Policy Change
  - Audit Authorization Policy Change
Activamos Success y Failure
Lets you detect tampering with security settings.


Privilege Use
  - Audit Sensitive Privilege Use
Activamos Success y Failure
This helps detect abuse of powerful privileges.


Object Access (Minimal) - Don't enable everything here yet.
  - Audit File Share
  - Audit Detailed File Share
Activamos Success y Failure
Useful when you later deploy file servers.

-----------------------------------------------------------------------
Evitar estas configuraciones por ahora:
  - Audit File System
  - Audit Registry
  - Audit Kernel Object
  - Audit Handle Manipulation
  - Audit Removable Storage
  - Audit Filtering Platform Packet Drop
  - Audit Filtering Platform Connection
Pueden generar un volumen muy grande de logs y es mejor introducirlos despues
-----------------------------------------------------------------------


-----------------------------------------------------------------------
DC Baseline GPO
-----------------------------------------------------------------------
Para el controlador de dominio crearemos otra GPO Base

Para esto debemos crear una nueva OU que sera la de los Controladores de Dominio

Dentro del Server Manager vamos a Tools > Active Directory Users and Computers

Aqui entramos a corp.lab > Corp y creamos la nueva OU "Domain Controllers"

Cerramos y volvemos al Server Manager


Ahora entramos a Tools > Group Policy Management

Navegamos a Forest > Domains > corp.lab > Corp > Domain Controllers

Damos click derecho sobre la carpeta Domain Controllers y damos en "Create a GPO in this domain, and Link it here" y la llamamos "DC Baseline"



Click derecho sobre "DC Baseline" y damos en "Edit" para configurar las politicas:

Las cosas que configuraremos seran:
  1. Disable LLMNR
  2. DC-Specific Auditing
  3. PowerShell Logging
  4. Process Command Line Logging


-----------------------------------------------------------------------
1. Disable LLMNR

Misma configuracion de la Workstation Baseline

-----------------------------------------------------------------------
2. DC-Specific Auditing

Vamos a Computer Configuration > Policies > Windows Settings > Security Settings > Advanced Audit Policy > Audit Policies

Habilitamos las mismas politicas que la Workstation Baseline y configuramos estas nuevas

- Directory Service Changes (DS Access)
- Directory Service Access (DS Access)
- User Account Management (Account Management)
- Security Group Management (Account Management)

-----------------------------------------------------------------------
3. PowerShell Logging

Misma configuracion de la Workstation Baseline

-----------------------------------------------------------------------
4. Process Command Line Logging

Vamos a Computer Configuration > Administrative Templates > System > Audit Process Creation y activamos "Include command line in process creation events"

