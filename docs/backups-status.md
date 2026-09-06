# Estado de backups — SSD devops-lab

## Qué existe hoy

Todos los backups viven en `/var/backups/pi-images/` en la Pi (no en la laptop).

| Archivo | Tamaño | Qué es |
|---|---|---|
| `microsd-pre-ssd-migration-2026-09-05-rootfs.tar.gz` | 1.7GB | Snapshot único de la microSD original, tomado post-migración antes de liberar la tarjeta. Histórico, no se repite. |
| `devops-lab-ssd-root-2026-09-05.tar.gz` | 1.5GB | Partición root (ext4) del SSD en vivo. |
| `devops-lab-boot-firmware-2026-09-05.tar.gz` | 165MB | Partición boot (FAT32, `/boot/firmware`): `cmdline.txt`, `config.txt`, kernel, initrd. |

**Los dos últimos van siempre juntos.** El SSD tiene dos particiones con filesystems
distintos (FAT32 boot + ext4 root); `tar --one-file-system` no cruza esa frontera,
así que un solo tar del root deja fuera `cmdline.txt` (donde vive el quirk crítico
`usb-storage.quirks=152d:0578:u`) y el resto del boot. Sin el segundo tar, el sistema
restaurado no arranca.

## Comando de backup usado

```bash
sudo tar -czf /var/backups/pi-images/devops-lab-ssd-root-<fecha>.tar.gz \
  --exclude='var/backups' --exclude='proc' --exclude='sys' --exclude='dev' \
  --exclude='run' --exclude='tmp' --exclude='mnt' --exclude='media' \
  --one-file-system -C / .

sudo tar -czf /var/backups/pi-images/devops-lab-boot-firmware-<fecha>.tar.gz \
  -C /boot/firmware .
```

**Trampa importante:** con `-C / .`, tar registra las rutas como `./var/...`, así que
los patrones de `--exclude` deben ser **relativos sin `/` inicial**. Con rutas
absolutas (`/var/backups/pi-images`) no hacen match, y como el propio archivo de
salida se escribe dentro del árbol que se está empaquetando, tar termina
incluyéndose a sí mismo mientras crece — un bucle de retroalimentación. Pasó en la
práctica: el archivo llegó a 6.2GB (de un contenido real de ~1.6GB) antes de matar
el proceso. Cualquier script futuro debe usar exclude relativos, o mejor,
escribir el archivo de salida fuera del árbol de origen y moverlo después.

## Prueba de restore realizada (2026-09-05)

Se validó que ambos tars juntos reconstruyen un sistema completo y coherente,
usando una imagen de disco loop en el propio SSD (sin tocar el sistema en vivo,
sin necesitar la microSD física):

1. Imagen de 5.5GB con tabla de particiones MBR idéntica a la real (boot FAT32
   512MB + root ext4).
2. Restauración de ambos tars en las particiones correspondientes.
3. Corrección de PARTUUID en `cmdline.txt`/`fstab` (los tars conservan los
   PARTUUID originales del SSD real, hay que apuntarlos a los del loop device
   nuevo — mismo paso que ya exige el runbook de migración SD→SSD).
4. Validación por chroot: SO, hostname, usuario `antonio` con sudoers NOPASSWD,
   638 paquetes, `dpkg --audit` limpio, kernel/initrd/SSH host keys presentes.
5. `fsck` limpio en ambas particiones restauradas.
6. Cleanup completo (loop device liberado, imagen borrada).

**Limitación encontrada:** `tar --exclude` también excluye el directorio vacío en
sí, no solo su contenido. Un restore real necesita recrear `dev/ proc/ sys/ run/
tmp/ mnt/ media/` con `mkdir` antes de montar/chrootear — no vienen en el backup.

## Mejoras futuras (no implementadas, quedaron en pausa)

- **Script de backup recurrente con retención de 2** (tar-based, generando ambos
  tars, borrando el par más viejo al crear uno nuevo). Es sencillo pero significa
  reimplementar a mano lógica de retención que herramientas ya resuelven.
- **Evaluar restic o borgbackup** en vez de tar a mano: deduplicación, cifrado,
  retención nativa (`restic forget --keep-last N`), y soporte directo para
  sincronizar a Google Drive vía `rclone` (la mejora que Antonio mencionó como
  posible a futuro). Más alineado con lo que se ve en la industria que un script
  de tar casero.
- **Replantear el alcance del backup una vez que Ansible/k3s existan**: si el SO
  y su configuración quedan versionados como código (Ansible), la recuperación
  ante desastre deja de depender de un backup completo del sistema — se
  reflashea y se corre el playbook. En ese escenario el backup relevante pasa a
  ser solo el *estado que no se puede regenerar* (volúmenes de Docker/k3s,
  configs runtime), más pequeño y con más sentido para restic/borg.
- Decisión pendiente: prioridad baja frente a Ansible/Docker/k3s/CI-CD/monitoreo,
  que aportan más señal en un portafolio DevOps/SRE.
