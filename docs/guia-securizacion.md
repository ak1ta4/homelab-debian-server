# Guía de securización: qué se aplica, qué se adapta y qué se descarta

Referencia: [How To Secure A Linux Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server), de imthenachoman, bajo licencia [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). Este documento resume con palabras propias qué se hizo con cada sección. La guía se toma como punto de partida, no como script: su autor desaconseja copiar sin entender, y avisa de que sus fragmentos no verifican el resultado y de que hay secciones a medio escribir.

**Estados:** *Aplicada* (tal cual), *Adaptada* (con cambios justificados), *Pendiente* (con la fase prevista), *Descartada* (con motivo), *Evaluar* (decidir con prueba y reversión).

| Sección | Estado | Motivo y dónde |
|---|---|---|
| Before You Start (principios, distribución, instalación) | Leída | Informativa. Debian 13 ya estaba instalado |
| Ansible playbooks | Descartada | Se quiere cada paso entendido y documentado a mano |
| **SSH Public/Private Keys** | Adaptada | Ed25519 con frase de paso. Sin `ssh-copy-id`: no se permitió nunca entrar con contraseña, así que la clave se instala a mano antes del servidor. [Fase 1, paso 2](01-base-y-endurecimiento/02-ssh-de-emergencia.md) |
| **Create SSH Group For AllowGroups** | Aplicada | `sshusers`, solo con el administrador. El usuario del agente no entra |
| **Secure `/etc/ssh/sshd_config`** | Adaptada | Archivo propio en `sshd_config.d/`. `MaxAuthTries 3` y `MaxSessions 4` (la guía, 2 y 2) para dar margen. `UseDNS`, `Compression` y `TCPKeepAlive` con los valores de OpenSSH, sin cambios: no aportan aquí. Sin `ListenAddress`: lo limita el firewall. Algoritmos modernos en `11-homelab-algoritmos.conf`, sin avisos en ssh-audit ([paso 9](01-base-y-endurecimiento/09-resto-de-la-base.md)). Termius queda sin acceso hasta revisarlo |
| **Remove Short Diffie-Hellman Keys** | Descartada (paso 9) | Con los `KexAlgorithms` del paso 9 no se ofrece Diffie-Hellman de campo finito, así que `moduli` no se usa. (El filtro de la guía, `>= 3071`, es correcto: el archivo guarda los bits menos uno) |
| **2FA/MFA for SSH** | Adaptada | Sin `nullok`, con `AuthenticationMethods publickey,keyboard-interactive` y sin `common-auth` en PAM, para que nunca pida la contraseña de Unix |
| Limit Who Can Use sudo | Aplicada de serie | Debian ya limita `sudo` al grupo `sudo`, que solo tiene a `<USUARIO_ADMIN>`. Sin cambios en `sudoers`. El agente nunca entra |
| Limit Who Can Use su | Aplicada (paso 9) | `pam_wheel.so group=sudo` en `/etc/pam.d/su` |
| Run applications in a sandbox with FireJail | Descartada | Los servicios van en contenedores y el escritorio es de uso ocasional |
| NTP Client | Adaptada (paso 9) | chrony con NTS (hora autenticada) y solo fuentes NTS de cuatro operadores, en lugar de `systemd-timesyncd` |
| Securing /proc (`hidepid`) | Descartada | El autor avisa de que puede romper sistemas con systemd; aquí arriesga GNOME y el escritorio remoto |
| Force Accounts To Use Secure Passwords | Adaptada (paso 9) | `libpam-pwquality` con `minlen = 12` en `pwquality.conf.d/`. Sin reglas de tipos de carácter: la longitud pesa más |
| Automatic Security Updates and Alerts | Adaptada (paso 6) | `unattended-upgrades` con los orígenes por defecto de Debian: seguridad y versiones de la estable. Reinicio automático sobre las 6:00 solo si hace falta ([ADR 0006](decisiones/0006-reinicio-e-inicio-de-sesion-automaticos.md)). **Pendiente** (Fase 5): los avisos por correo |
| More Secure Random Entropy Pool (WIP) | Descartada (paso 9) | Desde el núcleo 5.6, `/dev/random` no se bloquea una vez iniciado el generador; con el 6.12 y una CPU con `RDRAND`, `haveged` o `rng-tools` no aportan |
| Add Panic/Secondary/Fake password | Descartada | Su ejemplo es destructivo (borra carpetas personales) y toca la autenticación del sistema |
| Firewall With UFW | Adaptada (paso 7) | Entrada denegada por defecto; SSH, Sunshine y mDNS solo desde `<RED_LAN>`, sin limitar por interfaz. Activado con desactivación automática a los 5 minutos. Salida abierta para el sistema y filtrada para el agente con una tabla nftables propia ([ADR 0007](decisiones/0007-filtro-de-salida-del-agente.md)) |
| Docker And UFW | Adaptada (paso 8) | Docker se salta UFW en los puertos publicados: por defecto se publica en `127.0.0.1` (`"ip"` en `daemon.json`), y lo que deba verse desde casa se publica a propósito y se limita en `DOCKER-USER` ([ADR 0008](decisiones/0008-docker-y-ufw.md)). **Pendiente** (Fase 8): auditoría desde otro equipo |
| PSAD | Descartada | Solapa a Fail2ban y aporta poco sin exposición a internet |
| **Fail2Ban** | Adaptada | Solo `sshd`, leyendo el journal (Debian 13 no tiene `auth.log`), con nftables y excluyendo el PC de rescate |
| CrowdSec | Descartada | Solapa a Fail2ban |
| AIDE (WIP) | Pendiente (Fase 5) | Solo rutas del sistema; base de referencia al final de la fase |
| ClamAV (WIP) | Pendiente (Fase 5) | Solo la carpeta de descargas, bajo demanda y sin demonio |
| Rkhunter (WIP) | Pendiente (Fase 5) | Con `rkhunter.conf.local` y excepciones justificadas |
| chkrootkit (WIP) | Descartada | Solapa a rkhunter |
| logwatch | Pendiente (Fase 5) | Resumen diario por la cuenta de correo dedicada |
| ss | Aplicada | En la auditoría ([Fase 0](00-auditoria.md)) y en la validación final (Fase 8) |
| Lynis | Aplicada (paso 9) | De 72 a 76, sin avisos. Los informes se quedan en el servidor |
| OSSEC | Descartada | Se instala desde código fuente, exige mantenimiento y solapa AIDE, rkhunter y Fail2ban |
| Linux Kernel sysctl Hardening | Adaptada (paso 9) | 14 valores en `90-homelab.conf`, uno a uno. Sin tocar `ip_forward` (Docker), `sysrq`, `modules_disabled` ni los espacios de nombres de usuario; `rp_filter` flexible por Docker y Tailscale |
| Password Protect GRUB | Descartada (paso 9) | Sin cifrado de disco ([ADR 0001](decisiones/0001-sin-cifrado-de-disco.md)), quien tenga el portátil delante arranca desde un USB y lee el disco igualmente. Solo añadiría riesgo al arranque desatendido |
| Disable Root Login | Descartada (paso 9) | Root ya no entra por SSH y `su` queda limitado al grupo `sudo`. Bloquear la cuenta quitaría el modo de rescate en consola |
| Change Default umask | Descartada | Sin causa concreta; un `umask` global puede romper servicios |
| Orphaned Software | Descartada | `deborphan` solo ante una causa concreta |
| The Simple way with MSMTP | Descartada tal cual | Guarda credenciales en texto plano y contiene errores. Se adaptará el correo en la Fase 5 |
| Gmail and Exim4 As MTA With Implicit TLS | Adaptar (Fase 5) | Cuenta dedicada y contraseña de aplicación en `/etc/homelab/secrets/` |
| Separate iptables Log File | Descartada (paso 9) | UFW registra en el journal, y Fail2ban también lee de ahí. La gestión de registros se revisa en la Fase 5 |
| needrestart y protocolos sin uso | Añadidas (paso 9) | Sugerencias de Lynis: `needrestart` solo en modo aviso y `dccp`, `sctp`, `rds` y `tipc` bloqueados en `modprobe.d` |
