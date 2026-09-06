# Runbook: migrar el boot de Raspberry Pi 4 de microSD a SSD/USB

Procedimiento verificado el 2026-09-05 en un Raspberry Pi 4B (Ubuntu Server 24.04 LTS,
hostname `devops-lab`). Documenta los comandos exactos que funcionaron, en el orden
correcto para reproducirlo desde cero (no el orden cronológico en que los descubrimos,
que incluyó un cuelgue de USB a mitad de camino — ver la sección de troubleshooting).

Todos los comandos se corren desde la laptop, vía `ssh raspberrypi '...'` (ajusta el
alias/host a tu setup). Requiere sudo en la Pi; si no tienes sudo sin contraseña
configurado, quita `-n`/ajusta según pida el password.

## 0. Prerrequisitos

- El disco destino (SSD/USB) está conectado físicamente y **no tiene datos que te importen**
  (todo el proceso lo borra por completo).
- Sabes qué device es (`lsblk`) — en este caso fue `/dev/sda`, 223.6GB.
- Tienes la Raspberry Pi arrancando normalmente desde la microSD.

```bash
ssh raspberrypi 'lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,TRAN'
```

## 1. Identificar el chipset del enclosure USB (evita el problema de UAS de entrada)

Varios adaptadores USB-SATA/NVMe (notablemente los que usan **JMicron JMS578/JMS579**)
tienen incompatibilidad conocida con el driver UAS en el controlador xHCI del Pi 4:
bajo carga de escritura sostenida, el controlador USB puede colapsar por completo
(`xHCI host controller not responding`, `HC died`, el disco desaparece de `lsusb`).

```bash
ssh raspberrypi 'lsusb'
# Busca el Vendor:Product ID del disco, ej: 152d:0578 JMicron JMS578
```

Si el chipset es JMicron (o cualquiera con reportes de problemas de UAS), aplica el
quirk **antes** de clonar, no después:

```bash
ssh raspberrypi '
sudo cp /boot/firmware/cmdline.txt /boot/firmware/cmdline.txt.bak
current=$(cat /boot/firmware/cmdline.txt)
# Reemplaza 152d:0578 por el VID:PID real que viste en lsusb
new="${current} usb-storage.quirks=152d:0578:u"
echo "$new" | sudo tee /boot/firmware/cmdline.txt
'
ssh raspberrypi 'sudo reboot'
```

Espera a que vuelva (~60-90s) y confirma que quedó forzado a `usb-storage` en vez de `uas`:

```bash
ssh raspberrypi 'lsusb -t'
# Debe decir Driver=usb-storage en la línea del disco, no Driver=uas
ssh raspberrypi 'sudo dmesg | grep -i "UAS is ignored"'
# Debe aparecer: "UAS is ignored for this device, using usb-storage instead"
```

## 2. Instalar rpi-clone

```bash
ssh raspberrypi '
cd /tmp && rm -rf rpi-clone && git clone -q https://github.com/billw2/rpi-clone.git
sudo cp rpi-clone/rpi-clone /usr/local/sbin/
sudo cp rpi-clone/rpi-clone-setup /usr/local/sbin/
'
```

## 3. Clonar el sistema completo (en caliente, sin apagar la Pi)

`rpi-clone` reparticiona el disco destino él mismo (copia la estructura boot+root de la
SD y estira la partición root para llenar el resto del disco destino) y sincroniza
archivos con `rsync`. No hace falta particionar el SSD a mano antes.

```bash
ssh raspberrypi '
nohup sudo rpi-clone sda -f -U -L ssd-root > /tmp/rpi-clone.log 2>&1 &
disown
'
```

- `sda` = disco destino (sin el `/dev/`).
- `-f` = forzar reinicializar la tabla de particiones del destino.
- `-U` = modo desatendido, no pide confirmaciones (necesario por SSH no interactivo).
- `-L ssd-root` = etiqueta que le pone a la partición root clonada.
- Se lanza con `nohup ... & disown` para que sobreviva aunque se corte la sesión SSH —
  el clonado de un sistema real tarda varios minutos.

Monitorea el progreso:

```bash
ssh raspberrypi 'pgrep -af rpi-clone; tail -30 /tmp/rpi-clone.log'
```

Busca al final del log una sección `Done with clone to /dev/sda` con el tiempo
transcurrido. En nuestra corrida (~223GB destino, ~3GB usados en origen) tomó 2:15 min.

## 4. Corregir cmdline.txt y fstab del disco clonado (paso que rpi-clone NO hace solo)

`rpi-clone` renombra la partición root del destino (con `-L`) pero **no actualiza**
`cmdline.txt` ni `/etc/fstab` para que apunten a esa nueva etiqueta — se quedan
apuntando a las labels originales de la SD (`LABEL=writable`, `LABEL=system-boot`),
lo que rompe el boot si algún día el SSD es el único disco conectado, o genera
ambigüedad si ambos discos coexisten con la misma label.

Primero identifica los PARTUUID reales del disco clonado (van a ser distintos cada vez):

```bash
ssh raspberrypi 'sudo blkid /dev/sda1 /dev/sda2'
```

Monta ambas particiones y corrige las referencias (reemplaza los PARTUUID de ejemplo
por los que obtuviste arriba):

```bash
ssh raspberrypi '
sudo mkdir -p /mnt/ssd_boot /mnt/ssd_root
sudo mount /dev/sda1 /mnt/ssd_boot
sudo mount /dev/sda2 /mnt/ssd_root

sudo sed -i "s/root=LABEL=writable/root=PARTUUID=<PARTUUID_DE_SDA2>/" /mnt/ssd_boot/cmdline.txt
sudo sed -i "s/LABEL=writable/PARTUUID=<PARTUUID_DE_SDA2>/; s/LABEL=system-boot/PARTUUID=<PARTUUID_DE_SDA1>/" /mnt/ssd_root/etc/fstab

cat /mnt/ssd_boot/cmdline.txt
cat /mnt/ssd_root/etc/fstab

sudo umount /mnt/ssd_boot /mnt/ssd_root
'
```

## 5. Configurar el orden de arranque del EEPROM (USB primero, SD como fallback automático)

Esto es lo que hace que el Pi realmente arranque desde el SSD, pero sin perder la
microSD como "botón de pánico": si el SSD fallara, el Pi cae solo a la SD.

```bash
ssh raspberrypi '
sudo rpi-eeprom-config > /tmp/eeprom-orig.txt   # backup del config actual
cp /tmp/eeprom-orig.txt /tmp/eeprom-new.txt
echo "BOOT_ORDER=0xf14" >> /tmp/eeprom-new.txt   # rightmost nibble = intento 1: 4=USB, luego 1=SD, luego f=repetir
sudo rpi-eeprom-config --apply /tmp/eeprom-new.txt
'
ssh raspberrypi 'sudo reboot'
```

En nuestra prueba, un solo reinicio bastó para que el nuevo `BOOT_ORDER` quedara
grabado y aplicado (el Pi ya arrancó desde el SSD en ese mismo reinicio).

## 6. Verificar que arrancó desde el SSD

```bash
ssh raspberrypi '
findmnt /
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINT
df -h /
sudo rpi-eeprom-update
systemctl --failed --no-pager
'
```

`findmnt /` debe mostrar `SOURCE=/dev/sda2` (o el disco que hayas usado). La microSD
(`mmcblk0`) debe aparecer en `lsblk` pero sin punto de montaje — intacta, sin tocar.

## 7. Benchmark: cuánto se gana realmente con el SSD

Medido el 2026-09-05 con `dd`, misma metodología en los dos discos: archivo de
prueba del mismo tamaño, `oflag=direct`/`iflag=direct` para saltar la caché de
página (mide velocidad real del dispositivo, no de la RAM), y `drop_caches` antes
de cada lectura para no leer datos que quedaron cacheados de la escritura previa.

```bash
ssh raspberrypi '
cd <directorio_de_prueba>   # ej: /home/antonio (SSD) o /mnt/sd_root (microSD montada)

# Escritura secuencial (512MB, bloques de 1MB)
dd if=/dev/zero of=./benchfile.bin bs=1M count=512 oflag=direct conv=fdatasync

# Lectura secuencial (mismo archivo, cache limpio antes de leer)
sync
echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null
dd if=./benchfile.bin of=/dev/null bs=1M iflag=direct
rm -f ./benchfile.bin

# Escritura pequeña con sync forzado (4KB x 8192 = 32MB) — simula el patrón de
# escritura de Docker/k3s/Prometheus (muchas escrituras chicas y síncronas)
dd if=/dev/zero of=./benchfile_small.bin bs=4k count=8192 oflag=direct,dsync
rm -f ./benchfile_small.bin
'
```

Para la microSD, hay que montarla primero (estaba desconectada del sistema de
archivos tras la migración):

```bash
ssh raspberrypi '
sudo mkdir -p /mnt/sd_root
sudo mount /dev/mmcblk0p2 /mnt/sd_root
sudo chown antonio:antonio /mnt/sd_root
# ... correr los mismos comandos de arriba con cd /mnt/sd_root ...
sudo umount /mnt/sd_root
'
```

### Resultados

| Prueba | microSD | SSD (USB, sin UAS) | Mejora |
|---|---|---|---|
| Escritura secuencial (bloques 1MB) | 33.7 MB/s | 192 MB/s | ~5.7x |
| Lectura secuencial (bloques 1MB) | 42.9 MB/s | 276 MB/s | ~6.4x |
| Escritura pequeña síncrona (bloques 4KB) | 0.6 MB/s | 2.0 MB/s | ~3.4x |

La prueba de escritura pequeña síncrona es la más representativa del uso real
(contenedores, bases de datos de métricas, logs) y es donde las microSD sufren más.

**Nota sobre el techo de velocidad del SSD**: el SSD es un Crucial BX500 (SATA III,
6Gb/s nominal, ~550MB/s reales de tope) — spec real, no de marketing. Pero en este
setup no se alcanza ese máximo porque hay dos cuellos de botella entre medio: el
puente USB-SATA JMicron JMS578 (limitado a USB 3.0/3.1 Gen1, ~5Gbps) y, sobre todo,
que forzamos ese puente a modo `usb-storage` (sin UAS) en el paso 1 para evitar el
crash del controlador — ese modo tiene más overhead por comando que UAS, a cambio
de estabilidad. Es un trade-off consciente: preferimos 192-276 MB/s estables sobre
arriesgar otro cuelgue del xHCI por unos MB/s más.

## Troubleshooting: `partprobe`/re-lectura de tabla de particiones colgada

Si un `parted`/`partprobe`/paso de `rpi-clone` que "relee la tabla de particiones" se
queda colgado (proceso en estado `D`, no avanza), y el enclosure resulta ser JMicron
(u otro con historial de problemas UAS): es casi seguro el mismo problema de UAS
manifestándose de forma más leve. La tabla de particiones en el disco puede ya estar
correcta (verificar con `sudo fdisk -l /dev/sda`) aunque el proceso que la relee esté
colgado — en ese caso matar el proceso colgado (`sudo kill -9 <pid>`) es seguro. Pero
la solución de raíz es aplicar el quirk de `usb-storage` del paso 1 **antes** de
volver a intentar cualquier escritura pesada al disco.

## Notas

- Los PARTUUID cambian en cada clonado — nunca los copies de una corrida anterior.
- Si vuelves a correr `rpi-clone -f` sobre un disco que ya tiene un clon previo,
  vuelve a repetir el paso 4 (la corrección de `cmdline.txt`/`fstab` no persiste
  entre corridas de `rpi-clone`).
- Este runbook asume que quieres el SSD entero como una sola partición de sistema
  (sin partición separada para backups) — ver la decisión correspondiente en las
  memorias del proyecto.
