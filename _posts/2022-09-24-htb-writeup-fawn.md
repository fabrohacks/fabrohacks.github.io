---
layout: single
title: Fawn - Hack The Box
excerpt: "Fawn corresponde a la segunda caja de la serie Learn the Basics of Penetration Testing, para la resolución de este objetivo es necesario generar una instancia de máquina a través de openVPN. Fawn explota las vulnerabilidades del Protocolo de Transferencia de Archivos (FTP). Asi que, sigamos y hackeemos la segunda caja de Hack The Box."
date: 2022-09-23
classes: wide
header:
  teaser: /assets/images/htb-writeup-fawn/fawn-logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hack the box
  - starting point
  - tier 0
tags:
  - external
  - penetration tester level 1
  - enumeration
  - ftp
---

![](/assets/images/htb-writeup-fawn/fawn1.png)

Fawn corresponde a la segunda caja de la serie Learn the Basics of Penetration Testing, para la resolución de este objetivo es necesario generar una instancia de máquina a través de openVPN. Fawn se considera una máquina muy facil con un sistema operativo linux que ejecuta un servidor FTP simple y mal configurado. Así que, sigamos y hackeemos la segunda caja de Hack The Box 

## Introducción

El Protocolo de Transferencia de Archivos (FTP) es un protocolo de comunicación estandar utilizado para transferir archivos de computadora desde un servidor a un cliente utilizando el puerto 21. FTP se basa en una arquitectura cliente - servidor y utiliza conexiones de datos y control separadas entre el cliente (host) y el servidor (un dispositivo de almacenamiento de datos centralizado).
- En terminos generales, el cliente siempre es el host que descarga y carga archivos al servidor, y el servidor siempre es el host que almacena de forma segura los datos que se transfieren

Los usuarios de FTP pueden autenticarse con un protocolo de inicio de sesión, generalmente consta de un nombre de usuario y contraseña, FTP a menudo se protege con SSL/TLS (FTPS) para una transmisión segura que proteje el usuario, la contraseña y el contenido mediante cifrado, o a su vez se reemplaza con el protocolo de transferencia de archivos SSH (SFTP) mediante el puerto 22 o HTTPD (servidor web puerto 80), Sin embargo, el servicio FTP permite conexiones de manera anonima si el servidor esta configurado para permitirlo.

Si los administradores de red permiten usar FTP sin la capa de cifrado esta comunicación se podría frustar con éxito utilizando el ataque Hombre-en-el-Medio (Man-in-the-Middle attacks, MITM), no obstante si la conexión es cifrada con el protocolo SSL/TLS a través de SSH para agregar una capa de cifrado que solo los host de origen y destino pueden descifrar, esta técnica frustaría con éxito la mayoría de los ataques MITM.

![](/assets/images/htb-writeup-fawn/ftp-esquema.png)

<p align="center">Esquema del protocolo FTP, cifrado y método de Ataque MITM</p>
<p align="center">Fuente: Fawn Write-up - Hack The Box</p>

## Enumeración

![](/assets/images/htb-writeup-fawn/ip-fawn.png)

Antes de realizar un escaneo de los puertos de la maquina Fawn, debemos verificar si nuestra conexión VPN está establecida, para ello hacemos uso del protocolo ping para llegar al objetivo y obtener una respuesta. El protocolo ping se invoca desde un terminal usando el comando `ping {target_IP}`, donde target_IP es la dirección IP de la máquina Fawn, si se tiene problemas en crear la conexión VPN revisar el tutorial <a href="/htb-writeup-meow/" target="_blank">Meow</a> donde se explica como crear una instancia de máquina con Hack The Box.

```
┌─[fabro@parrot]─[~]
└──╼ $ping -c4 10.129.94.137
PING 10.129.94.137 (10.129.94.137) 56(84) bytes of data.
64 bytes from 10.129.94.137: icmp_seq=1 ttl=63 time=173 ms
64 bytes from 10.129.94.137: icmp_seq=2 ttl=63 time=173 ms
64 bytes from 10.129.94.137: icmp_seq=3 ttl=63 time=178 ms
64 bytes from 10.129.94.137: icmp_seq=4 ttl=63 time=175 ms

--- 10.129.94.137 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 172.810/174.655/177.699/1.935 ms
```

Ahora podemos escanear los servicios abiertos en el host con el script de Network Mapper (nmap) el comando seria `sudo nmap {target_IP}`

```
┌─[fabro@parrot]─[~]
└──╼ $sudo nmap 10.129.94.137
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-26 18:49 -05
Nmap scan report for 10.129.94.137
Host is up (0.18s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp

Nmap done: 1 IP address (1 host up) scanned in 3.40 seconds
```

Con el comando anterior observamos que el servicio FTP se encuentra ejecutándose en el puerto abierto 21, no obstante no conocemos la versión del servicio por lo cuál ejecutamos el siguiente comando `sudo nmap -sV {target_IP}` el interruptor -sV de nmap nos permite conocer la versión del servicio, el escaneo llevará más tiempo pero podriámos saber si el objetivo es vulnerable debido a la ejecucción de un software obsoleto 

```
┌─[fabro@parrot]─[~]
└──╼ $sudo nmap -sV 10.129.94.137
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-26 18:53 -05
Nmap scan report for 10.129.94.137
Host is up (0.18s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
Service Info: OS: Unix
```

## Foothold

Para acceder al servicio FTP, se utiliza el comando `ftp` en nuestro host, si la herramienta no se encuentra instalada lo realizamos con el siguiente comando: `sudo apt install ftp -y`, la instancia -y se usa para aceptar la instalación sin interrumpir el proceso, una vez finalizada la instalación se puede ejecutar el comando `ftp -h` para ver las capacidades del servicio.

```
┌─[fabro@parrot]─[~]
└──╼ $sudo apt install ftp -y 
Leyendo lista de paquetes... Hecho
Creando árbol de dependencias... Hecho
Leyendo la información de estado... Hecho
ftp ya está en su versión más reciente (0.17-34.1.1).
```

Para acceder al host de destino usamos el comando `ftp {target_IP}`, este comando iniciará una solicitud de autenticación en el servicio FTP que se encuentra ejecutando

```
┌─[fabro@parrot]─[~]
└──╼ $ftp 10.129.94.137
Connected to 10.129.94.137.
220 (vsFTPd 3.0.3)
```

El prompt nos pedirá el nombre de usuario con el que queremos iniciar sesión. Aqui es donde ocurre la magia.

Una mala configuración típica para la ejecucción de los servicios de FTP es permitir que una cuenta anónima acceda al servicio como cualquier usuario autenticado.
- El usuario anonymous se puede ingresar cuando aparece el mensaje, seguido de cualquier contraseña, debido que el servicio ignorará la contraseña de esta cuenta.

```
Name (10.129.94.137:fabro): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

Podemos ver que iniciamos sesión correctamente, nuestro prompt cambia para indicarnos que ahora podemos emitir comandos ftp, si escribimos el comando `help, -h o --help` generará una lista de todos los comandos disponible, por el momento nos interesa listar el contenido de la máquina y descargar los archivos que está contenga.

Con el comando `ls` podemos listar el contenido de la carpeta.

```
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
226 Directory send OK.
```
Para descargar los archivos a nuestro host utilizamos el comando `get` seguido del nombre del archivo, esto descargará el archivo en el directorio que nos encontremos dentro de nuestro host, por ahora nos interesa el archivo `flag.txt`

```
ftp> get flag.txt
local: flag.txt remote: flag.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for flag.txt (32 bytes).
226 Transfer complete.
32 bytes received in 0.00 secs (10.8620 kB/s)
```
Para salir de la conexión ftp ejecutamos el comando `bye` esto nos devolvera el control de nuestro terminal, a continuación procedemos a listar el contenido de nuestro directorio actual y verificamos si el archivo existe.

```
ftp> bye
421 Timeout.
┌─[fabro@parrot]─[~]
└──╼ $ls
flag.txt
```

¡Ahora podemos tomar la bandera y enviarla a la plataforma para obtener la caja! ¡Buen trabajo!

```
┌─[fabro@parrot]─[~]
└──╼ $cat flag.txt 
035db21c881520061c53e0536e44f815
```

## Respuestas

Task 1

What does the 3-letter acronym FTP stand for? 
- File Transfer Protocol

Task 2

Which port does the FTP service listen on usually? 
- 21

![](/assets/images/htb-writeup-fawn/task1-2.png)

Task 3

 What acronym is used for the secure version of FTP? 
- SFTP

Task 4

What is the command we can use to send an ICMP echo request to test our connection to the target? 
- ping

![](/assets/images/htb-writeup-fawn/task3-4.png)

Task 5

 From your scans, what version is FTP running on the target? 
- vsftpd 3.0.3 

Task 6

From your scans, what OS type is running on the target? 
- Unix

![](/assets/images/htb-writeup-fawn/task5-6.png)

Task 7

What is the command we need to run in order to display the 'ftp' client help menu? 
- ftp -h

Task 8

What is username that is used over FTP when you want to log in without having an account?
- anonymous

![](/assets/images/htb-writeup-fawn/task7-8.png)

Task 9

What is the response code we get for the FTP message 'Login successful'? 
- 230

Task 10

There are a couple of commands we can use to list the files and directories available on the FTP server. One is dir. What is the other that is a common way to list files on a Linux system. 
- ls

![](/assets/images/htb-writeup-fawn/task9-10.png)

Task 11

What is the command used to download the file we found on the FTP server?
- get

Task 12

Submit root flag
- 035db21c881520061c53e0536e44f815

![](/assets/images/htb-writeup-fawn/task11-12.png)

## Certificado

<p> Al culminar HTB nos otorgara un certificado, mi certificado de la máquina Fawn se encuentra disponible <a href="https://www.hackthebox.com/achievement/machine/914953/393" target="_blank">aquí</a>.</p>
![](/assets/images/htb-writeup-fawn/certificado.png)
