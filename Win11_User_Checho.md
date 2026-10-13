# 4. Workstation Windows 11 unida al dominio

[⬅️ Anterior: Active Directory](ActiveDirectory_Checho.md) · [Volver al README](README.md) · [Siguiente: GPOs de seguridad ➡️](SecurityBaselineGPOs_Checho.md)

Instalación de la primera estación de trabajo de usuario (`WIN11-01`) y su unión al dominio `corp.lab`.

## Especificaciones

| Parámetro | Valor |
| --------- | ----- |
| Hostname | `WIN11-01` |
| Sistema operativo | Windows 11 **Pro** |
| RAM / Disco | 8 GB / 64 GB |
| Red | Users – `10.0.10.0/24` (interfaz LAN en pfSense) |
| IP / Gateway | `10.0.10.110` / `10.0.10.1` |
| DNS | `10.0.20.10` (ADDC01) |

---

## Instalación

1. Descargar la ISO oficial: <https://www.microsoft.com/es-es/software-download/windows11>.
2. Crear la VM en VMware conectada a la red de Users.
3. Instalar la edición **Pro**: la edición Home no puede unirse a un dominio.
4. Cuando pregunte el tipo de configuración, elegir **Set up for work or school** y luego la opción de **unirse a un dominio** en lugar de usar una cuenta Microsoft.

   Si esa opción no aparece, presionar `Shift + F10` para abrir una terminal y ejecutar:

   ```cmd
   OOBE\BYPASSNRO
   ```

   El equipo se reinicia; al volver a elegir *work or school* aparecerá la opción de dominio.

5. Crear una cuenta local y terminar la instalación.
6. Revisar **Settings → Windows Update** e instalar las actualizaciones pendientes.

---

## Configuración

1. [Configurar la red](#1-configurar-la-red)
2. [Verificar conectividad](#2-verificar-conectividad)
3. [Renombrar la workstation](#3-renombrar-la-workstation)
4. [Unir al dominio](#4-unir-al-dominio)
5. [Iniciar sesión con un usuario del dominio](#5-iniciar-sesión-con-un-usuario-del-dominio)
6. [Mover el equipo a la OU Workstations](#6-mover-el-equipo-a-la-ou-workstations)
7. [Verificar la funcionalidad de AD](#7-verificar-la-funcionalidad-de-ad)
8. [Tomar un snapshot](#8-tomar-un-snapshot)

### 1. Configurar la red

**Control Panel → Network and Sharing Center → Change adapter settings** → propiedades del adaptador → doble clic en **IPv4**:

| Campo | Valor |
| ----- | ----- |
| IP Address | `10.0.10.110` |
| Subnet Mask | `255.255.255.0` |
| Gateway | `10.0.10.1` |
| DNS Server | `10.0.20.10` |

> ⚠️ El DNS **debe** ser el del controlador de dominio: Active Directory depende de DNS para que los equipos encuentren el dominio.

![Configuración IPv4](images/win11/01_configuracion_ip.png)

![ipconfig](images/win11/02_ipconfig.png)

### 2. Verificar conectividad

En PowerShell:

```powershell
ping 10.0.10.1          # conectividad con pfSense
ping 10.0.20.10         # conectividad con ADDC01
nslookup addc01.corp.lab
nslookup corp.lab
```

`nslookup addc01.corp.lab` debe devolver:

```text
Name:    addc01.corp.lab
Address: 10.0.20.10
```

y `nslookup corp.lab` debe resolver.

> En este punto la LAN de pfSense solo tiene las reglas por defecto (todo permitido), así que estas pruebas deberían funcionar sin cambios en el firewall.

![Ping al gateway](images/win11/03_ping_gateway.png)

![Ping al controlador de dominio](images/win11/04_ping_dc.png)

### 3. Renombrar la workstation

En PowerShell como administrador:

```powershell
Rename-Computer -NewName "WIN11-01" -Restart
```

![Renombrar el equipo](images/win11/05_renombrar.png)

### 4. Unir al dominio

**Settings → System → About → Domain or workgroup → Change** → elegir **Domain** y escribir `corp.lab`.

![Unir al dominio corp.lab](images/win11/06_unir_dominio.png)

Pedirá credenciales de una cuenta con permisos en el dominio: usar `sergio.admin`. Debe aparecer *Welcome to the corp.lab domain* y luego pedir un reinicio.

![Bienvenida al dominio](images/win11/07_bienvenida_dominio.png)

![Reinicio requerido](images/win11/08_reinicio.png)

### 5. Iniciar sesión con un usuario del dominio

En la pantalla de inicio, elegir **Other user** e ingresar la cuenta del dominio (`sergio` o `sergio@corp.lab`).

![Inicio de sesión como otro usuario](images/win11/09_login_otro_usuario.png)

Como la cuenta se creó con *User must change password at next logon*, Windows pide cambiar la contraseña. Después aparece la bienvenida como si fuera una instalación nueva.

![Cambio de contraseña obligatorio](images/win11/10_cambio_contrasena_requerido.png)

![Sesión iniciada con la cuenta del dominio](images/win11/11_sesion_dominio.png)

### 6. Mover el equipo a la OU Workstations

En `ADDC01`: **Server Manager → Tools → Active Directory Users and Computers**. El equipo `WIN11-01` aparece en el container `Computers`:

![WIN11-01 en Computers](images/win11/12_equipo_en_computers.png)

Moverlo a **Corp → Workstations**, confirmar el aviso con *Yes* y verificar que quedó en la OU correcta (así recibirá las GPO de workstations):

![WIN11-01 en la OU Workstations](images/win11/13_equipo_en_workstations.png)

### 7. Verificar la funcionalidad de AD

En `WIN11-01` (cmd):

```cmd
whoami                 :: corp\sergio
echo %logonserver%     :: \\ADDC01
gpupdate /force        :: debe completarse sin errores
gpresult /r            :: muestra las políticas recibidas del dominio
```

![Verificación de AD](images/win11/14_verificacion_ad.png)

### 8. Tomar un snapshot

📸 **"WIN11-01 - Joined to Domain"**.

[⬅️ Anterior: Active Directory](ActiveDirectory_Checho.md) · [Volver al README](README.md) · [Siguiente: GPOs de seguridad ➡️](SecurityBaselineGPOs_Checho.md)
