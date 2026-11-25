# Guía de implementación · Almacenamiento y respaldos

Esta guía deja listo en su servidor el criterio **Almacenamiento y respaldos** del proyecto final: un volumen LVM para los respaldos con opciones de montaje restrictivas, un respaldo automático con instantáneas y una restauración probada. Al final, todo queda en su repositorio para que otra persona pueda reconstruirlo.

!!! info "Entorno: VirtualBox"
    Esta guía está hecha para implementarse en una VM de **Ubuntu Server 24.04 sobre VirtualBox**, la misma de la [Sesión 01](../session-01.md) a la que se entra con `ssh srv` *(reenvío de puertos 2222 → 22)*. Los pasos para agregar discos, los nombres de los discos (`sdb`, `sdc`) y el uso de instantáneas asumen VirtualBox con controlador **SATA** y discos **VDI**. Si usa otro hipervisor (VMware, KVM, Hyper-V), el procedimiento para agregar los discos y tomar instantáneas cambia, y los discos pueden aparecer con otro nombre (por ejemplo `vdb` o `nvme0n2`).

---

## 0. ¿Ya lo tiene?

Si ya armó el volumen y el respaldo, compruébelo antes de repetir nada:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
sudo vgs datos
findmnt -no SOURCE,OPTIONS /srv/respaldos
systemctl is-enabled respaldo.timer
ls -l /srv/respaldos
```

| Si ve esto | Haga esto |
| --- | --- |
| `datos` con `VFree` mayor que 0, `/srv/respaldos` con `nodev,nosuid,noexec`, el timer `enabled` y carpetas con fecha | Vaya directo a la sección 6 *(restaurar)* y luego a la 8 *(repositorio)* |
| `Volume group "datos" not found` | Empiece en la sección 1 |
| `datos` existe pero `/srv/respaldos` no está montado | Revise la sección 3 *(fstab)* |
| El volumen está bien, pero no hay timer o no hay carpetas con fecha | Empiece en la sección 4 |
| `VFree` en `0` | Vea *Si se quedó sin espacio libre* al final de la sección 3 |

### Requisitos

El respaldo copia `/srv/proyecto` y `/srv/estado`, así que su servidor ya debe tenerlos:

```bash
ls -l /srv/proyecto
sudo ls -l /srv/proyecto/codigo
systemctl is-active estado
command -v setfacl rsync
```

- **Resultado esperado:** `codigo` con `drwxrws---+` *(el `+` indica que tiene ACL)*, `app.py` dentro, `estado` en `active`, y las rutas de `setfacl` y `rsync`.
- Si falta `setfacl`: `sudo apt install -y acl`. Si falta `rsync`: `sudo apt install -y rsync`.
- Si falta `/srv/proyecto` o el servicio `estado`, constrúyalos primero: son criterios del proyecto y el respaldo los necesita.

---

## 1. Agregar dos discos a la VM

1. Apague la VM: `sudo systemctl poweroff`.
2. **Tome una instantánea** en VirtualBox *(Máquina → Tomar instantánea, con el nombre `antes-almacenamiento`)*. Si algo sale mal, vuelve a este punto.
3. En **Configuración → Almacenamiento**, seleccione el controlador **SATA** y use el botón **Agregar disco duro → Crear**: tipo **VDI**, **reservado dinámicamente**, **5 GB**. Selecciónelo para conectarlo.
4. Repita el paso 3 para un segundo disco de 5 GB.
5. Encienda la VM y entre con `ssh srv`.

Necesita unos 10 GB libres en su equipo. Como los discos son dinámicos, empiezan casi vacíos y crecen con el uso.

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

```text
NAME                       SIZE TYPE FSTYPE      MOUNTPOINTS
sda                         50G disk
├─sda1                       1M part
├─sda2                       2G part ext4        /boot
└─sda3                      48G part LVM2_member
  └─ubuntu--vg-ubuntu--lv   24G lvm  ext4        /
sdb                          5G disk
sdc                          5G disk
sr0                       1024M rom
```

- `sdb` y `sdc` son los discos nuevos: **5G, sin particiones, sin `FSTYPE` y sin punto de montaje**. Los tamaños de `sda` cambian de una VM a otra.
- **Antes de seguir, confirme cuáles son sus discos nuevos.** Si en su VM tienen otro nombre, use ese nombre en todos los comandos. Particionar el disco equivocado borra el sistema.
- Los `loop` que aparecen en `lsblk` son de los snaps; ignórelos.

---

## 2. El volumen de datos con LVM

LVM pone una capa entre los discos y los sistemas de archivos. Con ella se puede **agrandar un volumen y agregar discos sin desmontar nada**:

```text
 discos                   /dev/sdb1          /dev/sdc
                             │                   │
 volúmenes físicos (PV)   pvcreate          pvcreate
                             └───────┬───────────┘
 grupo de volúmenes (VG)           datos                vgcreate / vgextend
                             ┌───────┴───────┐
 volúmenes lógicos (LV)  respaldos        (libre)       lvcreate / lvextend
                             │
 sistema de archivos      ext4 en /srv/respaldos        mkfs / mount
```

Primero, una partición GPT en `sdb` marcada para LVM, y el volumen encima:

```bash
sudo parted /dev/sdb --script mklabel gpt mkpart datos 1MiB 100% set 1 lvm on
lsblk /dev/sdb
sudo pvcreate /dev/sdb1
sudo vgcreate datos /dev/sdb1
sudo lvcreate -L 2G -n respaldos datos
sudo mkfs.ext4 -L respaldos /dev/datos/respaldos
sudo lvs datos
```

```text
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
sdb      8:16   0   5G  0 disk
└─sdb1   8:17   0   5G  0 part
  Physical volume "/dev/sdb1" successfully created.
  Volume group "datos" successfully created
  Logical volume "respaldos" created.
…
Writing superblocks and filesystem accounting information: done
  LV        VG    Attr       LSize
  respaldos datos -wi-a----- 2.00g
```

- `1MiB` como inicio deja la partición alineada con los bloques del disco.
- `set 1 lvm on` marca la partición como `Linux LVM`. Se ve con `sudo fdisk -l /dev/sdb`.

---

## 3. Montarlo con opciones restrictivas

```bash
sudo mkdir -p /srv/respaldos
echo "UUID=$(sudo blkid -s UUID -o value /dev/datos/respaldos) /srv/respaldos ext4 defaults,nodev,nosuid,noexec 0 2" | sudo tee -a /etc/fstab
sudo findmnt --verify
sudo systemctl daemon-reload
sudo mount -a && echo mount-ok
findmnt /srv/respaldos
df -h /srv/respaldos
```

```text
UUID=d92c66d3-… /srv/respaldos ext4 defaults,nodev,nosuid,noexec 0 2
…
mount-ok
TARGET         SOURCE                      FSTYPE OPTIONS
/srv/respaldos /dev/mapper/datos-respaldos ext4   rw,nosuid,nodev,noexec,relatime
Filesystem                   Size  Used Avail Use% Mounted on
/dev/mapper/datos-respaldos  2.0G  536K  1.8G   1% /srv/respaldos
```

| Opción | Qué impide en el volumen de respaldos |
| --- | --- |
| `nodev` | Que un archivo de dispositivo copiado ahí dé acceso a un disco |
| `nosuid` | Que un binario con el bit SUID se ejecute con los privilegios de su dueño |
| `noexec` | Ejecutar cualquier programa guardado ahí |

- Se monta por **UUID** y no por `/dev/sdb1`: si cambia el orden de los discos, el UUID sigue siendo el mismo.
- `findmnt --verify` revisa todo `/etc/fstab` y termina con un resumen. Lo que importa es que no haya errores *(`Success, no errors…` o `0 errors`)*: una advertencia sobre otra línea *(por ejemplo, la del swap)* no es de este paso.
- **`sudo findmnt --verify` y `mount -a` van antes de cualquier reinicio.** Una línea mal escrita en `/etc/fstab` puede dejar la VM en modo de emergencia al arrancar. Si alguno da error, corrija la línea con `sudo nano /etc/fstab`.
- `tee -a` **agrega** la línea al final. Si corrió el comando dos veces, borre la línea repetida.

*Pruebe que `noexec` actúa:*

```bash
sudo cp /usr/bin/true /srv/respaldos/ && /srv/respaldos/true; echo "rc=$?"
sudo rm /srv/respaldos/true
```

```text
bash: /srv/respaldos/true: Permission denied
rc=126
```

- `126` significa que el archivo se encontró pero no se puede ejecutar.

### Agregar el segundo disco y extender en caliente

Con el volumen ya montado, el segundo disco entero se suma al grupo, y el volumen crece **sin desmontar**:

```bash
sudo pvcreate /dev/sdc
sudo vgextend datos /dev/sdc
sudo vgs datos
df -h /srv/respaldos
sudo lvextend -r -L +1G datos/respaldos
df -h /srv/respaldos
```

```text
  Physical volume "/dev/sdc" successfully created.
  Volume group "datos" successfully extended
  VG    #PV #LV #SN Attr   VSize VFree
  datos   2   1   0 wz--n- 9.99g 7.99g
/dev/mapper/datos-respaldos  2.0G  536K  1.8G   1% /srv/respaldos
  Size of logical volume datos/respaldos changed from 2.00 GiB (512 extents) to 3.00 GiB (768 extents).
Filesystem at /dev/datos/respaldos is mounted on /srv/respaldos; on-line resizing required
…
  Logical volume datos/respaldos successfully resized.
/dev/mapper/datos-respaldos  2.9G  536K  2.8G   1% /srv/respaldos
```

- Aquí el disco `sdc` se usó **entero**, sin partición. LVM acepta las dos formas.
- `-r` agranda también el sistema de archivos. Sin `-r`, el volumen crece pero `df` sigue mostrando el tamaño viejo.
- `on-line resizing required`: ext4 se agranda **montado**, sin detener nada.

!!! warning "Deje espacio libre en `datos`"
    Use `-L +1G`, **no** `-l +100%FREE`. El criterio pide `VFree` mayor que 0, y en la sustentación le pueden pedir que extienda el volumen en vivo. Sin espacio libre, no podría hacerlo.

**Si se quedó sin espacio libre:** ext4 no se puede achicar montado. Vuelva a la instantánea `antes-almacenamiento`, o agregue un tercer disco a la VM y súmelo con `sudo pvcreate /dev/sdd` y `sudo vgextend datos /dev/sdd`.

---

## 4. El script de respaldo

`/usr/local/sbin/respaldo.sh` copia `/srv/proyecto` y `/srv/estado` a una carpeta con la fecha y la hora, y conserva las siete más recientes:

```bash
#!/usr/bin/env bash
# respaldo.sh — instantáneas por fecha de /srv/proyecto y /srv/estado
set -euo pipefail

DESTINO=/srv/respaldos
FECHA=$(date +%Y-%m-%d_%H%M%S)
ULTIMO="$DESTINO/ultimo"
CONSERVAR=7

opciones=(-aAX --delete)
if [[ -d $ULTIMO ]]; then
  opciones+=(--link-dest="$ULTIMO")
fi

mkdir -p "$DESTINO/$FECHA"
rsync "${opciones[@]}" /srv/proyecto /srv/estado "$DESTINO/$FECHA/"
ln -sfn "$FECHA" "$ULTIMO"

find "$DESTINO" -mindepth 1 -maxdepth 1 -type d -name '20*' | sort | head -n -"$CONSERVAR" | xargs -r rm -rf

echo "Respaldo $FECHA listo"
```

| Parte | Qué hace |
| --- | --- |
| `-aAX` | Conserva dueño, grupo, permisos y fechas *(`-a`)*, ACL *(`-A`)* y atributos extendidos *(`-X`)* |
| `--delete` | La instantánea refleja el estado actual: lo que se borró del original no aparece en la copia nueva |
| `--link-dest` | Lo que no cambió desde la instantánea anterior se crea como **enlace duro** al mismo archivo: no ocupa espacio otra vez |
| `ultimo` | Enlace simbólico a la instantánea más reciente |
| `find … head -n -7` | Borra las instantáneas más viejas y deja siete |

```bash
sudo nano /usr/local/sbin/respaldo.sh        # pegar el script
sudo chmod 750 /usr/local/sbin/respaldo.sh
sudo /usr/local/sbin/respaldo.sh
sleep 1
sudo /usr/local/sbin/respaldo.sh
ls -l /srv/respaldos
```

```text
Respaldo 2026-11-23_182049 listo
Respaldo 2026-11-23_182050 listo
drwxr-xr-x 4 root root  4096 Nov 23 18:20 2026-11-23_182049
drwxr-xr-x 4 root root  4096 Nov 23 18:20 2026-11-23_182050
drwx------ 2 root root 16384 Nov 23 18:20 lost+found
lrwxrwxrwx 1 root root    17 Nov 23 18:20 ultimo -> 2026-11-23_182050
```

- El script vive en `/usr/local/sbin` y no en `/srv/respaldos`, porque ese volumen es `noexec`: ahí no se podría ejecutar.
- `lost+found` lo crea `mkfs.ext4`. Es donde `fsck` deja lo que recupera.

### Comprobar que las instantáneas comparten los archivos

```bash
sudo sh -c 'ls -i /srv/respaldos/20*/proyecto/codigo/app.py'
sudo du -sh /srv/respaldos/20*
```

```text
21 /srv/respaldos/2026-11-23_182049/proyecto/codigo/app.py
21 /srv/respaldos/2026-11-23_182050/proyecto/codigo/app.py
52K     /srv/respaldos/2026-11-23_182049
36K     /srv/respaldos/2026-11-23_182050
```

- **El mismo número de inodo** en las dos instantáneas: es un solo archivo con dos nombres *(un enlace duro)*. La segunda instantánea no lo copió otra vez.
- Con archivos tan pequeños, `du` casi no muestra diferencia, porque lo que más ocupa son las carpetas, y esas sí se crean de nuevo. **La prueba es el inodo, no el tamaño.** Para verlo en `du`, cree antes un archivo grande, por ejemplo `sudo -u ana dd if=/dev/urandom of=/srv/proyecto/codigo/datos.bin bs=1M count=20`. La instantánea siguiente sube unos 20M, la otra se queda en decenas de K. Después borre el archivo.
- El `sh -c` hace que el `*` lo expanda root. Su usuario no puede entrar a `codigo/`, y su shell no encontraría los archivos.

---

## 5. Programarlo con un timer

```bash
sudo tee /etc/systemd/system/respaldo.service > /dev/null <<'EOF'
[Unit]
Description=Respaldo de /srv/proyecto y /srv/estado
RequiresMountsFor=/srv/respaldos

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/respaldo.sh
EOF

sudo tee /etc/systemd/system/respaldo.timer > /dev/null <<'EOF'
[Unit]
Description=Respaldo diario

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now respaldo.timer
systemctl list-timers respaldo.timer --no-pager
sudo systemctl start respaldo.service
journalctl -u respaldo.service -n 5 --no-pager
```

```text
Created symlink '/etc/systemd/system/timers.target.wants/respaldo.timer' → '/etc/systemd/system/respaldo.timer'.
NEXT                        LEFT LAST PASSED UNIT           ACTIVATES
Tue 2026-11-24 02:00:00 UTC   7h -         - respaldo.timer respaldo.service
… Starting respaldo.service - Respaldo de /srv/proyecto y /srv/estado...
… respaldo.sh[5205]: Respaldo 2026-11-23_182051 listo
… respaldo.service: Deactivated successfully.
… Finished respaldo.service - Respaldo de /srv/proyecto y /srv/estado.
```

| Línea | Para qué |
| --- | --- |
| `RequiresMountsFor=/srv/respaldos` | Si el volumen no está montado, el respaldo no corre, y así no llena la raíz por error |
| `Type=oneshot` | El servicio corre, termina y no queda vigilado |
| `OnCalendar=*-*-* 02:00:00` | Todos los días a las 02:00 |
| `Persistent=true` | Si la VM estaba apagada a las 02:00, el respaldo corre apenas arranque |

- **Mire la zona horaria** en la línea `NEXT`. Si dice `UTC`, las 02:00 son las 21:00 del día anterior en Colombia. Para usar la hora local: `sudo timedatectl set-timezone America/Bogota`.
- `systemd-analyze calendar '*-*-* 02:00:00'` muestra cuándo será la próxima ejecución.
- **Encienda la VM varios días antes de la entrega.** Así acumula instantáneas de días distintos, y la sección 6 tiene más de una de dónde elegir.

---

## 6. Restaurar un archivo borrado

Es el paso 3 de la demostración. Practíquelo hasta poder hacerlo sin mirar:

```bash
sudo getfacl /srv/proyecto/codigo/app.py > antes.txt
sudo rm /srv/proyecto/codigo/app.py
sudo ls /srv/proyecto/codigo
sudo rsync -aAX /srv/respaldos/ultimo/proyecto/codigo/app.py /srv/proyecto/codigo/
sudo cat /srv/proyecto/codigo/app.py
sudo getfacl /srv/proyecto/codigo/app.py | diff antes.txt - && echo "permisos idénticos"
```

```text
getfacl: Removing leading '/' from absolute path names
print(1)
getfacl: Removing leading '/' from absolute path names
permisos idénticos
```

- El `ls` de la carpeta vacía no imprime nada: el archivo ya no está.
- El aviso `Removing leading '/'` de `getfacl` **no es un error**. Sale por la salida de errores y no entra en `antes.txt`.
- `permisos idénticos` quiere decir que volvieron el dueño, el grupo y la ACL. Restaurar no es solo recuperar el contenido. Sin `-A`, el archivo volvería sin su ACL y el equipo perdería el acceso.
- Use `rsync -aAX` y no `cp`: un `cp` simple deja el archivo a nombre de root y sin ACL.

### Cuando `ultimo` ya no tiene el archivo

`ultimo` apunta a la instantánea **más reciente**. Si el archivo se borró **antes** de ese respaldo, por `--delete` la copia nueva tampoco lo tiene, y hay que sacarlo de una instantánea anterior. Practíquelo con un archivo de prueba:

```bash
sudo -u ana sh -c 'echo "versión 1" > /srv/proyecto/codigo/prueba.txt'
sudo systemctl start respaldo.service
sudo rm /srv/proyecto/codigo/prueba.txt
sleep 1
sudo systemctl start respaldo.service
sudo ls /srv/respaldos/ultimo/proyecto/codigo
sudo sh -c 'ls -l /srv/respaldos/20*/proyecto/codigo/prueba.txt 2>/dev/null'
```

```text
app.py
-rw-rw----+ 1 ana desarrollo 11 Nov 23 18:25 /srv/respaldos/2026-11-23_182530/proyecto/codigo/prueba.txt
```

- `ultimo` ya no tiene `prueba.txt`. La única instantánea que lo tiene es la anterior.
- Restaure desde esa instantánea, con su nombre real:

```bash
sudo rsync -aAX /srv/respaldos/2026-11-23_182530/proyecto/codigo/prueba.txt /srv/proyecto/codigo/
sudo ls -l /srv/proyecto/codigo/prueba.txt
sudo rm /srv/proyecto/codigo/prueba.txt
```

- Si `ana` no existe en su servidor, use cualquier usuario del grupo dueño de `codigo`.
- La fecha de cada carpeta dice **cuándo** se tomó. Antes de restaurar, compare esas fechas con la hora en que se perdió el archivo.

---

## 7. Si el respaldo falla

Un respaldo que falla en silencio es peor que no tener respaldo. Dónde mirar:

```bash
systemctl --failed
systemctl status respaldo.service --no-pager
journalctl -u respaldo.service -n 20 --no-pager
findmnt /srv/respaldos
df -h /srv/respaldos
sudo du -xh --max-depth=1 /srv/respaldos | sort -h
```

| Síntoma | Causa probable |
| --- | --- |
| `Dependency failed` o el servicio no arranca | `/srv/respaldos` no está montado *(`RequiresMountsFor=`)*: revise `findmnt` y `/etc/fstab` |
| El respaldo «funciona» pero `df -h /` crece | Falta la línea de `/etc/fstab`: sin ella no hay unidad de montaje que exigir, y el respaldo escribe en la carpeta `/srv/respaldos` de la raíz |
| `Permission denied` al ejecutar el script | Falta el permiso de ejecución: `sudo chmod 750 /usr/local/sbin/respaldo.sh` |
| `rsync error: … (code 23)` | No existe alguna de las carpetas de origen |
| `No space left on device` o `rsync error: error in file IO (code 11)` | El volumen está lleno: busque con `du` **qué** lo llenó antes de decidir |

- Arregle la causa y vuelva a correrlo con `sudo systemctl start respaldo.service`. Luego confirme que quedó una instantánea nueva con `ls -l /srv/respaldos`.
- **Que el volumen esté lleno no se arregla con extenderlo** si lo que lo llenó no es del respaldo. Primero encuentre qué lo llenó y por qué.

---

## 8. Al repositorio

```text
so-<apellido>/
├── config/
│   ├── fstab-respaldos.txt
│   ├── respaldo.service
│   └── respaldo.timer
└── scripts/
    ├── almacenamiento.sh
    └── respaldo.sh
```

```bash
grep /srv/respaldos /etc/fstab > config/fstab-respaldos.txt   # en la VM; luego cópielo al repositorio
```

- `config/`: las copias de la línea de `fstab` y de las dos unidades.
- `scripts/respaldo.sh`: el script de la sección 4.
- `scripts/almacenamiento.sh`: el volumen en **una VM nueva**, en una sola corrida. Recibe los dos discos y no toca un disco que ya tenga particiones o datos:

```bash
#!/usr/bin/env bash
# almacenamiento.sh — volumen de datos para los respaldos, en una VM nueva
# Uso: sudo ./almacenamiento.sh /dev/sdb /dev/sdc
set -euo pipefail

D1=${1:?Falta el primer disco, por ejemplo /dev/sdb}
D2=${2:?Falta el segundo disco, por ejemplo /dev/sdc}

for d in "$D1" "$D2"; do
  if [[ ! -b $d ]]; then
    echo "$d no es un disco"; exit 1
  fi
  if [[ $(lsblk -n -o NAME "$d" | wc -l) -gt 1 || -n $(lsblk -n -o FSTYPE "$d" | tr -d '[:space:]') ]]; then
    echo "$d ya tiene particiones o datos: no se toca"; exit 1
  fi
done

parted "$D1" --script mklabel gpt mkpart datos 1MiB 100% set 1 lvm on
udevadm settle
P1=$(lsblk -n -o PATH "$D1" | sed -n 2p)

pvcreate "$P1"
pvcreate "$D2"
vgcreate datos "$P1" "$D2"
lvcreate -L 3G -n respaldos datos
mkfs.ext4 -L respaldos /dev/datos/respaldos

mkdir -p /srv/respaldos
UUID=$(blkid -s UUID -o value /dev/datos/respaldos)
if ! grep -q "$UUID" /etc/fstab; then
  echo "UUID=$UUID /srv/respaldos ext4 defaults,nodev,nosuid,noexec 0 2" >> /etc/fstab
fi
systemctl daemon-reload
mount -a
findmnt /srv/respaldos
vgs datos
```

- **No lo corra en su VM actual**: ahí el volumen ya existe, y el script se detiene con `ya tiene particiones o datos`. Es la receta para reconstruir el servidor.
- Crea el volumen directamente con 3G y deja libre el resto de `datos`: el mismo resultado de las secciones 2 y 3.

En el `README.md`, el paso *Discos, LVM, fstab, respaldo.sh, respaldo.service y respaldo.timer* de *Reconstruir desde cero* puede quedar así:

```markdown
5. Almacenamiento y respaldos
   - Agregar a la VM dos discos VDI de 5 GB (apagada).
   - `sudo ./scripts/almacenamiento.sh /dev/sdb /dev/sdc`
   - Copiar `scripts/respaldo.sh` a `/usr/local/sbin/` con permisos 750.
   - Copiar `config/respaldo.service` y `config/respaldo.timer` a `/etc/systemd/system/`,
     y luego `sudo systemctl daemon-reload && sudo systemctl enable --now respaldo.timer`.
   - Restaurar un archivo: `sudo rsync -aAX /srv/respaldos/<instantánea>/proyecto/<ruta> /srv/proyecto/<ruta>`
```

```bash
git add config scripts README.md
git commit -m "feat: volumen de respaldos con LVM, respaldo diario y restauración"
git push
```

---

## 9. Antes de la entrega

Reinicie la VM y compruebe que todo vuelve solo:

```bash
sudo systemctl reboot
```

Al entrar de nuevo con `ssh srv`:

```bash
sudo lvs datos
sudo vgs datos
findmnt -no OPTIONS /srv/respaldos
systemctl list-timers respaldo.timer --no-pager
systemctl is-active estado
ls -l /srv/respaldos
```
