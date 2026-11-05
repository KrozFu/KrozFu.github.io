# Comandos básicos de Linux

Referencia de los comandos vistos en la primera clase: conexión, reconocimiento del sistema y primeras herramientas sobre un servidor **Ubuntu Server 24.04** en VirtualBox al que se accede con `ssh srv`.

## 1. Conectarse al servidor

| Comando | Qué hace |
| --- | --- |
| `ssh -p 2222 <usuario>@127.0.0.1` | Entra a la VM por el reenvío de puertos de VirtualBox (2222 → 22) |
| `ssh-keygen -t ed25519 -C "<correo>"` | Crea el par de llaves en el equipo; la privada nunca se comparte |
| `ssh-copy-id -p 2222 <usuario>@127.0.0.1` | Copia la llave pública al servidor (Linux/macOS) |
| `ssh srv` | Entra usando el alias definido en `~/.ssh/config` |
| `scp archivo srv:` | Copia un archivo del equipo a la carpeta personal del servidor |
| `sudo poweroff` / `sudo reboot` | Apaga / reinicia el servidor |

Alias en `~/.ssh/config` del equipo:

```text
Host srv
    HostName 127.0.0.1
    Port 2222
    User <usuario>
```

En Windows (PowerShell), sin `ssh-copy-id`:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 <usuario>@127.0.0.1 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

## 2. Conocer el sistema

| Comando | Qué hace |
| --- | --- |
| `hostnamectl` | Nombre del servidor, SO, kernel y virtualización |
| `uname -a` / `uname -r` | Kernel completo / solo la versión |
| `cat /etc/os-release` | Distribución y versión |
| `ip -brief address` | Interfaces y sus IP (en la NAT: `10.0.2.15`) |
| `ss -tln` | Puertos TCP en escucha (`:22` es SSH) |
| `nproc` | Número de CPU |
| `cat /proc/cpuinfo`, `head -3 /proc/meminfo` | CPU y memoria, desde el kernel |
| `cat /proc/uptime`, `cat /proc/loadavg` | Tiempo encendido y carga promedio |
| `echo $$` | PID de la shell actual |
| `grep -E "^(Name | State | PPid | Threads)" /proc/$$/status` | Estado de un proceso visto en `/proc` |
| `ls -l /proc/$$/fd` | Descriptores abiertos: 0 entrada, 1 salida, 2 error |
| `lsmod`, `modinfo e1000` | Módulos del kernel cargados / información de uno |

!!! note
    `/proc` es un sistema de archivos virtual: el kernel genera su contenido en el momento en que se lee; no está en el disco.

## 3. Instalar paquetes y rastrear syscalls

| Comando | Qué hace |
| --- | --- |
| `sudo apt update` | Actualiza la lista de paquetes (no instala nada) |
| `sudo apt install -y strace build-essential` | Instala paquetes |
| `strace -e trace=write -o traza.txt echo hola` | Guarda en un archivo las syscalls `write` de un programa |
| `strace -c ls` | Resumen de syscalls por tipo |
| `time <comando>` | Tiempo `real`, `user` (modo usuario) y `sys` (modo kernel) |

<!-- ## 4. Git del repositorio del estudiante

```bash
git init so-<apellido>
git add .
git commit -m "docs: registro de la instalación del servidor"
git branch -M main
git remote add origin <url>
git push -u origin main
``` -->

!!! danger
    Nunca se sube la llave privada (`id_ed25519`) ni contraseñas.
