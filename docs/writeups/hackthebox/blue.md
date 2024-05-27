
## Información Box

| Parámetros | Características                                  |
| ---------- | ------------------------------------------------ |
| OS         | Windows                                          |
| Dificultad | Easy                                             |
| Creador    | ch4p                                             |
| Link       | [Blue](https://www.hackthebox.com/machines/blue) |

## Introducción
Blue es un CTF que podemos encontrar en diversas plataformas. La dificultad de esta máquina es fácil. Tendremos que explotar la vulnerabilidad MS17-010 para obtener una shell como System. Veremos cómo enumerar si la máquina es vulnerable a MS17-010 usando Nmap, y cómo explotarla tanto con Metasploit como usando scripts en Python.

## Requisitos previos
Creamos **directorios** para poder almacenar los diferentes archivos que se trabajan en la maquina.
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

Al finalizar se observa el siguiente escaneo
```java
# Nmap 7.94SVN scan initiated Sun May 26 15:54:05 2024 as: nmap -p- -sV -sC --open -sS -vvv -Pn -oN escaner 10.129.230.92
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
# Nmap done at Sun May 26 15:55:50 2024 -- 1 IP address (1 host up) scanned in 105.50 seconds
```

Como se puede observar, el servicio `smb` está presente, lo cual es muy común en los servicios de Windows. Además, hay otros puertos abiertos típicos de un servidor de Windows. Hemos logrado obtener información sobre la versión del sistema operativo de la máquina, que es `Windows 7 Professional`.
Se conoce que el protocolo `smb` utiliza el puerto `445`. En este laboratorio, nos enfocaremos en escalar privilegios mediante este protocolo.

## Enumeración de SMB
Realizamos un escaneo al protocolo SMB, que es comúnmente utilizado por los sistemas Windows para compartir archivos, impresoras y otros recursos en la red. Ejecutamos todos los scripts de `nmap` para `smb-vuln`, los cuales buscan vulnerabilidades conocidas en servicios SMB. También utilizamos el escaneo `-sT` para conexión TCP, un método más confiable en comparación con otros tipos de escaneo. Sin embargo, este método es más lento y puede ser detectado por sistemas de seguridad. En este caso, optamos por este tipo de escaneo para identificar las vulnerabilidades presentes.

```bash
nmap -p445 -script=smb-vuln-\* -sT $blue -oN smbScan
```

Obtenemos la siguiente salida al ejecutar el comando y podemos observar la vulnerabilidad que podemos utilizar para explotar este servicio.

```java
# Nmap 7.94SVN scan initiated Sun May 26 16:03:54 2024 as: nmap -p445 -script=smb-vuln-* -sT -oN smbScan 10.129.230.92
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

# Nmap done at Sun May 26 16:04:07 2024 -- 1 IP address (1 host up) scanned in 13.25 seconds
```
Se puede observar al finalizar el escaneo, que posee una vulnerabilidad con CVE de `CVE:CVE-2017-0143`.

## Explotación
En este apartado vamos ayudarnos de la herramienta de `Metasploit`, para poder explotar esta maquina, realizamos la búsqueda de la vulnerabilidad en la base de datos de Metasploit.

![img](./img/blue/Pasted%20image%2020240526151354.png)

Vamos a utilizar el `exploit 0`, y configuramos los parámetros necesarios para realizar el escalamiento.

![img](./img/blue/Pasted%20image%2020240526151714.png)

Configuración de los parámetros, con ayuda de Metasploit se logra tener una conexión al servidor, y lograr tener el respectivo acceso al sistemas.

![img](./img/blue/Pasted%20image%2020240526152443.png)

Luego de obtener acceso a la maquina por medio de Metasploit, se puede observar que tenemos acceso a la línea de comandos de Windows, con lo cual ahora podemos acceder a los diferentes directorios del sistema.

![img](./img/blue/Pasted%20image%2020240526161044.png)

Navegamos  por los directorios del sistema, y logramos acceder como `Administrator` que es el usuario `root` de Windows, obteniendo los máximos privilegios del sistema.

![img](./img/blue/Pasted%20image%2020240526161242.png)

<h1>!pwned</h1>