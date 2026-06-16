Para instalar la primera vm de usuario que ira conectada al dominio toca primero descargar una iso de Windows 11 de la pagina oficial de Windows (https://www.microsoft.com/es-es/software-download/windows11)

Ya con la imagen creamos una nueva maquina virtual en VMWare en la red de users (10.0.10.0/24) con 8GB de ram y 64 de disco.

Instalamos la version Pro para que la maquina se pueda unir a un dominio (Con la version Hme no se puede).

Continuamos la instalación, y cuando pida si quiere configurar la cuenta como personal o de trabajo, escogemos de trabajo ("Set up for work or school"). Despues pedira una cuenta de Microsft, pero hay otra opción que es incluirla a un dominio. Escogemos esa opcion. 

Si no aparece oprimimos Shift + F10 para que aparzca la terminal y corremos el comando:
  OOBE\BYPASSNRO

Eso reiniciara la maquina y toca volver a configurarla para trabajo y aparecera la opcion del dominio

Creamos una cuenta con el usuario y contraseña que qeuramos y se completa la instalación.

-----------------------------------------------------------------------------------------------

Para 
