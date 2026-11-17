# Progreso

Registro para retomar el trabajo en una sesión nueva. Fechas en formato AAAA-MM-DD.

## Fase 0: auditoría (terminada)

Guía: [`00-auditoria.md`](00-auditoria.md).

- Auditoría de solo lectura del sistema, discos, gráficos, red y exposición.
- Decisiones:
  - Sin cifrado de disco: riesgo físico aceptado por el dueño ([ADR 0001](decisiones/0001-sin-cifrado-de-disco.md)).
  - Agente de IA en una cuenta sin privilegios desde antes de la Fase 1, sin sandbox por ahora ([ADR 0002](decisiones/0002-permisos-del-agente-de-ia.md)).
- Revisión del router (2026-11-02 y 2026-11-08):
  - Sin CGNAT. DMZ, reenvíos y UPnP desactivados.
  - El SSH propio del router respondía desde internet y su panel no permite cerrarlo. Se tapó con un reenvío del TCP 22 hacia `<IP_SUMIDERO>` ([ADR 0004](decisiones/0004-tapar-ssh-del-router.md)). Comprobado desde datos móviles.

## Fase 1: base y endurecimiento (en curso)

Guía: [`01-base-y-endurecimiento/`](01-base-y-endurecimiento/README.md).

| Paso | Estado |
|---|---|
| 1. Reservas DHCP del servidor y del PC | Hecho (2026-11-08). Se verifica en el próximo reinicio |
| 2. SSH de emergencia | Hecho (2026-11-08) |
| 3. Repositorio y documentación | Hecho (2026-11-09) |
| 4. Portátil como servidor | En curso (2026-11-14) |
| 5. Disco de datos | Hecho (2026-11-14). Falta la prueba de arranque |
| 6. Actualizaciones de seguridad | Hecho (2026-11-14). Falta la prueba de reinicio |
| 7. Firewall | Hecho (2026-11-14). Falta la prueba de reinicio |
| 8. Docker | Hecho (2026-11-15). Falta la prueba de reinicio |
| 9. Resto de la base | Hecho (2026-11-15). Falta la prueba de reinicio y Termius |

### Paso 2: SSH de emergencia

- OpenSSH 10.0 de Debian 13 con configuración propia en `/etc/ssh/sshd_config.d/10-homelab.conf`: solo el grupo `sshusers`, sin root, `AuthenticationMethods publickey,keyboard-interactive` y sin reenvíos.
- TOTP con `libpam-google-authenticator`. `/etc/pam.d/sshd` no incluye `common-auth`, así que nunca pide la contraseña de Unix.
- Fail2ban 1.1.0, solo para `sshd`: 5 fallos en 10 min → 1 h, excluye `<IP_PC>`.
- **Comprobado:**
  - Entra con clave + código.
  - Se rechazan la contraseña, root, el código incorrecto y la conexión sin clave.
  - Un código de emergencia sirve.
  - Fail2ban bloquea y desbloquea, y no bloquea el PC.

### Paso 3: repositorio

- `git` de Debian y gitleaks 8.30.1 (binario oficial verificado con su checksum).
- Hooks `pre-commit`, `pre-applypatch` y `pre-push` con una configuración de gitleaks propiedad de root, fuera del repositorio. Ignoran `gitleaks:allow`. **Comprobado:** un token falso bloquea el commit.
- El agente entrega parches; `<USUARIO_ADMIN>` los revisa, los aplica con `git am` y los sube con una clave de despliegue con frase de paso.
- Workflow de gitleaks en GitHub propio, en verde. Sustituye a `gitleaks-action`, que no revisaba nada en la primera subida ([lecciones](lecciones-aprendidas.md#gitleaks-action-y-la-primera-subida)).
- Guías de la Fase 0, de los pasos 1 a 3, cinco ADR, la tabla de la guía de securización, lecciones aprendidas y preguntas de entrevista.

### Paso 4: portátil como servidor

- Hecho y comprobado por D-Bus, sin suspender:
  - Suspensión e hibernación desactivadas (`sleep.conf.d/10-homelab.conf`).
  - La tapa no hace nada (`logind.conf.d/10-homelab.conf`), aplicado con `reload`.
  - UPower: `PowerOff` al 10 % (umbrales 25/15/10). No admite archivos parciales: se editó `UPower.conf` con copia `.bak-20261114`.
- Pendiente:
  - BIOS (2026-11-15): límite de carga activado; no existe encender tras un corte (la batería hace de SAI). Revisar en unos días que la carga baja del 100 %.
  - Pruebas físicas aplazadas por el dueño: tapa cerrada, cargador desenchufado y reinicio con la tapa cerrada.
- Moonlight: con la **tapa entreabierta**, por decisión del dueño (2026-11-15). Se descartan la pantalla virtual y el enchufe HDMI falso.

### Paso 5: disco de datos

- SSD de 2 TB por USB con btrfs (btrfs-progs 6.14): datos `single`, metadatos `DUP`, etiqueta `datos`.
- Subvolúmenes `archivos`, `fotos` y `snapshots`. Se monta la raíz en `/srv/datos` desde `fstab` por UUID, con `noatime,compress=zstd:1,nofail`. Copia en `/etc/fstab.bak-20261114`.
- **Comprobado:**
  - `findmnt --verify` sin errores.
  - Montado con TRIM asíncrono.
  - El arranque no espera por él (`nofail`).
  - Escritura y lectura correctas, y `device stats` a 0.
  - El USB, estable en el registro del núcleo.
- Pendiente, en la sesión en persona: arrancar con el disco conectado y desconectado. El programa de snapshots y los permisos, cuando lleguen los servicios.

### Paso 6: actualizaciones de seguridad

- `unattended-upgrades` 2.12 con los orígenes por defecto de Debian. Reinicio automático entre las 6:00 y las 7:00, solo si hace falta ([ADR 0006](decisiones/0006-reinicio-e-inicio-de-sesion-automaticos.md)).
- GNOME Software sin descargas automáticas.
- Inicio de sesión automático en GDM con bloqueo de pantalla a los 3 segundos. Copia en `/etc/gdm3/daemon.conf.bak-20261114`.
- **Comprobado:**
  - La prueba en seco acepta solo los orígenes de Debian.
  - El temporizador está programado.
  - `loginctl lock-session` bloquea la sesión.
- Pendiente:
  - Primera ejecución real (registro en `/var/log/unattended-upgrades/`).
  - En el primer reinicio: que entre solo, que bloquee la pantalla y que Moonlight funcione.

### Paso 7: firewall

- UFW 0.36.2: entrada denegada por defecto; SSH, Sunshine (con su panel) y mDNS solo desde `<RED_LAN>`. Activado con desactivación automática a los 5 minutos.
- Filtro de salida del agente ([ADR 0007](decisiones/0007-filtro-de-salida-del-agente.md)): tabla nftables `inet agente_salida` con `homelab-agente-salida.service`. Sin LAN ni servicios locales; a internet solo DNS y HTTPS.
- **Comprobado:**
  - Con UFW activo, SSH nuevo y Moonlight funcionan.
  - El agente resuelve DNS y llega a Anthropic y Debian por HTTPS.
  - El router, el panel de Sunshine, el SSH local y el puerto 80 se rechazan.
- Pendiente: comprobar tras un reinicio que los dos siguen activos.

### Paso 8: Docker

- Docker Engine 29.9.3 y Compose 5.7.1 desde el repositorio oficial. La clave se comprobó con una segunda descarga por otro camino.
- `daemon.json`: `local`, `live-restore` y `"ip": "127.0.0.1"` ([ADR 0008](decisiones/0008-docker-y-ufw.md)). Siempre con `sudo`, con el grupo `docker` vacío. Entra en las actualizaciones automáticas.
- **Comprobado:**
  - Un puerto publicado sin IP escucha solo en `127.0.0.1`.
  - El filtro de salida del agente sigue funcionando con Docker activo.

### Paso 9: resto de la base

- Lynis 3.1.4 "antes": índice 72, sin avisos, 46 sugerencias. ssh-audit 3.3.0 "antes" guardado.
- Algoritmos SSH (2026-11-15) en `sshd_config.d/11-homelab-algoritmos.conf`: sin curvas NIST, SHA-1 ni MAC sin *encrypt-then-MAC*. Probado con desactivación automática a los 5 minutos.
  - **Comprobado:** el PC entra, y `ssh-audit -l warn` no da avisos ni fallos.
  - **Termius (iPhone y iPad) no entra:** negocia la clave ECDSA y, con solo `ed25519` y RSA, corta antes de autenticarse, incluso tras borrar el *known host*. El dueño decide aplicarlo igualmente y dejar Termius para más adelante.
- Base (2026-11-15): `su` solo para el grupo `sudo` (`pam_wheel`, copia en `/etc/pam.d/su.bak-20261115`); `libpam-pwquality` con `minlen = 12`; `needrestart` 3.11 solo en modo aviso (`l`); `dccp`, `sctp`, `rds` y `tipc` sin poder cargarse. **Comprobado:** `modprobe -n -v dccp` da `install /bin/false` y `needrestart` no ve nada pendiente.
- Hora (2026-11-15): chrony 4.6.1 con NTS en sustitución de `systemd-timesyncd`. Solo fuentes NTS (`sources.d/homelab-nts.sources`: Netnod, PTB ×2, TimeNL y Cloudflare); el pool de Debian y las fuentes por DHCP, comentados (copia `chrony.conf.bak-20261115`). **Comprobado:** las 5 en modo NTS con 8 cookies, sincronizado y `Leap status: Normal`.
- Núcleo (2026-11-15): `/etc/sysctl.d/90-homelab.conf` con `kptr_restrict=2`, `ldisc_autoload=0`, `protected_fifos=2`, `ptrace_scope=1`, `bpf_jit_harden=2`, `kexec_load_disabled=1`, sin `send_redirects` ni `secure_redirects`, `log_martians=1` y `rp_filter=2` (flexible, por Docker y Tailscale). Sin tocar `ip_forward`, `sysrq`, `modules_disabled` ni los espacios de nombres de usuario. **Comprobado:** SSH nuevo, Moonlight, Jellyfin y la salida del agente funcionan.
- Lynis "después": **76** (antes 72), sin avisos, 41 sugerencias. Guía: [`09-resto-de-la-base.md`](01-base-y-endurecimiento/09-resto-de-la-base.md).

## Adelantado: Jellyfin (2026-11-15)

Pedido por el dueño fuera de orden. Guía: [`stacks/jellyfin/`](../stacks/jellyfin/README.md).

- Imagen oficial `12.4` con Compose, usuario `jellyfin` sin capacidades y vídeos en solo lectura desde `/srv/datos/Videos` (subvolumen propio).
- Puerto 8096 solo en `<IP_SERVIDOR>`, limitado a `<RED_LAN>` en `DOCKER-USER` por `homelab-jellyfin-lan.service`.
- VA-API en la AMD integrada.
- **Comprobado:**
  - `vainfo` abre la AMD dentro del contenedor.
  - Una película se ve en la tele con reproducción directa.
  - Al bajar la calidad a 8 Mbps, transcodifica sin cortes.
- Pendiente: la prueba de reinicio.
- Cortes en la tele por Wi-Fi (2026-11-15): el servidor no tenía cola y el disco lee a 861 MB/s, pero la tele solo recibía ~5 Mbit/s (el router mostraba su enlace a `866/6` con buena señal). Por cable recibe 15-19 Mbit/s y va bien. En el router, el 5 GHz pasó de `Auto` (canal 100, DFS, 160 MHz) a canal 36 con 80 MHz; no bastó por sí solo. Para volver al Wi-Fi: reiniciar el router y, si se repite, probar a quitar funciones como *Beam Forming* o *MLO* una a una.

## Fase 2: escritorio remoto (2026-11-15)

Guía: [`02-escritorio-remoto.md`](02-escritorio-remoto.md).

- Sunshine 2026.914.233613 desde su repositorio oficial (Cloudsmith), con un pin que solo acepta `sunshine`, y dentro de las actualizaciones automáticas. Clave comprobada con la firma del `InRelease` y comparando el `.deb` con el de GitHub (misma SHA-256).
- Configuración correcta: `kms`, `vaapi` y `renderD128`. Emparejados solo el PC y el iPad.
- Moonlight con la tapa entreabierta (decisión del dueño).
- El aviso de `nvidia-drm` se repitió a las 19:41 al conectar Moonlight: lo provoca Sunshine al abrir todas las pantallas por KMS. Sin efecto visible; drivers sin tocar.
- Pendiente:
  - Reiniciar Sunshine para borrar el aviso de arranque del panel (`Unable to find display or encoder during startup`).
  - Comprobar Moonlight tras un reinicio del servidor.

## Fase 3: acceso desde fuera (2026-11-15)

Guía: [`03-acceso-remoto.md`](03-acceso-remoto.md). Decisión: [ADR 0009](decisiones/0009-tailscale.md).

- Tailscale 1.104.7 desde su repositorio (clave comprobada en tres sitios), con pin y en las actualizaciones automáticas. Servidor con `tag:server` y sin el DNS de Tailscale.
- Política: los dispositivos del dueño solo llegan al servidor, por TCP 22, 8096, 47984, 47989 y 48010 y UDP 47998-48000.
- UFW: esos puertos solo en `tailscale0`, más UDP 41641. Jellyfin en `0.0.0.0` (IPv4), con `RETURN` para la tailnet en `DOCKER-USER`, y la tailnet en sus **Redes locales**.
- Incidente: al conectar, Tailscale puso su DNS y el agente se quedó sin conexión (su filtro bloquea `100.64.0.0/10`). Lo corrigió el dueño con `tailscale set --accept-dns=false`.
- **Comprobado** desde el iPhone con datos: Jellyfin y Moonlight (iPhone emparejado en Sunshine). Por el repetidor de Madrid, 40 ms.
- Pendiente:
  - SSH por la tailnet (Termius, o el PC con Tailscale).
  - Tailscale en el PC y el iPad.
  - La prueba de reinicio.
  - Tailnet Lock, como mejora.

## Fase 4: DNS con AdGuard (2026-11-15 y 2026-11-16)

Guía: [`04-dns.md`](04-dns.md). Decisión: [ADR 0010](decisiones/0010-dns-de-casa-con-adguard.md). Solo AdGuard por decisión del dueño; Caddy, HTTPS y el dominio, aplazados.

- AdGuard Home `v0.107.80` en Docker ([stack](../stacks/adguard/README.md)): usuario `adguard`, sin capacidades salvo `NET_BIND_SERVICE` (el binario lleva `setcap`), DNS en 5353 y panel en 3000 dentro; publicado `53→5353` y `3000` en `0.0.0.0`.
- `DOCKER-USER` (`homelab-adguard-lan.service`): DNS desde `<RED_LAN>` y la tailnet por `tailscale0`; panel solo desde `<RED_LAN>`. Con `--ctstate DNAT`, para no bloquear la salida del contenedor.
- DoH a Quad9 y Cloudflare, bootstrap `9.9.9.9` y `1.1.1.1`, solicitudes paralelas y DNSSEC.
- El servidor fija sus DNS en los del operador (`ipv4.ignore-auto-dns`), por el filtro del agente.
- Router (2026-11-16): pool DHCP de casa con `<IP_SERVIDOR>` y `9.9.9.9` de respaldo. Los pools del decodificador (`[IAL]`) y de Telefónica (`TEF_IOT`), sin tocar.
- Tailscale (2026-11-16): `udp:53` y `tcp:53` en la política, el servidor como nameserver global con *Override DNS servers*, y `--snat-subnet-routes=false`.
- **Comprobado:**
  - Desde el PC, `ad.doubleclick.net` da `0.0.0.0`.
  - Cuatro dispositivos de casa en el panel, con dominios bloqueados.
  - Sin IPv6 global en la LAN.
  - El iPhone con datos y Tailscale llega con su IP `100.x`.
  - El servidor y el agente siguen resolviendo.
  - Jellyfin sigue funcionando por Tailscale tras `--snat-subnet-routes=false`.
- Pendiente: la prueba de reinicio.

## Fase 5: servicios (en curso, desde 2026-11-16)

Guía: [`05-servicios/`](05-servicios/README.md). Plan aprobado por el dueño:
- **5.0** Dominio y Caddy (DNS-01 con Cloudflare). Aplazado hasta que el dueño compre el dominio.
- **5.1** Monitorización: Beszel, Dozzle y Homepage como panel para acceder a todo. Incluye lo que quedó de la Fase 4: hoja imprimible para volver al DNS del router, registros de AdGuard 7 días y filtros más fuertes.
- **5.2** Alertas por correo dedicado y bot de Telegram (avisos y consultas de una lista cerrada). La parte de hablar con una IA va a la Fase 7.
- **5.3** Nube, **adelantada por el dueño**: Syncthing del PC al servidor, y Samba para el iPhone y el iPad. File Browser se descarta: está archivado desde el 2026-09-01.
- **5.4** Traer archivos del PC (Wake-on-LAN). **5.5** Obsidian (necesita HTTPS). **5.6** Immich, después de la nube. **5.7** Descargas con ClamAV. **5.8** Auditoría: logwatch, rkhunter y AIDE.

### 5.1 Monitorización y panel (hecho, 2026-11-17)

Guías: [`01-panel.md`](05-servicios/01-panel.md), [`01-monitorizacion.md`](05-servicios/01-monitorizacion.md) y [Fase 4, 4.8](04-dns.md#48-filtros-de-amenazas). Decisiones: [ADR 0013](decisiones/0013-panel-con-homepage.md) y [ADR 0014](decisiones/0014-beszel-y-dozzle.md).

- **Panel:** Homepage `v2.8.0` ([stack](../stacks/homepage/README.md)): usuario `homepage`, `cap_drop: ALL`, sin socket de Docker y el disco de datos en `:ro`. Estilo gruvbox, marcadores y el tiempo.
  - Nombre `homelab.home.arpa` en AdGuard, con una reescritura con `$client` para `<RED_LAN>` y otra para la tailnet. En el servidor, una línea en `/etc/hosts` hacia `127.0.0.1`.
  - Red de Docker `panel`, solo con las webs: puntos de estado de seis servicios.
  - Datos de Immich (clave con solo `server.statistics`), Jellyfin (clave de administrador) y AdGuard (usuario `panel` con contraseña aleatoria). Claves en archivos `400`, leídas con `HOMEPAGE_FILE_*`. Syncthing sin datos: su clave da control total.
- **Beszel `0.21.2`** ([stack](../stacks/beszel/README.md)): hub sin root; agente como servicio con el usuario `beszel`, sin el grupo `docker` (el `.deb` oficial lo añadía), sin puertos y solo con la GPU AMD.
- **Dozzle `v11.4.3`** ([stack](../stacks/dozzle/README.md)): con contraseña, sin acciones ni consola, detrás de un proxy del socket con `POST=0` en una red interna.
- **AdGuard:** listas de HaGeZi (amenazas, Pro y DNS cifrados) y registro de 7 días. Memoria de 92 a 427 MB.
- `DOCKER-USER`: 80, 8090 y 8080 desde `<RED_LAN>` y la tailnet. En la política: `"tcp:80"`, `"tcp:8384"`, `"tcp:8090"` y `"tcp:8080"`.
- **Comprobado:**
  - El nombre da una sola IP desde el PC; el panel abre en el PC, en el servidor y en el iPhone con datos. Al principio no cargaba fuera: faltaba `"tcp:80"`.
  - Seis puntos verdes y datos en las tres tarjetas.
  - Hub como `beszel-hub`; agente como `beszel`, sin puertos; sistema en verde.
  - El proxy de Dozzle responde `403` a un `POST`; Dozzle ve los 11 contenedores.
  - `dns.google` bloqueado; las webs de siempre abren.
- Pendiente:
  - Beszel y Dozzle desde el iPhone fuera de casa.
  - La prueba de reinicio.
  - Alertas de Beszel a Telegram (paso 5.2).

### 5.3 Nube (hecho, 2026-11-16 y 2026-11-17)

Guía: [`05-servicios/03-nube.md`](05-servicios/03-nube.md). Decisión: [ADR 0011](decisiones/0011-nube-con-syncthing-y-samba.md).

- Syncthing `2.1.7` en Docker ([stack](../stacks/syncthing/README.md)):
  - Usuario `syncthing` sin capacidades.
  - Datos en `/srv/datos/archivos/pc`; configuración en `/srv/syncthing/var`.
  - `homelab-syncthing-lan.service`: el 22000 (TCP y UDP), desde `<RED_LAN>` y la tailnet; el panel 8384, solo desde `<IP_PC>` y la tailnet.
  - Paneles con contraseña y HTTPS, sin relays, descubrimiento ni NAT traversal.
- En el PC: zip oficial `2.1.7` (hash comprobado), acceso directo en Inicio y sin reglas de firewall. Llama a `tcp://<IP_SERVIDOR>:22000`.
- Carpetas Documentos, Imágenes y Vídeos: el PC envía solamente y el servidor recibe solamente, con versiones escalonadas de 365 días. Documentos con sus [patrones a ignorar](../stacks/syncthing/documentos.stignore), revisados con el dueño.
- Samba 4.22 en el host ([configuración](../stacks/samba/README.md)):
  - `pc` en solo lectura y `subidas` con escritura, para un usuario sin shell;
  - solo SMB3, sin NetBIOS ni invitados, y `vfs_fruit` para el iPhone;
  - UFW 445 desde `<RED_LAN>` y `tailscale0`, y `"tcp:445"` en la política de Tailscale.
- Incidente: los recomendados de Samba metieron `winbind` en PAM y en `nsswitch`. Se quitaron; `sudo` y SSH, probados.
- **Comprobado:**
  - 13.756 archivos y 76,8 GiB al día en los dos lados, en el disco de datos, sin secretos ni carpetas excluidas.
  - Panel del servidor: `403` sin credenciales.
  - Windows con SMB 3.1.1 cifrado; `pc`, solo lectura.
  - iPhone en casa y por la tailnet: sube y descarga.
- Pendiente: la prueba de reinicio. `UMASK=027` no se aplica (sin efecto, por el `750` de la carpeta).

### 5.6 Fotos con Immich (2026-11-17)

Guía: [`05-servicios/06-fotos.md`](05-servicios/06-fotos.md). Decisión: [ADR 0012](decisiones/0012-fotos-con-immich.md).

- Immich `v3.4.1` en Docker ([stack](../stacks/immich/README.md)):
  - Los cuatro contenedores como `immich`, `cap_drop: ALL` y `no-new-privileges`.
  - Base de datos en el NVMe; lo subido, en `/srv/datos/immich`.
- Imágenes del PC como biblioteca externa `:ro` (con `group_add` de `syncthing`), sin la carpeta de Manga. IA en la CPU.
- `homelab-immich-lan.service`: el 2283 desde `<RED_LAN>` y la tailnet. `"tcp:2283"` en la política de Tailscale.
- iPhone:
  - app con subida manual y cambio automático de URL según el wifi;
  - Tailscale con VPN On Demand (desconecta en el wifi de casa).
- **Comprobado:**
  - `pong` y los cuatro contenedores con el usuario `immich`.
  - `immich` no puede escribir en las imágenes del PC.
  - La biblioteca está escaneada.
  - App en casa sin VPN y fuera por la tailnet.
- Primer procesado de la IA: carga de 12 a 26 con 12 hilos (a la 01:48 del 2026-11-17). El dueño lo deja con la concurrencia por defecto.
- A las 02:43, lo que más cargaba era `ffmpeg` (cerca del 970 % de CPU), convirtiendo los vídeos HEVC del iPhone. A las 15:39, carga de 0,1: el procesado ha terminado (las colas, sin mirar).
- Pendiente:
  - Que terminen los trabajos.
  - La prueba de reinicio.
  - No borrar fotos del iPhone hasta la Fase 6.

## Pendiente de verificar

- Las reservas DHCP, en el próximo reinicio de cada equipo.
- Que la regla del router que tapa su SSH sigue en su sitio: en el calendario periódico.
- Aviso del núcleo en `nvidia-drm` (550.163.01) el 2026-11-14 a las 20:16: `WARNING` en `nv_drm_revoke_modeset_permission`, dos veces seguidas, sin caída. Sin tocar drivers. Se repite al conectar Moonlight (ver Fase 2); solo se investiga si coincide con cortes o cuelgues.

## Próximo

- Sesión en persona:
  - Pruebas físicas del paso 4.
  - Arranque con el disco de datos conectado y desconectado.
- Termius sin SSH tras el cambio de algoritmos.
