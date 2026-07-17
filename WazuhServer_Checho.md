Antes de iniciar la instalacion modificamos las reglas de pfsense en la interfaz de la sebred de seguridad:

## Rule 1 – Allow Security subnet to the Internet

Purpose:

Ubuntu updates
Wazuh package updates
Threat intelligence downloads (future)
```
Action: Pass
Interface: Security
Protocol: IPv4 TCP/UDP
Source: Security net
Destination: any
Port 443
```
```
Action: Pass
Interface: Security
Protocol: IPv4 TCP/UDP
Source: Security net
Destination: any
Port 80
```

## Rule 2 – Allow DNS to the Domain Controller

Instead of allowing all outbound DNS, explicitly allow queries to your AD-integrated DNS server:
```
Action: Pass
Source: Security net
Destination: 10.0.20.10
Ports:
53 TCP
53 UDP
```

## Rule 3 – Allow NTP

Either use pfSense or your Domain Controller as the time source.
```
Source: Security net
Destination:
10.0.50.1
UDP 123
```

## Rule 4 – Block RFC1918 (optional)

Once the lab is complete, you can explicitly block unnecessary traffic from the Security subnet to internal networks.

Example:

Block
```
Security net
Destination:
Users net
```
Block
```
Security net
Destination:
Servers net
```
Block
```
Security net
Destination:
DMZ
```
-----------------------------------------------------------------------------
# Instalacion de Wazuh

Para el Servidor Wazuh usaremos Ubuntu Server 24.02LTS

Lo configuramos con la siguiente información
| Setting | Value                  |
| ------- | ---------------------- |
| Name    | WAZUH01                |
| vCPU    | 4                      |
| RAM     | 8 GB (12 GB preferred) |
| Disk    | 50 GB                  |
| Network | 10.0.50.0/27           |
| IP      | 10.0.50.10             |
| Gateway | 10.0.50.1              |
| DNS     | 10.0.20.10             |

En la instalacion escogemos que se instale OpenSSH Server

Nota: Mientras se instala todo en el firewall se aplica una regla que permita el acceso a internet por parte de esta subred

Despues de la instalacion, actualizamos paquetes y reiniciamos la máquina

```
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

ara configurar la informacion de red (IP, Gateway, DNS) toca editar el archivo de configuracion de Netplan

Para esto miramos cual es el nombre de la interfaz de red que esta conectada a la maquina con el comando 
```
ip a
```
Deberia ser "eth0", "enp0s3", o "ens33"

Despues buscamos el archivo de configuracion de netplan en la siguiente carpeta:
```
ls /etc/netplan/
```
Debe ser un archivo .yaml como "00-installer-config.yaml" o "50-cloud-init.yaml"

Cuando lo hayamos localizado lo editamos con:
```
sudo nano nombre_del_archivo.yaml
```

Como vamos a establecer una ip estatica editamos el archivo de la siguiente manera
```
network:
  version: 2
  renderer: networkd
  ethernets:
    nombre_interfaz:
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
Crucial YAML Rules: Do NOT use tabs; use exact spaces for indentation. Keep the dashes - formatted exactly as shown.


Ahora, antes de instalar Wazuh le hacemos Hardening al servidor

1. Crear un usuario administrativo
2. Configurar ssh
3. Instalar Fail2Ban
4. Configurar UFW

-----------------------------------------------------------------------------
## 1. Crear un usuario administrativo
```
sudo adduser sergioadmin
sudo usermod -aG sudo sergioadmin #este comando le da permisos de usar sudo al usuario sergio admin (la flag -a es para append la -G es para definir a que grupo)
```
y lo probamos con
```
su - sergioadmin
sudo whoami
```
y deberia aparecer
```
root
```
-----------------------------------------------------------------------------
## 2. Configurar ssh

Editamos el archivo de configuracion de ssh
```
sudo nano /etc/ssh/sshd_config
```

y ponemos las configuraciones asi:
```
PermitRootLogin no

PasswordAuthentication no

PubkeyAuthentication yes

kbdinteractiveauthentication no

UsePAM yes

MaxAuthTries 3

LoginGraceTime 30

X11Forwarding no
```

guardamos reiniciamos el servicio y verificamos

```
sudo systemctl restart ssh
sudo systemctl enable ssh
sudo systemctl status ssh
```

-----------------------------------------------------------------------------
## 3. Instalar Fail2Ban

```
sudo apt install fail2ban -y
```
y verificamos
```
sudo systemctl status fail2ban
```

-----------------------------------------------------------------------------
## 4. Configurar UFW

Aunque el firewall principal sea pfsense, otro firewall local da mas proteccion
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
Permitimos el trafico de la subred de manejo (Management)
```
sudo ufw allow from 10.0.40.0/28 to any port 22
```


-----------------------------------------------------------------------------
-----------------------------------------------------------------------------
# Preparar pfSense

Create firewall rules before installing Wazuh.
Cada regla la crearemos en la interfaz origen

Allow:

Management → Security
```
Port	Purpose
22	SSH
443	Dashboard
```

Users → Security
```
Port	Purpose
1514 TCP	Agent
1515 TCP	Enrollment
```

Servers → Security
```
Port	Purpose
1514 TCP	Agent
1515 TCP	Enrollment
```

-----------------------------------------------------------------------------
-----------------------------------------------------------------------------
# Instalar Wazuh

Seguimos la guia de instalacion: https://documentation.wazuh.com/current/quickstart.html

Corremos el comando 
```
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```
Al correr este comando se instala wazuh automaticamente y da las credenciales de acceso al dashboard al cual podremos entrar desde la AdminVM
