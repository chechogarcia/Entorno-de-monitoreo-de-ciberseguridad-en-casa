Para instalar la primera vm de usuario que ira conectada al dominio toca primero descargar una iso de Windows 11 de la pagina oficial de Windows (https://www.microsoft.com/es-es/software-download/windows11)

Ya con la imagen creamos una nueva maquina virtual en VMWare en la red de users (10.0.10.0/24) con 8GB de ram y 64 de disco.

Instalamos la version Pro para que la maquina se pueda unir a un dominio (Con la version Hme no se puede).

Continuamos la instalación, y cuando pida si quiere configurar la cuenta como personal o de trabajo, escogemos de trabajo ("Set up for work or school"). Despues pedira una cuenta de Microsft, pero hay otra opción que es incluirla a un dominio. Escogemos esa opcion. 

Si no aparece oprimimos Shift + F10 para que aparzca la terminal y corremos el comando:
  OOBE\BYPASSNRO

Eso reiniciara la maquina y toca volver a configurarla para trabajo y aparecera la opcion del dominio

Creamos una cuenta con el usuario y contraseña que qeuramos y se completa la instalación.

Cuando termine de instalar entramos a configuracion y vemos si hay actualizaciones pendientes.

-----------------------------------------------------------------------------------------------

Para terminar de configurar el host debemos seguir los siguientes pasos:

1. Configurar la red
2. Verificar conectividad
3. Renombrar la Workstation
4. Entrar al dominio
5. Iniciar sesión en el dominio
6. Mover el objeto del Computador en el dominio
7. Verificar la funcionalidad de AD
8. Crear un Snapshot


-----------------------------------------------------------------------------------------------

1. Configurar la red

Entramos al panel de control, buscamos "Network and Sharing Center" y damos en "Change adapter settings"

Doble click en ipv4 y configuramos el host con la siguiente informacion:
  IP Address: 10.0.10.110
  Subnet Mask: 255.255.255.0
  Gateway: 10.0.10.1
  DNS Server: 10.0.20.10

Importante que el DNS sea 10.0.20.10 pues AD depende de DNS


2. Verificar conectividad

Desde PwerShell probar los siguientes comandos:
  ping 10.0.10.1 (conectividad con pfsense)
  ping 10.0.20.10 (conectividad con ADDC01)
  nslookup dc01.corp.lab
  nslookup corp.lab


Para nslookup dc01.corp.lab deberia mostrar:
  Name: dc01.corp.lab
  Address: 10.0.20.10


Para nslookup corp.lab deberia resolver

(Nota: En este momento no hay ninguna regla ademas de las qe son por defecto en la LAN del pfsense, por lo que el trafico hacia cualquier lado esta permitido, entonces estos pings deberian servir sin problema)



3. Renombrar la Workstation

En PowerShell como Administrador correr el comando

  Rename-Computer -NewName "WIN11-01" -Restart


4. Entrar al dominio

Entrar a Settings > System > About > Related Links > Domain or Workgroup > Change

Elegimos "Domain" y escribimos "corp.lab"

Damos ok y nos pedirá que ingresemos una cuenta que tenga acceso a este dominio. Usamos la cuenta de administrador que creamos anteriormente en el Directorio activo (sergio.admin)

Debera aparecer un aviso con "Welcome to the corp.lab domain" y toca reiniciar.



5. Iniciar sesión en el dominio

Despues de reiniciar, en vez de iniciar en el usuario local, damos en "Other User" e ingresamos la cuenta de usuario del directorio activo (sergio o sergio@corp.lab)

Por la politica de contraseña que le pusimos al crear el usuario, va a pedir que se cambie la contraseña, lo cambiamos y damos enter. Esto nos dara la bienvenida como si acabaramos de instalar windows.



6. Mover el objeto del Computador en el dominio

Nos metemos al directorio activo y en "Server Manager" vamos a Tools > Active Directory Users and Computers

Aqui buscamos el computador WIN11-01 (Seguramente esta en la carpeta "Computers") y lo movemos a la carpeta Corp-Workstations. Aparece un aviso, le damos Yes y verificamos que se haya movido correctamente a la carpeta de Workstations



7. Verificar la funcionalidad de AD

En la maquina de windows en en cmd o powershell corremos "whoami" y deberia aparecer "corp\sergio"

Despues escribir "echo %logonserver%" y deberia dar "\\ADDC01"

Despues "gpupdate /force" y deberia completarse satisfactoriamente.



8. Crear un Snapshot
Tomar un snapshot de la maquina Windows y llamarlo "WIN11-01 - Joined to Domain"
