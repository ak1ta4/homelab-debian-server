# Paso 5: disco de datos con btrfs

> **Versiones usadas:** Debian 13.7, núcleo 6.12.117, btrfs-progs 6.14-1. Disco: SSD externo de 2 TB por USB 3.2 Gen 2 (10 Gb/s, controlador `uas`).

## 1. Qué vamos a hacer y por qué

**En sencillo:** el portátil tiene un disco pequeño para el sistema y un SSD externo grande para los datos: archivos y fotos. Lo preparamos con un formato moderno (btrfs) que detecta si un archivo se ha corrompido, comprime lo que se puede comprimir y permite hacer "fotos" instantáneas de una carpeta (*snapshots*). Lo dejamos montado siempre en el mismo sitio y, si un día no está conectado, el equipo arranca igualmente.

**Técnicamente:**
- Tabla GPT con una partición y btrfs con los valores por defecto de btrfs-progs 6.14: datos `single`, metadatos y sistema `DUP`.
- Subvolúmenes `archivos`, `fotos` y `snapshots` en la raíz del sistema de archivos.
- Montaje de la raíz en `/srv/datos` desde `/etc/fstab`, con `noatime`, `compress=zstd:1` y `nofail`.
- Sin cifrado ([ADR 0001](../decisiones/0001-sin-cifrado-de-disco.md)). Las bases de datos y los secretos se quedan en el NVMe.

## 2. Conceptos previos

- **btrfs:** sistema de archivos con sumas de verificación (detecta datos corruptos), compresión transparente y *copy-on-write* (al cambiar un archivo escribe en un sitio nuevo, y por eso los snapshots salen casi gratis).
- **Subvolumen:** una carpeta especial de btrfs que puede tener sus propios snapshots. Un snapshot **no es recursivo**: no incluye los subvolúmenes que haya dentro.
- **Snapshot:** una copia instantánea de un subvolumen, en el mismo disco. **No es una copia de seguridad**: si el disco muere, mueren con él. Las copias van al PC con restic (Fase 5).
- **Perfil `DUP`:** guarda dos copias de los metadatos en el mismo disco. Protege frente a sectores dañados, no frente a la muerte del disco.
- **`fstab`:** la lista de sistemas de archivos que se montan al arrancar.
- **`nofail`:** el arranque no espera al disco ni falla si no está.

## 3. Requisitos y estado inicial

- `btrfs-progs` sin instalar.
- El SSD en NTFS, montado por GNOME en `/media/<USUARIO_ADMIN>/…`. **Comprueba el contenido antes de nada:** aquí tenía cuatro carpetas vacías y metadatos de Windows (4 KB en total).
- TRIM por USB: `lsblk -D` con `DISC-GRAN` distinto de 0 (aquí, `4K`).
- Ahorro de energía del USB desactivado para el disco: `power/control` = `on` en su dispositivo de `/sys/bus/usb/devices/`.

## 4. Pasos

En el servidor, con `<USUARIO_ADMIN>`. Pega los comandos **de uno en uno**.

### 4.1 Revisar el contenido

```bash
ls -la "/media/<USUARIO_ADMIN>/Nuevo vol"
```
```bash
du -sh "/media/<USUARIO_ADMIN>/Nuevo vol"
```
Si hay algo que conservar, cópialo antes. **El formateo no se puede deshacer.**

### 4.2 Comprobar que el USB es estable

```bash
sudo journalctl -k -b -1 | grep -ciE "uas|usb 2-1.*reset"
```
```bash
sudo journalctl -k -b 0 | grep -E "uas_|usb 2-1:|sd 0:0:0:0|sda" | tail -15
```
`usb 2-1` es el puerto del disco aquí: sale en `lsblk` y en el registro al detectarlo. **Bien:** pocos avisos, y en el arranque actual solo la detección del disco. **Mal:** `reset`, `uas_eh_abort_handler` o `I/O error` repetidos.

Cuidado con filtrar solo por `uas`: también aparece en las listas de módulos que el núcleo imprime con cualquier aviso interno (ver Errores frecuentes).

### 4.3 Instalar btrfs-progs

```bash
sudo apt update
```
```bash
sudo apt install btrfs-progs
```
Revisa que no aparezca una sección de paquetes que se eliminarán.

### 4.4 Identificar el disco sin ambigüedad

```bash
sudo umount "/media/<USUARIO_ADMIN>/Nuevo vol"
```
```bash
D=/dev/disk/by-id/<ID_DISCO_DATOS>
```
El nombre de `/dev/disk/by-id/` incluye el modelo y el número de serie: no cambia entre arranques, a diferencia de `/dev/sda`. Haz el resto del paso **en la misma terminal**, porque la variable `D` solo existe en ella.
```bash
lsblk -o NAME,MODEL,SERIAL,SIZE,TRAN,MOUNTPOINTS "$D"
```
**Resultado:** El modelo, el número de serie y el tamaño esperados, conexión `usb` y **sin punto de montaje**.

### 4.5 Formatear (borra el disco)

```bash
sudo wipefs -a "$D-part1"
```
```bash
sudo wipefs -a "$D"
```
```bash
sudo parted -s "$D" mklabel gpt mkpart datos btrfs 1MiB 100%
```
```bash
sudo udevadm settle
```
```bash
sudo mkfs.btrfs -L datos "$D-part1"
```
`mkfs.btrfs` está en `/usr/sbin`, que no está en la ruta de un usuario normal: sin `sudo` sale `command not found`. **Resultado:** La salida dice `Label: datos`, `Data: single` y `Metadata: DUP`.

### 4.6 Subvolúmenes

```bash
sudo mkdir -p /srv/datos
```
```bash
sudo mount -o noatime,compress=zstd:1 LABEL=datos /srv/datos
```
```bash
sudo btrfs subvolume create /srv/datos/archivos
```
```bash
sudo btrfs subvolume create /srv/datos/fotos
```
```bash
sudo btrfs subvolume create /srv/datos/snapshots
```
```bash
sudo btrfs subvolume list /srv/datos
```
**Resultado:** Tres líneas: `path archivos`, `path fotos` y `path snapshots`.
```bash
sudo umount /srv/datos
```

Se monta la raíz del disco y no cada subvolumen por separado, porque las opciones de montaje como `compress` se aplican a todo el sistema de archivos igualmente (`man 5 btrfs`). Una sola línea en `fstab` y rutas sin `@` para los contenedores.

### 4.7 Montaje en el arranque

UUID del sistema de archivos nuevo:
```bash
lsblk -o NAME,UUID "$D"
```
```bash
sudo cp -a /etc/fstab /etc/fstab.bak-AAAAMMDD
```
Añade al final de `/etc/fstab` (con `sudo nano /etc/fstab`, o preparándolo en un archivo y añadiéndolo con `sudo tee -a`; **sin la `-a`, `tee` sustituye el archivo entero**):
```
# Homelab, Fase 1, paso 5: disco de datos (SSD por USB, btrfs, etiqueta "datos").
# nofail: si el disco no está, el equipo arranca igual, con SSH.
UUID=<UUID_DATOS> /srv/datos btrfs noatime,compress=zstd:1,nofail 0 0
```
- `noatime`: no escribe la fecha de último acceso cada vez que se lee un archivo.
- `compress=zstd:1`: compresión rápida. Los archivos que no se comprimen bien, como las fotos, se detectan y se dejan tal cual.
- `nofail`: imprescindible en un servidor sin pantalla. Sin esa opción, si falta el disco, el arranque puede acabar en el modo de emergencia, **sin red ni SSH**.
- El último `0`: no se revisa al arrancar. btrfs no usa `fsck` en el arranque.

```bash
diff /etc/fstab.bak-AAAAMMDD /etc/fstab
```
**Resultado:** Solo líneas con `>` (añadidas).
```bash
sudo findmnt --verify
```
**Resultado:** `0 parse errors, 0 errors`. El aviso de que systemd usa la versión antigua desaparece con el comando siguiente.
```bash
sudo systemctl daemon-reload
```
```bash
sudo mount /srv/datos
```

### 4.8 Comprobar

Sin `sudo`:
```bash
findmnt /srv/datos
```
**Resultado:** `btrfs` con `noatime,compress=zstd:1,ssd,discard=async,space_cache=v2,subvolid=5`. El núcleo detecta solo el SSD y activa el TRIM asíncrono.
```bash
btrfs filesystem df /srv/datos
```
**Resultado:** `Data, single`, `System, DUP` y `Metadata, DUP`.
```bash
systemctl show srv-datos.mount -p WantedBy -p Before
```
**Resultado:** `WantedBy=local-fs.target` y **sin** `local-fs.target` en `Before`: el arranque no espera al disco.

Con `sudo`:
```bash
echo prueba | sudo tee /srv/datos/archivos/prueba.txt
```
```bash
sync
```
```bash
sudo cat /srv/datos/archivos/prueba.txt
```
```bash
sudo rm /srv/datos/archivos/prueba.txt
```
```bash
sudo btrfs device stats /srv/datos
```
**Resultado:** Los cinco contadores a `0`.

## 5. Verificación

- [x] Formato btrfs con datos `single` y metadatos `DUP`, etiqueta `datos`.
- [x] Subvolúmenes `archivos`, `fotos` y `snapshots`.
- [x] Montado en `/srv/datos` con `noatime`, `compress=zstd:1`, TRIM asíncrono y `nofail` (el arranque no espera por él).
- [x] Escritura y lectura; contadores de errores a 0.
- [ ] Arranque con el disco conectado: se monta solo. Se comprueba en la sesión en persona del paso 4.
- [ ] Arranque con el disco **desconectado**: el equipo vuelve y se entra por SSH. También en esa sesión.
- [ ] Programa de snapshots (snapper o btrbk): cuando haya datos y servicios.
- [ ] Dueños y permisos de `archivos` y `fotos`: los fijará cada servicio (Fases 2 a 5). De momento son de root.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `mkfs.btrfs: command not found` | `/usr/sbin` no está en la ruta de un usuario normal | Usa `sudo`, que sí lo encuentra |
| `bash: /srv/datos/…: No such file or directory` tras crear subvolúmenes | La línea se partió al pegarla y el último argumento quedó suelto | Un subvolumen por comando |
| Un filtro por `uas` en el registro saca líneas enormes con muchos nombres de módulos | Son la lista "Modules linked in" de un aviso del núcleo, que incluye `uas` | Filtra por `uas_`, `usb 2-1:` o `sda`. Para saber qué aviso fue: `grep -E "WARN\|BUG\|RIP:"` |
| `btrfs device stats` da `Operation not permitted` | Necesita root | Con `sudo` |
| Tras formatear, GNOME monta el disco en `/media/…` | Automontaje de GNOME | `sudo umount` antes de seguir. Con la línea en `fstab` ya no lo hace |

## 7. Cómo revertirlo

Quitar el montaje (los datos siguen en el disco):
```bash
sudo umount /srv/datos
```
```bash
sudo cp -a /etc/fstab.bak-AAAAMMDD /etc/fstab
```
```bash
sudo systemctl daemon-reload
```
El formateo **no se puede deshacer**: por eso se revisa el contenido primero.

## 8. Seguridad

**Mitiga:**
- Corrupción silenciosa: btrfs comprueba las sumas de verificación al leer, y `btrfs device stats` lleva la cuenta de los errores.
- Sectores dañados en los metadatos: perfil `DUP`.
- Arranque bloqueado por el disco: `nofail`.
- Formatear el disco equivocado: nombre de `/dev/disk/by-id/` con número de serie y comprobación con `lsblk` antes de borrar.

**Queda:**
- Sin cifrado: con el disco en la mano se lee todo ([ADR 0001](../decisiones/0001-sin-cifrado-de-disco.md)). Por eso no lleva secretos.
- Un solo disco: si muere, se pierde lo que haya. Los snapshots no protegen de eso; las copias con restic al PC, sí (Fase 5).
- Desconectar el disco en caliente con servicios escribiendo puede dejar archivos a medias. btrfs no se corrompe por ello gracias al *copy-on-write*, pero lo último escrito puede perderse.
- `btrfs scrub` periódico (lee todo y comprueba las sumas) y avisos de `device stats`: en la monitorización de la Fase 5.

## 9. Para la entrevista

**En 30 segundos:** "Preparé el SSD de datos con btrfs: metadatos duplicados, compresión zstd y subvolúmenes para archivos, fotos y snapshots. Para formatear usé el nombre de `/dev/disk/by-id` con el número de serie y lo comprobé con `lsblk` antes de borrar. En `fstab` usé el UUID con `nofail`, que en un servidor sin pantalla evita que un disco USB ausente deje el arranque en el modo de emergencia, sin SSH. Lo validé con `findmnt --verify`, montándolo en vivo y comprobando con `systemctl show` que el arranque no espera por él."

**Preguntas posibles:**
- *¿Por qué btrfs y no ext4?* Por las sumas de verificación de los datos, la compresión y los snapshots baratos, que servirán para las copias de Immich y Syncthing. Para un solo disco de datos es estable. Lo que no se usa es RAID 5/6 de btrfs.
- *¿Un snapshot es una copia de seguridad?* No: está en el mismo disco. Sirve para volver atrás tras un borrado o un error. La copia de seguridad va a otro equipo.
- *¿Por qué `nofail`?* Sin esa opción, el montaje es obligatorio para `local-fs.target`, y si el disco no aparece, el arranque se queda esperando y falla. En un servidor sin pantalla eso significa ir en persona.
- *¿Por qué UUID y no `/dev/sda1`?* Porque los nombres `sdX` dependen del orden de detección y pueden cambiar. El UUID es del sistema de archivos.
- *¿Por qué `zstd:1`?* Comprime casi tanto como los niveles altos en documentos, y gasta mucha menos CPU. btrfs deja sin comprimir lo que no se comprime bien.

## Fuentes

- `man mkfs.btrfs`, `man 5 btrfs` y `man btrfs-subvolume` de btrfs-progs 6.14.
- [Documentación de btrfs](https://btrfs.readthedocs.io/): mkfs.btrfs, Administration (opciones de montaje) y Subvolumes.
- [systemd.mount](https://www.freedesktop.org/software/systemd/man/latest/systemd.mount.html): `nofail` y `x-systemd.device-timeout`.
