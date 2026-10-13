# 7. Servidor Wazuh (SIEM)

[⬅️ Anterior: Sysmon](Deploy_Sysmon_Checho.md) · [Volver al README](README.md)

**Wazuh** centraliza los logs y la telemetría de los equipos del laboratorio (eventos de Windows, Sysmon, logs de Linux) y genera alertas. Se instala en la subred de Security con sus tres componentes (indexer, server y dashboard) en una sola VM.

## Contenido

- [Especificaciones](#especificaciones)
- [Reglas de pfSense para la subred Security](#reglas-de-pfsense-para-la-subred-security)
- [Instalación de Ubuntu Server](#instalación-de-ubuntu-server)
- [Hardening del servidor](#hardening-del-servidor)
- [Reglas de pfSense para Wazuh](#reglas-de-pfsense-para-wazuh)
- [Instalar Wazuh](#instalar-wazuh)
- [Instalar agentes](#instalar-agentes)

## Especificaciones

| Parámetro | Valor |
| --------- | ----- |
| Hostname | `WAZUH01` |
| Sistema operativo | Ubuntu Server 24.04 LTS |
| vCPU | 4 |
| RAM | 8 GB (12 GB recomendado) |
| Disco | 50 GB |
| Red | Security – `10.0.50.0/27` (interfaz OPT4 en pfSense) |
| IP | `10.0.50.10` |
| Gateway | `10.0.50.1` |
| DNS | `10.0.20.10` (ADDC01) |

### Puertos de Wazuh

| Puerto | Protocolo | Uso | Origen |
| ------ | --------- | --- | ------ |
| 443 | TCP | Dashboard web | Management |
| 22 | TCP | SSH de administración | Management |
| 1514 | TCP | Comunicación de los agentes | Users, Servers, Management |
| 1515 | TCP | Registro (*enrollment*) de agentes | Users, Servers, Management |

> Los agentes inician la conexión hacia Wazuh; el servidor no necesita abrir sesiones hacia los endpoints.

---

## Reglas de pfSense para la subred Security

Antes de instalar, configurar en pfSense las reglas de la interfaz de Security.

### Regla 1 – Salida a Internet (HTTP/HTTPS)

Para actualizaciones de Ubuntu, paquetes de Wazuh y, a futuro, descargas de *threat intelligence*.

| Acción | Interfaz | Protocolo | Origen | Destino | Puerto |
| ------ | -------- | --------- | ------ | ------- | ------ |
| Pass | Security | IPv4 TCP | Security net | any | 443 |
| Pass | Security | IPv4 TCP | Security net | any | 80 |

### Regla 2 – DNS solo hacia el controlador de dominio

En lugar de permitir DNS hacia cualquier destino, permitir solo consultas al DNS integrado con AD:

| Acción | Origen | Destino | Puerto |
| ------ | ------ | ------- | ------ |
| Pass | Security net | `10.0.20.10` | 53 TCP/UDP |

### Regla 3 – NTP

Usar pfSense (o el DC) como fuente de hora:

| Acción | Origen | Destino | Puerto |
| ------ | ------ | ------- | ------ |
| Pass | Security net | `10.0.50.1` | 123 UDP |

### Regla 4 – Bloquear tráfico hacia redes internas (opcional)

Cuando el laboratorio esté completo, bloquear explícitamente el tráfico innecesario desde Security hacia otras redes internas:

| Acción | Origen | Destino |
| ------ | ------ | ------- |
| Block | Security net | Users net |
| Block | Security net | Servers net |
| Block | Security net | DMZ net |

---

## Instalación de Ubuntu Server

- Durante la instalación, marcar **Install OpenSSH server**. En la pantalla de *Featured server snaps* no seleccionar ninguno.
- Mientras dura la instalación, mantener en pfSense una regla temporal que permita a esta subred salir a Internet.

Después de instalar, actualizar y reiniciar:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

### IP estática con Netplan

1. Ver el nombre de la interfaz (normalmente `eth0`, `enp0s3` o `ens33`):

   ```bash
   ip a
   ```

2. Ubicar el archivo de Netplan (algo como `00-installer-config.yaml`, `50-cloud-init.yaml` o `01-netcfg.yaml`):

   ```bash
   ls /etc/netplan/
   sudo nano /etc/netplan/<archivo>.yaml
   ```

3. Configurar la IP estática:

   ```yaml
   network:
     version: 2
     renderer: networkd
     ethernets:
       ens33:            # reemplazar por el nombre de la interfaz
         dhcp4: false
         addresses:
           - 10.0.50.10/27
         routes:
           - to: default
             via: 10.0.50.1
         nameservers:
           addresses:
             - 10.0.20.10
   ```

   > ⚠️ YAML no admite tabulaciones: usar solo espacios y respetar la indentación y los guiones `-` exactamente como en el ejemplo.

   ![Configuración de Netplan](images/wazuh/01_netplan.png)

4. Restringir los permisos del archivo (Netplan advierte si otros usuarios pueden leerlo) y aplicar:

   ```bash
   sudo chown root:root /etc/netplan/<archivo>.yaml
   sudo chmod 600 /etc/netplan/<archivo>.yaml
   sudo netplan apply
   ```

   ![Permisos del archivo de Netplan](images/wazuh/02_permisos_netplan.png)

---

## Hardening del servidor

### 1. Crear un usuario administrativo

```bash
sudo adduser sergioadmin
# -a = append (agregar sin quitar otros grupos), -G = grupo
sudo usermod -aG sudo sergioadmin
```

Probar:

```bash
su - sergioadmin
sudo whoami      # debe responder: root
```

### 2. Configurar SSH

```bash
sudo nano /etc/ssh/sshd_config
```

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
UsePAM yes
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
```

> ⚠️ Con `PasswordAuthentication no`, copiar antes la llave pública del administrador (`ssh-copy-id sergioadmin@10.0.50.10`) para no quedarse sin acceso.

```bash
sudo sshd -t
sudo systemctl restart ssh
sudo systemctl enable ssh
sudo systemctl status ssh
```

### 3. Instalar fail2ban

```bash
sudo apt install fail2ban -y
sudo systemctl status fail2ban
```

![SSH y fail2ban activos](images/wazuh/03_ssh_fail2ban.png)

### 4. Configurar ufw

Aunque el firewall principal sea pfSense, un firewall local agrega una capa más de protección:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.0.40.0/28 to any port 22    # SSH solo desde Management
```

![Reglas iniciales de ufw](images/wazuh/04_ufw.png)

El resto de reglas de `ufw` se agregan después de instalar Wazuh.

---

## Reglas de pfSense para Wazuh

Cada regla se crea en la **interfaz de origen** del tráfico.

| Origen (interfaz) | Destino | Puertos |
| ----------------- | ------- | ------- |
| Management (OPT3) | Security | 22 (SSH), 443 (Dashboard), 1514 TCP (agente del bastion), 1515 TCP (enrollment del bastion) |
| Users (LAN) | Security | 1514 TCP (agente), 1515 TCP (enrollment) |
| Servers (OPT1) | Security | 1514 TCP (agente), 1515 TCP (enrollment) |

![Reglas en Management (OPT3)](images/wazuh/05_reglas_management.png)

![Reglas en Users (LAN)](images/wazuh/06_reglas_users.png)

![Reglas en Servers (OPT1)](images/wazuh/07_reglas_servers.png)

---

## Instalar Wazuh

Seguir la guía oficial: <https://documentation.wazuh.com/current/installation-guide/index.html>

La guía permite instalar el **indexer**, el **server** y el **dashboard** por separado; en este laboratorio se usó la instalación **paso a paso** (*step-by-step*) con los tres componentes en `WAZUH01`.

> 🔐 Al terminar, el instalador muestra en la terminal credenciales generadas aleatoriamente. **Guardarlas en un gestor de contraseñas**: son las que se usan para entrar al dashboard. Nunca subirlas al repositorio ni dejarlas en capturas de pantalla.

Verificar los servicios:

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

### Reglas de ufw para Wazuh

```bash
# Dashboard y SSH desde Management
sudo ufw allow from 10.0.40.0/28 to any port 443 proto tcp
sudo ufw allow from 10.0.40.0/28 to any port 22 proto tcp

# Agentes (1514) desde Users, Servers y Management
sudo ufw allow from 10.0.10.0/24 to any port 1514 proto tcp
sudo ufw allow from 10.0.20.0/24 to any port 1514 proto tcp
sudo ufw allow from 10.0.40.0/28 to any port 1514 proto tcp

# Enrollment (1515) desde Users, Servers y Management
sudo ufw allow from 10.0.10.0/24 to any port 1515 proto tcp
sudo ufw allow from 10.0.20.0/24 to any port 1515 proto tcp
sudo ufw allow from 10.0.40.0/28 to any port 1515 proto tcp

sudo ufw enable
sudo ufw status numbered
```

### Cambiar las contraseñas de los usuarios de la API

Las cuentas `wazuh` y `wazuh-wui` de la API traen contraseñas generadas por el instalador; se pueden cambiar por unas propias:

```bash
sudo /var/ossec/framework/python/bin/python3 << 'EOF'
from wazuh.rbac.orm import AuthenticationManager
with AuthenticationManager() as auth:
    print("wazuh:", auth.update_user(user_id=1, password='<NUEVA_CONTRASEÑA>'))
    print("wazuh-wui:", auth.update_user(user_id=2, password='<NUEVA_CONTRASEÑA>'))
EOF
sudo systemctl restart wazuh-manager
```

Probar que la API acepta la nueva contraseña (debe devolver un token JWT):

```bash
curl -u wazuh:'<NUEVA_CONTRASEÑA>' -k -X POST \
  "https://localhost:55000/security/user/authenticate?raw=true"
```

Actualizar la contraseña de `wazuh-wui` en la configuración del dashboard, en la sección `hosts:` de `/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml`, y reiniciar el servicio:

```bash
sudo systemctl restart wazuh-dashboard
```

> 🔐 Usar contraseñas únicas y no reutilizarlas en otros servicios. No ejecutar este comando en una terminal que se vaya a capturar o grabar.

### Probar el dashboard

Desde la Admin VM, abrir `https://10.0.50.10` en el navegador e iniciar sesión.

![Dashboard de Wazuh sin agentes](images/wazuh/08_dashboard.png)

---

## Instalar agentes

Con el dashboard funcionando, instalar los agentes en los equipos para que reporten a Wazuh:

| Sistema | Guía |
| ------- | ---- |
| Windows | <https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html> |
| Linux | <https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html> |

Equipos con agente:

| Agente | IP | Sistema |
| ------ | -- | ------- |
| ADDC01 | `10.0.20.10` | Windows Server 2022 |
| WIN11-01 | `10.0.10.110` | Windows 11 Pro |
| bastion01 | `10.0.40.10` | Debian 13 |

![Los tres agentes activos en Wazuh](images/wazuh/09_agentes_activos.png)

---

## Siguientes pasos

- Enviar los eventos de Sysmon a Wazuh y verificar la telemetría
- Endurecer más el DC
- Montar la VM de ataque y generar detecciones
- Integrar Linux con AD
- Agregar AD CS

[⬅️ Anterior: Sysmon](Deploy_Sysmon_Checho.md) · [Volver al README](README.md)
