# Paso 9: resto de la base y Lynis

> **Versiones usadas:** Debian 13.7, núcleo 6.12.117, OpenSSH 10.0p2, Lynis 3.1.4, ssh-audit 3.3.0, libpam-pwquality 1.4.5, needrestart 3.11, chrony 4.6.1.

## 1. Qué vamos a hacer y por qué

**En sencillo:** repasamos el servidor con un programa de auditoría (Lynis), que da una nota y una lista de mejoras. Aplicamos las que tienen sentido aquí, una a una y comprobando que nada se rompe, y volvemos a pasar la auditoría para comparar.

**Técnicamente:**
1. Lynis y ssh-audit "antes".
2. SSH solo con algoritmos modernos.
3. `su` limitado al grupo `sudo`, longitud mínima de contraseña, `needrestart` y cuatro protocolos de red bloqueados.
4. Hora con chrony y NTS.
5. Ajustes del núcleo con `sysctl`.
6. Lynis "después".

## 2. Conceptos previos

- **Lynis:** audita el sistema sin cambiar nada y da un índice de endurecimiento (0-100). Sus sugerencias son genéricas: se aplican solo las que encajan.
- **ssh-audit:** revisa qué algoritmos ofrece el servidor SSH y marca los débiles.
- **Algoritmos SSH:** el intercambio de claves (KEX), la clave del servidor (*host key*), el cifrado y la integridad (MAC). Cliente y servidor eligen el primero que tengan en común.
- **PAM:** las piezas que deciden cómo se autentica cada programa (`su`, `login`, `passwd`…).
- **NTS (Network Time Security):** hora por NTP pero autenticada con TLS. Sin ella, quien controle la red puede adelantar o atrasar el reloj, y con él la validez de certificados y códigos TOTP.
- **`sysctl`:** parámetros del núcleo que se cambian en caliente y se fijan en `/etc/sysctl.d/`.

## 3. Requisitos y estado inicial

- Pasos 2 a 8 hechos. El PC entra por SSH: es el acceso de rescate para el punto 2.
- Lynis desde Debian (`sudo apt install lynis`), ssh-audit desde Debian (`sudo apt install ssh-audit`).

## 4. Pasos

En el servidor, con `<USUARIO_ADMIN>`. Cada punto se hizo con un script revisado con `sudo cat` antes de ejecutarlo; aquí van sus pasos.

### 4.1 Auditoría inicial

```bash
sudo lynis audit system --quiet
```
```bash
sudo grep hardening_index /var/log/lynis-report.dat
```
Resultado: **72**, sin avisos, 46 sugerencias. El informe tiene datos de la casa: no se sube al repositorio.

```bash
ssh-audit -l warn localhost
```
Marcaba los intercambios y claves con curvas NIST (`ecdh-sha2-nistp*`, `ecdsa-sha2-nistp256`), los MAC con SHA-1 o de 64 bits y los que no son *encrypt-then-MAC*.

### 4.2 Algoritmos SSH

Antes de empezar: una sesión SSH abierta desde el PC, como respaldo.

`/etc/ssh/sshd_config.d/11-homelab-algoritmos.conf`:
```
KexAlgorithms mlkem768x25519-sha256,sntrup761x25519-sha512,sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com
```
Cada directiva, en una sola línea.

```bash
sudo sshd -t
```
Sin salida = correcto. Si falla, borra el archivo y no sigas.

Vuelta atrás automática a los 5 minutos, por si se pierde el acceso:
```bash
sudo systemd-run --on-active=5m --unit=ssh-algoritmos-rescate /bin/sh -c "rm -f /etc/ssh/sshd_config.d/11-homelab-algoritmos.conf && systemctl reload ssh"
```
```bash
sudo systemctl reload ssh
```
Prueba una conexión **nueva** desde cada cliente. Si un cliente avisa de que la clave del servidor ha cambiado, compara la huella con la de `ed25519`:
```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```
Si todo va bien, cancela la vuelta atrás:
```bash
sudo systemctl stop ssh-algoritmos-rescate.timer
```
```bash
ssh-audit -l warn localhost
```
**Resultado:** Sin avisos ni fallos; solo una nota informativa sobre Terrapin (`chacha20-poly1305`), que ya protege el intercambio estricto de OpenSSH.

**Termius** (iPhone y iPad) no entra con esta configuración: negocia la clave ECDSA y corta antes de autenticarse, incluso tras borrar su *known host*. El dueño decidió aplicarlo igualmente y revisarlo más adelante. Para devolverle el acceso bastaría con añadir `ecdsa-sha2-nistp256` al final de `HostKeyAlgorithms`.

### 4.3 `su`, contraseñas, needrestart y protocolos

**`su` solo para el grupo `sudo`.** En `/etc/pam.d/su` (copia `su.bak-AAAAMMDD`), debajo de la línea comentada `# auth       required   pam_wheel.so`:
```
auth       required   pam_wheel.so group=sudo
```
root sigue pudiendo usar `su`, porque `pam_rootok` va antes. No afecta a `sudo` ni a SSH.

**Contraseñas nuevas de 12 caracteres o más:**
```bash
sudo apt install libpam-pwquality
```
`pam-auth-update` lo añade solo a `common-password`. `/etc/security/pwquality.conf.d/10-homelab.conf`:
```
minlen = 12
```
Solo se aplica al cambiar una contraseña.

**needrestart**, para saber qué servicios siguen usando bibliotecas viejas tras una actualización:
```bash
sudo apt install needrestart
```
`/etc/needrestart/conf.d/10-homelab.conf`:
```perl
$nrconf{restart} = 'l';
```
`l` solo lista; no reinicia nada por su cuenta, para que una actualización no corte SSH ni el escritorio.

**Protocolos sin uso.** `/etc/modprobe.d/homelab-protocolos.conf`:
```
install dccp /bin/false
install sctp /bin/false
install rds /bin/false
install tipc /bin/false
```
```bash
modprobe -n -v dccp
```
**Resultado:** `install /bin/false`.

### 4.4 Hora con chrony y NTS

```bash
sudo apt install chrony
```
Desinstala `systemd-timesyncd`: los dos son incompatibles.

En `/etc/chrony/chrony.conf` (copia `chrony.conf.bak-AAAAMMDD`) se comentan el pool de Debian y las fuentes por DHCP, que no van autenticadas:
```
# Homelab, solo NTS: pool 2.debian.pool.ntp.org iburst
# Homelab, solo NTS: sourcedir /run/chrony-dhcp
```
`/etc/chrony/sources.d/homelab-nts.sources`, con cinco servidores de cuatro operadores:
```
server nts.netnod.se iburst nts
server ptbtime1.ptb.de iburst nts
server ptbtime3.ptb.de iburst nts
server ntppool1.time.nl iburst nts
server time.cloudflare.com iburst nts
```
```bash
sudo systemctl restart chrony
```
```bash
sudo chronyc -N authdata
```
**Resultado:** Todas con `NTS` en `Mode` y `8` en `Cook`.
```bash
chronyc -N sources
```
**Resultado:** Una marcada con `^*`; y en `chronyc tracking`, `Leap status : Normal`.

Restos del paquete retirado (opcional): `sudo dpkg --purge systemd-timesyncd`.

### 4.5 Ajustes del núcleo

`/etc/sysctl.d/90-homelab.conf`:
```
kernel.kptr_restrict = 2
dev.tty.ldisc_autoload = 0
fs.protected_fifos = 2
kernel.yama.ptrace_scope = 1
net.core.bpf_jit_harden = 2
kernel.kexec_load_disabled = 1
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
```
```bash
sudo sysctl -p /etc/sysctl.d/90-homelab.conf
```

| Ajuste | Para qué |
|---|---|
| `kptr_restrict = 2` | Oculta las direcciones del núcleo, también a root |
| `ldisc_autoload = 0` | Un usuario no puede cargar disciplinas de terminal, fuente de fallos históricos |
| `protected_fifos = 2` | Sin escribir en FIFOs ajenos en carpetas compartidas como `/tmp` |
| `ptrace_scope = 1` | Un proceso solo puede depurar a sus hijos: otro proceso del mismo usuario no puede leer su memoria |
| `bpf_jit_harden = 2` | Endurece el compilador JIT de BPF |
| `kexec_load_disabled = 1` | Sin cargar otro núcleo en caliente. **No se puede deshacer sin reiniciar** |
| `send_redirects` y `secure_redirects = 0` | Este equipo no es el router de la LAN |
| `log_martians = 1` | Registra paquetes con un origen imposible |
| `rp_filter = 2` | Filtro de origen flexible. El estricto (`1`) da problemas con Docker y con Tailscale |

**Sin tocar, a propósito:**
- `ip_forward`: lo necesita Docker.
- `sysrq`: el valor de Debian (`438`) permite reiniciar con el teclado si el escritorio se cuelga.
- `modules_disabled`: impediría cargar módulos hasta reiniciar, también los de USB o NVIDIA.
- Espacios de nombres de usuario: los usan Flatpak y los navegadores.
- `unprivileged_bpf_disabled` ya está en `2`.

### 4.6 Auditoría final

```bash
sudo lynis audit system --quiet
```
Resultado: **76** (antes 72), sin avisos, 41 sugerencias (antes 46). La única nueva es purgar `systemd-timesyncd`.

## 5. Verificación

- [x] ssh-audit sin avisos ni fallos; el PC entra.
- [x] `su` limitado: la línea de `pam_wheel` en `/etc/pam.d/su`, después de `pam_rootok`.
- [x] `pam_pwquality` en `common-password` con `minlen = 12`.
- [x] `needrestart` en modo aviso: tras instalar chrony listó "No services need to be restarted".
- [x] `modprobe -n -v dccp` → `install /bin/false`.
- [x] chrony: 5 fuentes NTS con 8 cookies, sincronizado.
- [x] Tras los ajustes del núcleo: SSH nuevo, Moonlight, Jellyfin y la salida del agente funcionan.
- [x] Lynis: de 72 a 76, sin avisos.
- [ ] Termius en el iPhone y el iPad.
- [ ] Tras un reinicio: los ajustes del núcleo y chrony siguen activos. En la sesión en persona.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| Termius corta la conexión tras el cambio de algoritmos | Solo usa la clave ECDSA del servidor | Añadir `ecdsa-sha2-nistp256` a `HostKeyAlgorithms`, o dejarlo pendiente como aquí |
| Un cliente avisa de que la clave del servidor cambió | El servidor ya no ofrece la clave que el cliente guardó | Comparar con `ssh-keygen -lf …ed25519_key.pub` y aceptar solo si coincide |
| `chronyc authdata` con `Cook` por debajo de 8 o `Atmp` mayor que 1 | Sin salida al puerto TCP 4460 o un servidor caído | Probar la salida y cambiar el servidor por otro de la lista de NTS |
| `su: Permission denied` | El usuario no está en el grupo `sudo` | Es lo esperado. Usa `sudo` |
| `sudo cat /carpeta/archivo*` no encuentra el archivo | El `*` lo expande tu usuario antes de `sudo` | `sudo sh -c 'cat /carpeta/archivo*'` |

## 7. Cómo revertirlo

| Punto | Revertir |
|---|---|
| Algoritmos SSH | `sudo rm /etc/ssh/sshd_config.d/11-homelab-algoritmos.conf` y `sudo systemctl reload ssh` |
| `su` | Restaurar `/etc/pam.d/su.bak-AAAAMMDD` |
| Contraseñas | `sudo apt purge libpam-pwquality` |
| needrestart | `sudo apt purge needrestart` |
| Protocolos | `sudo rm /etc/modprobe.d/homelab-protocolos.conf` |
| Hora | `sudo apt install systemd-timesyncd` (desinstala chrony) |
| Núcleo | `sudo rm /etc/sysctl.d/90-homelab.conf` y reiniciar |

## 8. Seguridad

**Mitiga:**
- Algoritmos débiles o sospechosos en SSH; el intercambio de claves ya es híbrido poscuántico.
- Una cuenta sin privilegios que adivine la contraseña de root no puede usar `su`.
- Contraseñas nuevas cortas.
- Servicios con bibliotecas vulnerables ya parcheadas en disco sin que nadie se entere.
- Fallos en protocolos de red que no se usan.
- Hora falsificada desde la red: caducidad de certificados y códigos TOTP.
- Exploits del núcleo que necesitan sus direcciones, `ptrace` entre procesos o `kexec`.

**Queda:**
- Termius sin SSH hasta revisarlo.
- Las sugerencias de Lynis descartadas: ver la tabla de la [guía de securización](../guia-securizacion.md). Las de correo, AIDE, rkhunter, auditd y registros, en la Fase 5.

## 9. Para la entrevista

**En 30 segundos:** "Pasé Lynis antes y después: de 72 a 76 sin avisos, aplicando solo lo que encaja con un portátil que hace de servidor. En SSH dejé solo algoritmos modernos, con intercambio de claves híbrido poscuántico, y lo probé con vuelta atrás automática. Limité `su`, puse longitud mínima a las contraseñas y `needrestart`. Cambié la hora a chrony con NTS, porque TOTP y TLS dependen del reloj. Y ajusté el núcleo uno a uno, dejando fuera lo que rompería Docker o el escritorio."

**Preguntas posibles:**
- *¿Por qué no subir más la nota de Lynis?* Muchas sugerencias no encajan: particiones separadas, contraseña de GRUB, `sysrq` o `rp_filter` estricto. Cada una está justificada.
- *¿Por qué NTS?* NTP sin autenticar se puede manipular desde la red. Un reloj atrasado puede aceptar certificados caducados y uno adelantado rompe los códigos TOTP.
- *¿Por qué `rp_filter = 2` y no `1`?* El estricto descarta tráfico asimétrico legítimo: Docker y, sobre todo, Tailscale lo desaconsejan.
- *¿Y el archivo `moduli`?* Con esta lista de `KexAlgorithms` no se ofrece Diffie-Hellman de campo finito, así que `moduli` no se usa.

## Fuentes

- [Lynis](https://cisofy.com/lynis/) y [ssh-audit](https://github.com/jtesta/ssh-audit).
- [chrony.conf](https://chrony-project.org/doc/4.6/chrony.conf.html) y [chronyc](https://chrony-project.org/doc/4.6/chronyc.html).
- [Lista de servidores NTS](https://github.com/jauderho/nts-servers) (revisada el 2026-11-15).
- `chrony.conf` de Debian ([salsa](https://salsa.debian.org/debian/chrony/-/raw/debian/latest/debian/chrony.conf)).
- Documentación del núcleo: [sysctl/kernel](https://docs.kernel.org/admin-guide/sysctl/kernel.html) y [ip-sysctl](https://docs.kernel.org/networking/ip-sysctl.html).
- [How To Secure A Linux Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server) (CC BY-SA 4.0), adaptada: ver [`../guia-securizacion.md`](../guia-securizacion.md).
