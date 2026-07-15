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
sudo usermod -aG sudo sergioadmin
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

ChallengeResponseAuthentication no

UsePAM yes

MaxAuthTries 3

LoginGraceTime 30

X11Forwarding no
```

guardamos y reiniciamos el servicio

```
sudo systemctl restart ssh
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

Allow:

Management → Security
Port	Purpose
22	SSH
443	Dashboard

Users → Security
Port	Purpose
1514 TCP	Agent
1515 TCP	Enrollment

Servers → Security
Port	Purpose
1514 TCP	Agent
1515 TCP	Enrollment

