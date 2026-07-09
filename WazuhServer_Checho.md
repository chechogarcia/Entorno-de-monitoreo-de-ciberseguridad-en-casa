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

