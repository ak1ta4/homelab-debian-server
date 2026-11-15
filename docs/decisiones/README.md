# Decisiones (ADR)

Cada decisión importante tiene un documento corto con su **contexto**, las **opciones** consideradas, la **decisión** y sus **consecuencias**. Si una decisión cambia, se escribe un ADR nuevo que la sustituye, y el antiguo se marca como reemplazado.

| ADR | Decisión | Estado |
|---|---|---|
| [0001](0001-sin-cifrado-de-disco.md) | No cifrar los discos ni reinstalar Debian | Aceptada |
| [0002](0002-permisos-del-agente-de-ia.md) | Agente de IA en una cuenta sin privilegios, sin sandbox por ahora | Aceptada |
| [0003](0003-ssh-clave-totp-fail2ban.md) | SSH de emergencia con clave + TOTP y Fail2ban, sin cambiar el puerto | Aceptada |
| [0004](0004-tapar-ssh-del-router.md) | Tapar el SSH del router con un reenvío a una IP sin uso | Aceptada |
| [0005](0005-gitleaks-local-y-en-github.md) | gitleaks con configuración de root fuera del repositorio y workflow propio | Aceptada |
| [0006](0006-reinicio-e-inicio-de-sesion-automaticos.md) | Reinicio automático tras actualizar e inicio de sesión automático con bloqueo | Aceptada |
| [0007](0007-filtro-de-salida-del-agente.md) | Filtro de salida del agente: sin LAN ni servicios locales, solo DNS y HTTPS | Aceptada |
| [0008](0008-docker-y-ufw.md) | Docker publica por defecto solo en el propio equipo; lo de la LAN, a propósito | Aceptada |
| [0009](0009-tailscale.md) | Acceso desde fuera con Tailscale, sin tocar el router | Aceptada |

**Pendientes, en su fase:**
- Syncthing frente a Nextcloud.
- Conexión directa, Peer Relay o DERP.
- LiveSync.
- Caddy con DNS-01.
- `network_mode: host`.
- NVIDIA solo para cómputo.
- `rest-server` en modo `append-only`.
- Moshi con Mosh frente a SSH con tmux.
- Homepage frente a Homarr.
- Beszel y Dozzle frente a Grafana o Netdata.
