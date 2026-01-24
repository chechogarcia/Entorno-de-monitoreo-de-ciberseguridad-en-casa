Primero se instala el firewall pfSense en vmware:

(video guía: https://www.youtube.com/watch?v=p09ggIHt2j4)

1. Descargamos la imagen ISO de https://www.pfsense.org/download/

2. Creamos la máquina virtual en vmware con la imagen ISO de pfSense
     - ![Creación VM](/images/Creacion_Maquina_virtual_pfSense_ISO.png)

5. Crear la Red interna desde Virtual Network Editor (correr como administrador)
    - Esta red será del tipo Host-Only
    - Desactivamos el DHCP, para que esto sea manejado por el fiurewall como tal y no la maquina virtual.
    - ![Virtual Network Editor](/images/Virtual_Network_Editor_config.png)

4. Configuramos la máquina virtual para que tenga 2 adaptadores de red (uno para WAN y otro para la LAN)
       - Antes de iniciar la máquina virtual añadimos un adaptador de red y le ponemos 2 procesadores (minimo) y 4 de RAM.
       - El primer adaptador sera para la WAN, entonces elegimos el tipo de conexion "Bridged"
       - ![VM settings 1](/images/pfSense_VM_settings_network_adapter_1.png)
       - Para el segundo adaptador (que da hacia la LAN), elegimos el tipo de conexion Custom y escogemos la red virtual creada en el punto 3 (para mi caso sera VMNet2).
       - ![VM settings 2](/images/pfSense_VM_settings_network_adapter_2.png)

5. Iniciamos la máquina virtual y continuamos la instalacion
       - Se debe escoger las interfaces de red que se van a usar y en donde se van a usar. Nos aseguramos que la interfaz de la WAN (VMNet0) este configurada para la WAN en el firewall y que la de la LAN (VMNet2) este para la LAN.

6. Validamos que se hayan instalado correctamente viendo las IP's asignadas a las interfaces.

7. Probamos conectividad en internet
       - Para esto usé el comando "ping 8.8.8.8"

