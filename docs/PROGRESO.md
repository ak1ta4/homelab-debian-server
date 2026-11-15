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
| 9. Resto de la base | En curso (2026-11-15) |

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
  - BIOS: límite de carga y encendido tras un corte.
  - Pruebas físicas aplazadas por el dueño: tapa cerrada, cargador desenchufado y reinicio con la tapa cerrada (incluye Moonlight).

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

### Paso 9: resto de la base (en curso)

- Lynis 3.1.4 "antes": índice 72, sin avisos, 46 sugerencias. ssh-audit 3.3.0 "antes" guardado.
- Algoritmos SSH (2026-11-15): probados con desactivación automática a los 5 minutos. El PC entró; **Termius en el iPhone no**. Se deshizo solo y el iPhone vuelve a entrar. Falta saber qué algoritmo necesita Termius.

## Adelantado: Jellyfin (2026-11-15)

Pedido por el dueño fuera de orden. Guía: [`stacks/jellyfin/`](../stacks/jellyfin/README.md).

- Imagen oficial `12.4` con Compose, usuario `jellyfin` sin capacidades y vídeos en solo lectura desde `/srv/datos/Videos` (subvolumen propio).
- Puerto 8096 solo en `<IP_SERVIDOR>`, limitado a `<RED_LAN>` en `DOCKER-USER` por `homelab-jellyfin-lan.service`.
- VA-API en la AMD integrada.
- **Comprobado:**
  - `vainfo` abre la AMD dentro del contenedor.
  - Una película se ve en la tele con reproducción directa.
- Pendiente:
  - Una transcodificación real con VA-API.
  - La prueba de reinicio.

## Pendiente de verificar

- Las reservas DHCP, en el próximo reinicio de cada equipo.
- Que la regla del router que tapa su SSH sigue en su sitio: en el calendario periódico.
- Aviso del núcleo en `nvidia-drm` (550.163.01) el 2026-11-14 a las 20:16: `WARNING` en `nv_drm_revoke_modeset_permission`, dos veces seguidas, sin caída. Sin tocar drivers. Si se repite o coincide con cortes de Moonlight, se investiga en la Fase 2.

## Próximo

- Sesión en persona:
  - BIOS: límite de carga y encendido tras un corte.
  - Pruebas físicas del paso 4.
  - Arranque con el disco de datos conectado y desconectado.
- Mientras tanto: paso 9, el resto de la base y Lynis.
