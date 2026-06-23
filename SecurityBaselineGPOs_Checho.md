Una GPO Directiva de Grupo (Group Policy Object en inglés) es una herramienta de Windows Server que permite a los administradores de TI gestionar y configurar de forma centralizada los sistemas operativos, aplicaciones y permisos de seguridad para múltiples usuarios y equipos en una red.

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
