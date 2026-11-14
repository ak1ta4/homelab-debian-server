# Guía de securización: qué se aplica, qué se adapta y qué se descarta

Referencia: [How To Secure A Linux Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server), de imthenachoman, bajo licencia [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). Este documento resume con palabras propias qué se hizo con cada sección. La guía se toma como punto de partida, no como script: su autor desaconseja copiar sin entender, y avisa de que sus fragmentos no verifican el resultado y de que hay secciones a medio escribir.

**Estados:** *Aplicada* (tal cual), *Adaptada* (con cambios justificados), *Pendiente* (con la fase prevista), *Descartada* (con motivo), *Evaluar* (decidir con prueba y reversión).

| Sección | Estado | Motivo y dónde |
|---|---|---|
| Before You Start (principios, distribución, instalación) | Leída | Informativa. Debian 13 ya estaba instalado |
| Ansible playbooks | Descartada | Se quiere cada paso entendido y documentado a mano |
| **SSH Public/Private Keys** | Adaptada | Ed25519 con frase de paso. Sin `ssh-copy-id`: no se permitió nunca entrar con contraseña, así que la clave se instala a mano antes del servidor. [Fase 1, paso 2](01-base-y-endurecimiento/02-ssh-de-emergencia.md) |
| **Create SSH Group For AllowGroups** | Aplicada | `sshusers`, solo con el administrador. El usuario del agente no entra |
| **Secure `/etc/ssh/sshd_config`** | Adaptada | Archivo propio en `sshd_config.d/`. `MaxAuthTries 3` y `MaxSessions 4` (la guía, 2 y 2) para dar margen. `UseDNS`, `Compression` y `TCPKeepAlive` con los valores de OpenSSH, sin cambios: no aportan aquí. Sin `ListenAddress`: lo limita el firewall. **Pendiente** (paso 9): restringir algoritmos al perfil moderno tras probar los clientes |
| **Remove Short Diffie-Hellman Keys** | Pendiente (paso 9) | Filtrar `/etc/ssh/moduli` a 3072 bits o más (`>= 3072`; el filtro de la guía usa `3071`) |
| **2FA/MFA for SSH** | Adaptada | Sin `nullok`, con `AuthenticationMethods publickey,keyboard-interactive` y sin `common-auth` en PAM, para que nunca pida la contraseña de Unix |
| Limit Who Can Use sudo | Pendiente (paso 9) | Con `visudo` y un rescate probado. El agente nunca entra |
| Limit Who Can Use su | Pendiente (paso 9) | Igual |
| Run applications in a sandbox with FireJail | Descartada | Los servicios van en contenedores y el escritorio es de uso ocasional |
| NTP Client | Adaptada | `systemd-timesyncd` (Debian 13), activo y sincronizado. Se valorará chrony con NTS (hora autenticada) en el paso 9 |
| Securing /proc (`hidepid`) | Descartada | El autor avisa de que puede romper sistemas con systemd; aquí arriesga GNOME y el escritorio remoto |
| Force Accounts To Use Secure Passwords | Pendiente (paso 9) | — |
| Automatic Security Updates and Alerts | Adaptada (paso 6) | `unattended-upgrades` con los orígenes por defecto de Debian: seguridad y versiones de la estable. Reinicio automático sobre las 6:00 solo si hace falta ([ADR 0006](decisiones/0006-reinicio-e-inicio-de-sesion-automaticos.md)). **Pendiente** (Fase 5): los avisos por correo |
| More Secure Random Entropy Pool (WIP) | Evaluar (paso 9) | Comprobar si aporta algo con el núcleo 6.12 antes de instalar nada |
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
| Lynis | Pendiente (paso 9) | Antes y después, con puntuación y hallazgos saneados |
| OSSEC | Descartada | Se instala desde código fuente, exige mantenimiento y solapa AIDE, rkhunter y Fail2ban |
| Linux Kernel sysctl Hardening | Evaluar (paso 9) | Uno a uno, con justificación, prueba y reversión |
| Password Protect GRUB | Evaluar (paso 9) | Solo si protege la edición sin impedir el arranque desatendido |
| Disable Root Login | Evaluar (paso 9) | Solo con un rescate probado. Root ya no puede entrar por SSH |
| Change Default umask | Descartada | Sin causa concreta; un `umask` global puede romper servicios |
| Orphaned Software | Descartada | `deborphan` solo ante una causa concreta |
| The Simple way with MSMTP | Descartada tal cual | Guarda credenciales en texto plano y contiene errores. Se adaptará el correo en la Fase 5 |
| Gmail and Exim4 As MTA With Implicit TLS | Adaptar (Fase 5) | Cuenta dedicada y contraseña de aplicación en `/etc/homelab/secrets/` |
| Separate iptables Log File | Evaluar (paso 7) | — |
