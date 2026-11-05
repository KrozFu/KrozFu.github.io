# Comandos básicos de Linux

Referencia de los comandos vistos en la segunda clase: shell, redirecciones, usuarios, permisos, `sudo`, SSH endurecido y scripts, sobre el mismo servidor de la Clase 1 (`ssh srv`).

## 1. Moverse y entender el shell

| Comando | Qué hace |
| --- | --- |
| `ls /`, `ls -l`, `ls -li` | Lista archivos; `-l` en detalle, `-i` con número de inodo |
| `cd <carpeta>`, `cd ~`, `pwd` | Cambia de carpeta / va a la carpeta personal / muestra dónde está |
| `mkdir -p a/b/c` | Crea carpetas, incluidas las intermedias |
| `cat archivo`, `head`, `less` | Muestra un archivo / las primeras líneas / lo pagina |
| `df -h /`, `free -h` | Espacio en disco / memoria |
| `type cd`, `type ls` | Dice si es un comando interno, un alias o un programa |
| `echo $PATH` | Carpetas donde el shell busca los programas |

Las carpetas clave de la jerarquía (FHS) son `/etc` (configuración), `/home` (usuarios), `/var/log` (registros), `/tmp` (temporales), `/srv` (datos de servicios), `/usr/bin` (programas) y `/dev` (dispositivos).

## 2. Redirecciones y código de salida

| Sintaxis | Qué hace |
| --- | --- |
| `cmd > archivo` | La salida va al archivo (lo sobrescribe) |
| `cmd >> archivo` | Agrega al final |
| `cmd 2> archivo`, `cmd 2>/dev/null` | Guarda los errores / los descarta |
| `cmd1 | cmd2` | La salida de uno es la entrada del otro |
| `echo $?` | Código de salida del último comando: `0` es éxito |
| `grep -c bash /etc/passwd` | Cuenta las líneas que coinciden |

## 3. Usuarios y grupos

| Comando | Qué hace |
| --- | --- |
| `id`, `id ana` | UID, GID y grupos |
| `getent group sudo` | Miembros de un grupo |
| `sudo groupadd desarrollo` | Crea un grupo |
| `sudo useradd -m -s /bin/bash -G desarrollo ana` | Crea un usuario con carpeta personal, shell y grupo secundario |
| `sudo passwd beto` | Pone o cambia una contraseña |
| `sudo -u ana <comando>` | Corre un comando como otro usuario |
| `sudo -iu beto` | Abre una sesión como otro usuario (`exit` para volver) |

`/etc/passwd` lo lee todo el mundo; `/etc/shadow` (los hashes) solo root y el grupo `shadow`. Los UID de 0 a 999 son root y cuentas de sistema; desde 1000 son personas.

## 4. Permisos

| Comando | Qué hace |
| --- | --- |
| `chmod 640 archivo` | Permisos en octal (r=4, w=2, x=1; dueño, grupo, otros) |
| `chmod g+w,o-r archivo` | Permisos en notación simbólica |
| `chmod 2770 carpeta` | Con SGID: lo nuevo hereda el grupo de la carpeta |
| `chown root:desarrollo carpeta` | Cambia dueño y grupo |
| `umask` | Permisos que se quitan a lo nuevo |
| `stat -c '%A %a %U:%G %n' archivo` | Permisos en letras y en octal, dueño y grupo |
| `setfacl -m g:auditoria:rx carpeta` | ACL de acceso: un grupo más con permisos |
| `setfacl -d -m g:auditoria:rx carpeta` | ACL por defecto: la heredan los archivos nuevos |
| `getfacl carpeta` | Muestra las ACL (`ls -l` marca un `+`) |
| `find / -xdev -perm -4000 -type f 2>/dev/null` | Lista los binarios con SUID |
| `dpkg -S /usr/bin/passwd` | Dice qué paquete instaló un archivo |

!!! tip
    `setfacl` y `getfacl` vienen en el paquete `acl`, que no está en un servidor recién instalado: `sudo apt install -y acl`.

Referencia en octal:

| Octal | Letras | Uso típico |
| --- | --- | --- |
| `755` | `rwxr-xr-x` | Programas |
| `644` | `rw-r--r--` | Configuración |
| `600` | `rw-------` | Llaves |
| `700` | `rwx------` | `~/.ssh` |
| `2770` | `rwxrws---` | Carpeta de equipo |
| `1777` | `rwxrwxrwt` | `/tmp` |

En un directorio, `r` lista los nombres, `w` crea y borra, y `x` deja entrar.

## 5. `sudo` y SSH endurecido

| Comando | Qué hace |
| --- | --- |
| `sudo -l -U beto` | Qué puede correr un usuario con `sudo` |
| `sudo visudo -cf <archivo>` | Valida un archivo de sudoers antes de instalarlo |
| `sudo install -m 0440 <archivo> /etc/sudoers.d/<nombre>` | Instala la regla con los permisos que exige `sudo` |
| `sudo sshd -t` | Revisa la sintaxis de la configuración de SSH |
| `sudo sshd -T | grep -E '^(passwordauthentication | permitrootlogin) '` | Configuración efectiva de SSH |
| `sudo systemctl restart ssh` | Reinicia SSH (se prueba desde otra terminal, sin cerrar la actual) |
| `ssh -o PubkeyAuthentication=no srv` | Comprueba que ya no acepta contraseña: `Permission denied (publickey).` |
| `journalctl _COMM=sudo` | Quién usó `sudo` y para qué |

!!! warning
    Antes de reiniciar SSH, deja abierta la sesión actual y prueba la conexión desde otra terminal. Si la configuración quedó mal, todavía puedes corregirla.

## 6. Scripts

| Comando | Qué hace |
| --- | --- |
| `chmod +x script.sh && ./script.sh` | Lo hace ejecutable y lo corre |
| `bash -x script.sh` | Lo corre mostrando cada línea: así se depura |
| `set -euo pipefail` | Primera línea útil del script: para en el primer error |

!!! tip
    En Windows, guarda los scripts con fin de línea LF; con CRLF, bash falla con `$'\r': command not found`.
