# 3. Active Directory (Windows Server 2022)

[⬅️ Anterior: Bastion Host](DebianBastionHost_Checho.md) · [Volver al README](README.md) · [Siguiente: Workstation Windows 11 ➡️](Win11_User_Checho.md)

Instalación del controlador de dominio `ADDC01` con los roles AD DS y DNS, creación del bosque `corp.lab`, de las Unidades Organizativas (OU) y de los primeros usuarios.

> 🎥 Video guía de referencia: <https://www.youtube.com/watch?v=1XeDht_B-bA>

## Contenido

- [Especificaciones](#especificaciones)
- [Instalación de Windows Server](#instalación-de-windows-server)
- [Red y acceso a Internet](#red-y-acceso-a-internet)
- [1. Instalar actualizaciones](#1-instalar-actualizaciones)
- [2. Instalar AD DS y DNS](#2-instalar-ad-ds-y-dns)
- [3. Promover el servidor a controlador de dominio](#3-promover-el-servidor-a-controlador-de-dominio)
- [4. Verificar que AD funciona](#4-verificar-que-ad-funciona)
- [5. Ajustar el DNS](#5-ajustar-el-dns)
- [6. Crear las Unidades Organizativas](#6-crear-las-unidades-organizativas)
- [7. Crear usuarios](#7-crear-usuarios)

## Especificaciones

| Parámetro | Valor |
| --------- | ----- |
| Hostname | `ADDC01` |
| Sistema operativo | Windows Server 2022 Standard (Desktop Experience) |
| RAM / Disco | 8 GB / 80 GB |
| Red | Servers – `10.0.20.0/24` (interfaz OPT1 en pfSense) |
| IP / Gateway | `10.0.20.10` / `10.0.20.1` |
| Dominio | `corp.lab` (NetBIOS: `CORP`) |

---

## Instalación de Windows Server

1. Descargar la ISO desde el sitio oficial de Microsoft.
2. Crear la VM **sin** asignar la ISO en el asistente ("I will install the operating system later"), para evitar el *Easy Install* de VMware, que a veces da problemas con esta versión.
3. Asignar 8 GB de RAM y 80 GB de disco, montar la ISO en la unidad de CD y conectar la red a la subred de Servers.
4. Instalar la edición **Windows Server 2022 Standard (Desktop Experience)** para tener escritorio y no solo consola.
5. En el tipo de instalación elegir **Custom: Install Microsoft Server Operating System only (advanced)**: un controlador de dominio debe partir de un sistema limpio (la otra opción es para actualizar un Windows existente).

### Renombrar el equipo

**Search → This PC → Properties → Rename this PC** → `ADDC01`. Reiniciar.

---

## Red y acceso a Internet

Configurar IP estática `10.0.20.10`, gateway `10.0.20.1` y, **temporalmente**, DNS `8.8.8.8`:

![IP estática con DNS temporal](images/ad/01_ip_estatica_dns_temporal.png)

En pfSense (**Firewall → Rules → OPT1**) crear reglas para que la subred de Servers pueda salir a Internet a instalar roles y actualizaciones:

| Acción | Protocolo | Origen | Destino | Puerto |
| ------ | --------- | ------ | ------- | ------ |
| Pass | ICMP | OPT1 net | any | — (para probar con `ping`) |
| Pass | TCP | OPT1 net | any | 80 (HTTP) |
| Pass | TCP | OPT1 net | any | 443 (HTTPS) |
| Pass | TCP/UDP | OPT1 net | any | 53 (DNS) |

![Reglas de salida en OPT1](images/ad/02_reglas_opt1_salida.png)

Probar desde `ADDC01`:

```powershell
ping 10.0.20.1
ping 8.8.8.8
nslookup microsoft.com
```

---

## 1. Instalar actualizaciones

**Settings → Windows Update → Check for updates**. Repetir (y reiniciar las veces que haga falta) hasta que aparezca *You're up to date*.

![Windows Update al día](images/ad/03_windows_update.png)

---

## 2. Instalar AD DS y DNS

1. En **Server Manager → Manage → Add Roles and Features**, clic en *Next*.
2. Elegir **Role-based or feature-based installation**.

   ![Tipo de instalación](images/ad/04_tipo_instalacion.png)

3. Seleccionar el servidor `ADDC01`.

   ![Servidor de destino](images/ad/05_servidor_destino.png)

4. Marcar **Active Directory Domain Services** y **DNS Server**. Cada vez que se marca un rol aparece una ventana: clic en **Add Features**.

   ![Rol AD DS](images/ad/06_rol_adds.png)

   ![Features requeridas por AD DS](images/ad/07_features_adds.png)

   ![Features requeridas por DNS](images/ad/08_features_dns.png)

   ![Ambos roles seleccionados](images/ad/09_roles_seleccionados.png)

5. Continuar con *Next* hasta la confirmación y clic en **Install**.

   ![Confirmación de la instalación](images/ad/10_confirmar_instalacion.png)

   ![Instalación completada](images/ad/11_instalacion_completa.png)

---

## 3. Promover el servidor a controlador de dominio

1. Clic en la bandera de notificaciones de Server Manager → **Promote this server to a domain controller**.

   ![Promover a controlador de dominio](images/ad/12_promover_dc.png)

2. Elegir **Add a new forest** con *Root domain name* `corp.lab`.

   ![Nuevo bosque corp.lab](images/ad/13_nuevo_bosque.png)

3. Mantener marcados **DNS Server** y **Global Catalog (GC)** y definir una contraseña robusta de **DSRM** (*Directory Services Restore Mode*).
4. Dejar por defecto *DNS Options*, *NetBIOS Name* y *Paths*.

   ![Rutas de la base de datos, logs y SYSVOL](images/ad/14_rutas_ntds_sysvol.png)

   > 🔎 **Dato de seguridad:** en *Paths* se ve dónde queda `NTDS.dit`, la base de datos de Active Directory (incluye los hashes de las contraseñas del dominio). Es un objetivo prioritario para los atacantes: cualquier acceso o copia sospechosa de este archivo indica que probablemente todo el dominio está comprometido.

5. Revisar los prerrequisitos y clic en **Install**. El servidor se reinicia automáticamente.

   ![Verificación de prerrequisitos](images/ad/15_prerrequisitos.png)

---

## 4. Verificar que AD funciona

Tras el reinicio, el inicio de sesión aparece como `CORP\Administrator`, indicando que el equipo ya está en el dominio `CORP`:

![Inicio de sesión como CORP\Administrator](images/ad/16_login_corp_administrator.png)

En PowerShell:

```powershell
Get-ADDomain
```

Si devuelve la información de `corp.lab`, AD está funcionando.

![Salida de Get-ADDomain](images/ad/17_get_addomain.png)

---

## 5. Ajustar el DNS

Cambiar el DNS preferido a `10.0.20.10` (si no cambió automáticamente). Puede que Windows lo haya puesto en `127.0.0.1`; en este laboratorio es preferible la IP explícita porque facilita el troubleshooting de DNS, replicación de AD y errores de clientes, al saber exactamente qué servidor se está usando.

![DNS preferido 10.0.20.10](images/ad/18_dns_preferido.png)

Verificar en PowerShell como administrador:

```powershell
ipconfig /all          # DNS Servers debe incluir 10.0.20.10
```

![ipconfig /all](images/ad/19_ipconfig_all.png)

```powershell
nslookup addc01.corp.lab
```

Resultado esperado:

```text
Name:    addc01.corp.lab
Address: 10.0.20.10
```

![nslookup addc01.corp.lab](images/ad/20_nslookup_addc01.png)

Esto confirma que el DNS está instalado, que la zona `corp.lab` existe, que el DC registró su registro DNS y que el DNS resuelve el hostname del controlador de dominio.

```powershell
Get-Service DNS,NTDS     # ambos deben estar en Running
```

![Servicios DNS y NTDS](images/ad/21_servicios_dns_ntds.png)

Comprobar de nuevo la salida a Internet a través de pfSense:

```powershell
ping 10.0.20.1
ping 8.8.8.8
nslookup microsoft.com
```

![Conectividad final](images/ad/22_conectividad_final.png)

📸 Tomar un snapshot: **"DC01 - Fresh Domain Controller"**.

---

## 6. Crear las Unidades Organizativas

Antes de unir equipos al dominio se crean las OU. En **Server Manager → Tools → Active Directory Users and Computers**:

![Menú Tools → ADUC](images/ad/23_menu_aduc.png)

1. Sobre `corp.lab`, crear una OU llamada **Corp** (la OU raíz del laboratorio).

   ![Botón para crear una OU](images/ad/24_nueva_ou.png)

   ![OU Corp](images/ad/25_ou_corp.png)

2. Dentro de `Corp`, crear:

   ```text
   Corp
   ├── Users
   ├── Workstations
   ├── Servers
   ├── Admins
   ├── Service Accounts
   └── Groups
   ```

   ![Estructura de OUs](images/ad/26_estructura_ous.png)

**¿Por qué no usar las carpetas por defecto `Users` y `Computers`?** No son OU sino *containers*: no se les pueden vincular GPO y son casi siempre valores heredados (*legacy*). Las OU, en cambio, permiten aplicar Group Policy, delegar permisos y son la forma en que las empresas organizan AD. Por ejemplo:

```text
Workstations OU          Servers OU
      ↓                       ↓
Disable LLMNR            Otras políticas de seguridad
Enable Sysmon
Enable auditing
```

---

## 7. Crear usuarios

Se crean dos cuentas separadas para practicar la separación de privilegios: un usuario normal (`sergio`) y uno administrativo (`sergio.admin`).

### Usuario normal

En la OU **Corp → Users** → *Create a new user*:

![Crear un usuario nuevo](images/ad/27_nuevo_usuario.png)

![Usuario sergio](images/ad/28_usuario_sergio.png)

Marcar **User must change password at next logon**.

![Usuario creado](images/ad/29_usuario_creado.png)

### Usuario administrador

En la OU **Corp → Admins**, crear `sergio.admin`:

![Usuario sergio.admin](images/ad/30_usuario_sergio_admin.png)

Luego: clic derecho sobre el usuario → **Add to a group** → escribir `Domain Admins` → **Check Names** → **OK**.

![Agregar a Domain Admins](images/ad/31_grupo_domain_admins.png)

![Usuario agregado al grupo](images/ad/32_grupo_agregado.png)

---

## Siguientes pasos

1. ~~Terminar el despliegue de AD~~ ✅
2. [Unir una workstation al dominio](Win11_User_Checho.md)
3. Verificar DNS
4. [Verificar Group Policy](SecurityBaselineGPOs_Checho.md)
5. Crear usuarios y grupos administrativos
6. [Desplegar Wazuh](WazuhServer_Checho.md)
7. Endurecer el controlador de dominio

[⬅️ Anterior: Bastion Host](DebianBastionHost_Checho.md) · [Volver al README](README.md) · [Siguiente: Workstation Windows 11 ➡️](Win11_User_Checho.md)
