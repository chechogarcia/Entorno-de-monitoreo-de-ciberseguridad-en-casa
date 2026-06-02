Creamos una maquina virtual con Debian (En este caso usé Debian 13.4)
Lo montamos en la red de Management (10.0.40.0/28)
Al configurar la maquina virtual le damos 2 GB de RAM y 20 GB de Disco pues al ser un Bastion Host no neecesita muchos recursos y debe correr lo más mínimo

Al encender la máquina virtual, como no se ha configurado ninguna regla para esta red en el firewall, no va a tener conectividad. Por lo que es necesario configurar las opciones de red manualmente.

Mantenemos el default gateway y el name server (DNS) como 10.0.40.1
Cuando se configure el DNS en la red de servidores cambiaremos el DNS en esta máquina

Incluimos tambien un servidor de fallback (En este caso usaremos 8.8.8.8)

Como este es un ambiente local y de prueba, cuando se solicita el nombre del dominio usaremos lab.internal

De esta forma el FQDN será bastion01.lab.internal

Lo ideal, y para mejora la seguridad, es desactivar el acceso con root de forma remota. Esto lo hace Debian automaticamente si no se establece una contraseña para el usuario root en la configuracion de usuarios.

Al hacer esto nos pedira que creemos un usuario que va a tener menos privilegios que el superusuario, pero que puede usar root para acceso privilegiado temporal. Se pedira un nombre de usuario y una contraseña

Continuando la configuracion se nos va a pedir que configuremos un packet manager. Esto se maneja automaticamente en la instalacion, entonces solo seleccionamos el mirror de Estados Unidos y el mirror deb.debian.org

Despues se pregunta por un proxy. Como estamos por ahora comenzando no se incluye un proxy, sin embargo, para futuras actividades (Como Web filtering, Traffic inspection, Malware sandboxing, Corporate simulation) podemos desplegar un Squid, Burp Suite o mitmproxy

Ahora, para que se instalen los paquetes la maquina debe tener acceso a internet, por lo que añadimos una regla que permita el trafico de cualquier subred de management hacia afuera (outbould rule)

Se crea una regla que permite todo trafico IPV4 hacia afuera, sin embargo, esta regla se debe modificar mas adelante para solo permitir tráfico necesario hacia afuera (Ejemplo: DNS, HTTP/HTTPS, SSH/RDP) Todo el resto de trafico innecesario se deberia bloquear

Como la instalacion para un servidor Bastion debe ser minima, en la pestaña de "Software Selection", solo seleccionamos las casillas de "SSH Server" y "standard system utilities". Esto instalará openssh para la administracion remota, acceso seguro por shell, tunneling ssh, SCP/SFTP y pivoteo y manejo

No se incluye la GUI porque esta es mas dificil de asegurar (harden), es mas pesada y tiene una mayor superficie de ataque

Despues, como este es el unico sistema operativo en la maquina virtual, instalamos el GRUB boot loader como se haria en una maquina fisica normal

Aqui termina la instalacion y se reinicia la maquina virtual y ya se puede acceder a ella con el usuario que creamos anteriormente

-----------------------------------------------------------------------------------------------------------------------------------------------------
#Hardening

Despues de completar la instalación probamos conectividad

ip a

ip route (deberia mostrar la default gateway como la configuramos)

ping <default gateway>
ping 8.8.8.8
ping google.com

sudo apt update && sudo apt upgrade -y

Despues de actualizar los paquetes deshabilitamos la regla del firewall que permite el acceso a internet. 


-----------------------------------------------------------------------------------------------------------------------------------------------------
Ahora, para instalar herramientas como vim, curl/wget, tmux, htop, nmap, tcpdump, ufw, fail2ban, dnsutils, y para recibir actualizaciones de los paquetes debemos añadir una regla en el firewall que permita este tráfico

El firewall PfSense permite resolver nombres de dominio especificos y actualizar dinamicamente sus reglas de firewall usando el FDQN Alias que se haya configurado.

Para hacer esto abrimos la GUI del firewall y vamos al apartado Firewall>Aliases en la pestaña IP

Presionamos 'Add' y creamos un alias de la siguiente forma:

  1. Name: Bastion_Repo_Domains
  2. Type: Host(s)

Y añadimos el FDQN abajo.
  deb.debian.org
  security.debian.org

Guardamos y aplicamos los cambios.

Antes de continuar a crear las reglas para autorizar este trafico tomamos un snapshot del firewall y del Bastion.

*Insertar fotos snapshots*


Ahora creamos las reglas para permitir el trafico hacia afuera desde el bastion host.

1. Entramos a las reglas de la interfaz a la que esta conectado el Bastion y añadimos las siguientes reglas:

Action: Pass
Protocol: TCP/UDP
Source: Single Host or Alias -> [Your Bastion Host IP]
Destination: Any or Interface Address
Port Range: 53 (DNS)
Description: Allow DNS from Bastion Host


Action: Pass
Protocol: TCP
Source: Single Host or Alias -> [Your Bastion Host IP]
Destination: Single Host or Alias -> Bastion_Repo_Domains
Port Range: HTTP and HTTPS (You can create a Port Alias for 80 and 443, or create two separate rules)
Description: Allow Package Downloads from Bastion Host

Aplicamos los cambios


Ahora, para verificar la funcionalidad de estos cambios realizamos los siguientes pasos en el bastion host:

1. Borramos el cache de los paquetes viejos
  sudo apt-get clean
2. Actualizamos los paquetes para verificar que pfsense permite el trafico
  sudo apt-get update



Ahora que tenemos acceso al repositorio de paquetes, descargamos herramientas basicas dfe gestion para el bastion Host:

curl: Comando para probar connectividad a paginas web por http y https. Tambien sirve para probar APIs
auditd: Servicio de linux para auditar (man auditd despues de instalar para leer el manual)
fail2ban: herramienta para evitar ataques de fuerza bruta
ufw: firewall local para bloquear trafico malicioso
libpam-google-authenticator: Google Authenticator para MFA al iniciar sesion en el host (man google-authenticator)
htop: Monitoreo interactivo de telemetria del servidor (CPU, Memory and process usage)
wazuh agent
nmap: useful for network discovery and auditing security within the network (no puede salir de la red, solo escaneo hacia adentro)
tcpdump: para interceptar, capturar y analizar trafico de red (similar a wireshark pero en consola)
traceroute/MTR: Para diagnosticar problemas con la conectividad


Para todas aplicaciones, como estamos usando Debian 13 (que se especializa en seguridad y estabilidad), hay que revisar si hay algun tipo de bloque por AppArmor (activado por default en Debian)

AppArmor es una utilidad de Linux que sirve para restringir lo que pueden hacer programas especificos. Utiliza perfiles infividuales para restringir las actividades (por ejemplo leer un archivo o restringir a que redes puede acceder)

sudo aa-status | less
Esto para ver los perfiles cargados en apparmor

Los perfiles que esten en modo "Complain" son aplicaciones que funcionan con normalidad, pero cualquier accion que viole las reglas de AppArmor se registran en el log

Los perfiles que esten en modo "Enforce" son aplicaciones que ApppArmor restringe activamente segun su perfil

Para crear o modificar los perfiles toca instalar un paquete de utilidades:
sudo apt install apparmor-utils apparmor-profiles

Para modificar los perfiles toca modificar los archivos de texto plano de cada perfil ubicados en /etc/apparmor.d/

Los perfiles se nombran reemplazando las barras diagonales (/) de la ruta del ejecutable por puntos (.). Por ejemplo, el perfil para /usr/bin/ping se encuentra en /etc/apparmor.d/usr.bin.ping.
Aqui toca entender las reglas basicas:

Para acceso a archivos:
  r: Permite leer
  w: Permite escribir o modificar
  px: Ejecuta el archivo bajo su propio perfil de AppArmor.
  /**: El doble asterisco significa el directorio actual y todos sus subdirectorios.

Ejemplos:
  /var/log/nginx/*.log w,      # Permite escribir en cualquier archivo .log dentro de esa ruta
  /etc/nginx/** r,             # Permite leer todo lo que esté dentro de /etc/nginx/

Para acceso a la red:
  network inet stream,: Permite conexiones de red TCP (IPv4).
  network inet6 dgram,: Permite conexiones de red UDP (IPv6).


Otra forma es poniendo el programa en "modo aprendizaje" (Complain). Esto logea toda la actividad de ese programa. Despues de usar el programa de forma noprmal se corre el comando:
sudo aa-logprof

Con esto el sistema leera los logs de actividad y preguntará si quieres denegar o permitir cada una de las acciones. 
Esta es la forma recomendada.

Despues de modificar las reglas se corre el siguiente comando para recargar el perfil en AppArmor y se apliquen las restricciones en el kernel
sudo apparmor_parser -r /etc/apparmor.d/nombre.del.perfil

Finalmente, se regresa el programa al modo estricto (Enforce):
sudo aa-enforce /ruta/al/programa
-----------------------------------------------------------------------------------------------------------------------------------------------------

Ahora desplegamos fail2ban, ufw, el autenticador de google y auditd


1. ufw

En ufw debemos permitir el trafico desde la subred de manejo y el trafico ssh publico

sudo ufw allow from 10.0.40.0/28
sudo ufw allow 22/tcp

sudo ufw enable


2. Configurar el autenticador de google (PAM MFA)

Correr el comando:
  google-authenticator

Follow the on-screen prompts: Say Y to time-based tokens, Y to disallow multiple uses of the same token, N to the wide time-window, and Y to rate-limiting.

Actualizar la configuracion del PAM (Pluggable Authentication Modules) - PAM es un marco centralizado que gestiona la seguridad y la autenticación de usuarios
  sudo nano /etc/pam.d/sshd

Adicionar esta linea al comienzo del archivo justo debajo de la linea @include common-auth (nullok sirve para evitar el lockout si no se ha configurado la autenticacion para el usuario - se desactiva cuando termine el testeo)
  auth required pam_google_authenticator.so nullok

Se adiciona la linea en ese lugar para que no haya problemas con el setup de las claves ssh. Si se deja antews de esa linea cuando quiera iniciar sesion se pediria el codigo de google antes de pedir la conexion ssh


Finalmente, actualizamos la configuracion del servicio de ssh

sudo nano /etc/ssh/sshd_config

Modificamos el archivo para que incluya estas lineas

  PermitRootLogin no
  
  PubkeyAuthentication yes
  PasswordAuthentication yes
  
  KbdInteractiveAuthentication yes
  ChallengeResponseAuthentication yes
  
  UsePAM yes
  
  AuthenticationMethods keyboard-interactive
  
  X11Forwarding no
  AllowAgentForwarding no
  PermitTunnel no
  
  MaxAuthTries 3
  LoginGraceTime 30

Guardamos y corremos el siguiente comando para verificar que la configuracion esta bien escrita (Si no aparece nada es porque la configuracion esta bien):

sudo sshd -t

Reiniciar SSH
sudo systemctl restart ssh


3. Conectar Fail2Ban a UFW

(Otro recurso util para esta configuracion: https://www.alcancelibre.org/manuales/configuracion-de-fail2ban)

Crear una copia del archivo de configuracion de fail2ban

sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local

Modificamos el parametro "banaction" en la seccion [DEFAULT] y verificamos que la red de manejo este en la whitelist

  [DEFAULT]
  ignoreip = 127.0.0.1/8 ::1 10.0.40.0/28
  bantime  = 1h     # How long an IP is banned
  findtime  = 10m   # Time window to count failed attempts
  maxretry = 3      # Number of failures before ban
  banaction = ufw
  banaction_allports = ufw
  action = %(action_)s #Esta opcion ya estaactiva y action_ es para bloquear la ip invalida
  
  [sshd]
  enabled = true
  port    = ssh     # Or your custom SSH port

Habilitar e iniciar el servicio para aplicar los cambios:

  sudo systemctl enable fail2ban
  sudo systemctl start fail2ban


Use the fail2ban-client to check the status of your jails or unban yourself if necessary.

Check jail status: sudo fail2ban-client status sshd
Unban an IP: sudo fail2ban-client set sshd unbanip <IP_ADDRESS>
View logs: tail -f /var/log/fail2ban.log


Como estamos requiriendo doble autenticacion (clave ssh y codigo mfa) la configuracion de fail2ban probablemente no agarre los intentos fallidos de MFA, porque los logs son distintos a los que se forman por contraseña incorrecta

Para solucionar esto debemos entrar a /var/log/auth.log despues de un intento fallido de MFA para verificar que el filtro sshd de fail2ban coincida con el formato de log.

__________________________________________________________________________________________

Despues de probar la connectividad por ssh con contraseña y el codigo de google, creamos una nueva maquina que va a aser como la de un administrador que se conecta al bastion con ssh

Como el nuevo host esta dentro de la red de manejo el trafico ssh no pasa por el firewall, por lo que, asi no haya configurado una regla para el firewall que permita el ssh desde este nuevo host, la conexion igualmente es exitosa.

Cuando probamos el ssh, si todo quedó bien configurado, pedira contraseña y codigo de google, y despues de esto se hará la conexion.

___________________________________________________________________________________________

Ahora que vemos que la conexion por ssh con clave funciona, probaremos con las llaves publicas.

Para esto se debe crear una pareja de llaves publica-privada en el Host del administrador.

1. Crear pareja de llaves en la maquina de administrador

  ssh-keygen -t ed25519 -a 100

Why?

  Ed25519 is the modern SSH algorithm.
  Smaller keys.
  Faster.
  Excellent security.

Aceptar el default cuando se pida "Enter file in which to save the key"

Tambien, usar una passfrase cuando la pida

2. Verificar las llaves

  ls -l ~/.ssh

Esto mostrara la pareja de llaves (los permisos deberian ser iguales a los que se muestran en el ejemplo):
  -rw------- id_ed25519
  -rw-r--r-- id_ed25519.pub

3. Copiar la llave publica al bastion

  ssh-copy-id sergio@10.0.40.10

Nos va a pedir la clave de sergio@10.0.40.10, entonces la ponemos. Esto creará ~/.ssh en el bastion, authorized_keys, pondrá permisos, copiara la llave publica

Despues de hacerlo, el la terminal del bastion verificamos que si se haya copiado la llave publica: 
  cat ~/.ssh/authorized_keys

Deberia aparecer algo asi: ssh-ed25519 AAAAC3...

4. Requerir llave + mfa:

En el bastion:
  sudo nano /etc/ssh/sshd_config

Y modificamos esta linea para que quede de la siguiente manera:

  AuthenticationMethods publickey,keyboard-interactive


Verificamos que todo este asi:

  PermitRootLogin no

  PubkeyAuthentication yes
  PasswordAuthentication no
  
  KbdInteractiveAuthentication yes
  UsePAM yes
  
  AuthenticationMethods publickey,keyboard-interactive


5. Reiniciar SSH
  
  sudo sshd -t
  sudo systemctl restart ssh


6. Probar la autenticacion con la llave

En la maquina del administrador correr:
  ssh sergio@10.0.40.10

En este punto puede preguntar por la passfrase de la llave o directamente por el codigo de google


7. Verificar qué llave se esta usando para la conexión

  ssh -v sergio@10.0.40.10

Si aparecen estas dos lineas:
  Offering public key:
  Server accepts key

Eso confirma que la llave se esta usando correctamente.

8. Restringir ssh exclusivamwente para el Admin VM

En pfsense crear las reglas en OPT3:

| Action | Protocol | Source     | Destination | Port |
| ------ | -------- | ---------- | ----------- | ---- |
| Pass   | TCP      | 10.0.40.11 | 10.0.40.10  | 22   |

| Action | Protocol | Source   | Destination | Port |
| ------ | -------- | -------- | ----------- | ---- |
| Block  | TCP      | OPT3 net | 10.0.40.10  | 22   |



9. Testear antes de cerrar la session

Abrir otra terminal para el admin VM y volver a probar

  ssh sergio@10.0.40.10


El flow deberia ser el siguiente:

  SSH key accepted
  ↓
  Verification code:
  ↓
  TOTP accepted
  ↓
  Login successful


Steps adicionales:

En la terminal del bastion hacer lo siguiente:

  nano ~/.ssh/authorized_keys

Cambiar:

  ssh-ed25519 AAAA...
  ↓
  from="10.0.40.11" ssh-ed25519 AAAA...

Esto hace que asi alguien se haya robado la credencial publica, solo se pueda conectar desde 10.0.40.11

Por cierto, nunca se deberia tener la llave PRIVADA en el bastion. En el bastion unicamente debe estar la PUBLICA.
_____________________________________________________________________________

Para evitar ataques de fuerza bruta en caso de que el AdminVM este comprometido, editamos el archivo de configuracion de fail2ban para que evalue todas las requests

sudo nano /etc/fail2ban/jail.local

Modificamos el parametro "banaction" en la seccion [DEFAULT] para que quede de la siguiente manera:

  [DEFAULT]
  ignoreip = 127.0.0.1/8 ::1

Habilitar e iniciar el servicio para aplicar los cambios:

  sudo systemctl restart fail2ban
_____________________________________________________________________________

Para mejoras futuras, el AdminVM deberia ir en otra subred para que asi el trafico sí sea evaluado por el firewall.

Y tambien, incluir una vpn para la conexion segura.

For a realistic bastion architecture, many organizations place:

Admin workstation in one subnet/VLAN
Bastion in a different management subnet/VLAN

so that all SSH traffic must cross a firewall and can be filtered, logged, and monitored. In your lab, you may eventually want to separate the Admin VM and Bastion into different management segments if your goal is to practice enterprise-style access control.


Lecturas adicionales para mejoras futuras:
https://goteleport.com/blog/security-hardening-ssh-bastion-best-practices/
https://weakdh.org/
https://www.google.com/search?q=el+protocolo+icmp+es+vulnerable%3F&rlz=1C1GCEA_enCO1162CO1162&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTIHCAEQIRigATIHCAIQIRigAdIBCDgwNTJqMGo3qAIAsAIA&sourceid=chrome&ie=UTF-8&udm=50&fbs=ADc_l-bpk8W4E-qsVlOvbGJcDwpn60DczFdcvPnuv8WQohHLTaf9fS4tJ71bi2aHS-Pmeg0IA2nDWC3A9mnYKHxHonOwOArVa2mNEvEuOSTSa1dtwqTXiOEVAUkLmHqJS4NtOQBb5Y1SA11p0AKFH_iW_mlh-tTME_h7m1BDKcl9pi7TjFC6G6peerUfIZ5AehTCpBR6CaUu-gbsdEbUkKFZjmqUppXPSQ&aep=10&ntc=1&mstk=AUtExfCBKKBE20LseT9SRIkknmiZZ7w10wOxCwspEi37oGoQl3iOdcubHhUqkKQIMLJoTZVi9xnG8ViP651Ndhyub8h13UkHbLX_MQ439JZMQgB_RiaaHosh1rraOVQVILCeeP7ErzI3tv0E2hjLKXi9czD5Oqy7JBKZY0XQTw8B71ar6qHyWkBQyAJy7UbU5s4SmqWQXlQyyUPY1ZeC08z9mGkucfhI9poRkZ27DILgVdaZnn54cXmtqkmmhw&aioh=3&csuir=1&cs=1&mtid=B0gXaoerJY2HwbkP8YPUyA0
https://nvd.nist.gov/vuln/detail/CVE-2020-27347
