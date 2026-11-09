# Paso 2: SSH de emergencia con clave, TOTP y Fail2ban

> **Versiones usadas:** Debian 13.7, OpenSSH 1:10.0p1-7+deb13u4, libpam-google-authenticator 20191231-2.1, Fail2ban 1.1.0-8. Cliente: OpenSSH incluido en Windows.

## 1. Qué vamos a hacer y por qué

**En sencillo:** hasta ahora la única forma de manejar el servidor a distancia era el escritorio remoto (Moonlight), y si fallaba la gráfica, la red o el firewall, se perdía el control. Montamos una **puerta trasera de emergencia** muy bien cerrada: para abrirla hacen falta **una llave que solo está en el PC** y **un código que cambia cada 30 segundos en el móvil**. Y un vigilante (Fail2ban) que echa a quien intente forzarla.

**Técnicamente:**
- OpenSSH con autenticación por clave pública Ed25519 **y** TOTP (`AuthenticationMethods publickey,keyboard-interactive`).
- Solo los miembros de `sshusers`, nunca root, sin reenvíos.
- PAM configurado para pedir solo el TOTP, nunca la contraseña de Unix.
- Fail2ban vigilando el registro de `sshd`.
- Todo configurado **antes** de instalar el servidor, para que arranque ya protegido.

## 2. Conceptos previos

SSH, clave pública y privada, huella del servidor, TOTP, PAM y Fail2ban: ver [`../00-conceptos.md`](../00-conceptos.md#acceso-remoto).

## 3. Requisitos y estado inicial

- Servidor sin `openssh-server` instalado (`apt-cache policy openssh-server` → `Installed: (none)`).
- Hora sincronizada en el servidor (`timedatectl` → `System clock synchronized: yes`).
- PC con Windows 10 u 11 (trae el cliente `ssh`) y reserva DHCP hecha ([paso 1](01-reservas-dhcp.md)).
- Móvil con una app de códigos (Google Authenticator, Microsoft Authenticator, 2FAS, Aegis, Proton Authenticator…).
- Un gestor de contraseñas para la frase de paso y los códigos de emergencia.
- Un segundo dispositivo con app de SSH (aquí, Termius en el iPhone) para probar Fail2ban.
- Una sesión de escritorio remoto o acceso físico abierto durante todo el paso, como respaldo.

## 4. Pasos

Cada bloque indica **dónde** se ejecuta y **con qué usuario**. `<USUARIO_ADMIN>` es tu usuario administrador del servidor. Pega los comandos **de uno en uno** y sin las líneas de explicación.

### 4.1 Crear la clave en el PC (PC, Símbolo del sistema, tu usuario de Windows)

```
dir %USERPROFILE%\.ssh
```
Comprueba que **no existe** ya un `id_ed25519`. Si existe, no sigas: crear otra la sobrescribiría.

Crea antes, en tu gestor de contraseñas, una **frase de paso nueva** (5-6 palabras al azar o 20+ caracteres) y guárdala.

```
ssh-keygen -t ed25519 -C "pc-homelab"
```
- `-t ed25519`: tipo de clave moderno, corto y robusto. Es también el que Windows usa por defecto.
- `-C "pc-homelab"`: una etiqueta para reconocerla, sin datos personales.
- `Enter file…`: pulsa Enter (ruta por defecto, `C:\Users\<USUARIO_WINDOWS>\.ssh\id_ed25519`).
- `Enter passphrase`: pega la frase. No se ve nada al escribir.

Apunta la línea `The key fingerprint is: SHA256:…`.

```
type %USERPROFILE%\.ssh\id_ed25519.pub
```
Muestra la **parte pública**, una línea que empieza por `ssh-ed25519` y termina en `pc-homelab`. Esa sí se puede copiar. El archivo **sin** `.pub` es la llave privada: no sale nunca del PC.

### 4.2 Autorizar la clave (servidor, `<USUARIO_ADMIN>`, sin `sudo`)

```bash
install -d -m 700 ~/.ssh
```
Crea la carpeta `.ssh` si no existe y la deja solo para ti.

```bash
nano ~/.ssh/authorized_keys
```
Pega la línea pública **en una sola línea**, guarda (`Ctrl+O`, Enter) y sal (`Ctrl+X`). Si se te parte al pegar, se rompe: es más seguro copiarla al servidor como archivo y añadirla con `cat archivo.pub >> ~/.ssh/authorized_keys`.

```bash
chmod 600 ~/.ssh/authorized_keys
```
SSH rechaza el archivo si otros pueden escribir en él.

```bash
ssh-keygen -lf ~/.ssh/authorized_keys
```
Debe salir **una línea** con la misma huella `SHA256:…` que apuntaste en el PC, terminada en `pc-homelab (ED25519)`.

### 4.3 Grupo de acceso (servidor, `<USUARIO_ADMIN>`)

```bash
sudo groupadd sshusers
```
```bash
sudo usermod -aG sshusers <USUARIO_ADMIN>
```
`-aG` **añade** el grupo sin quitar los que ya tienes. Sin la `a`, te quitaría del resto, incluido `sudo`.

```bash
getent group sshusers
```
Salida esperada: `sshusers:x:<GID>:<USUARIO_ADMIN>`.

El usuario del agente de IA **no** entra en este grupo; si algún día necesita SSH (Fase 7), tendrá su propio bloque `Match`.

### 4.4 Configuración de `sshd` antes de instalarlo (servidor, `<USUARIO_ADMIN>`)

Debian incluye `/etc/ssh/sshd_config.d/*.conf` **al principio** de `sshd_config`, y `sshd` se queda con el **primer** valor que lee. Por eso un archivo propio en esa carpeta manda sobre el principal, y el principal se puede dejar como lo instala Debian.

```bash
sudo install -d -m 755 /etc/ssh/sshd_config.d
```
```bash
sudo nano /etc/ssh/sshd_config.d/10-homelab.conf
```

Contenido **de esta primera etapa**, solo con clave. El TOTP se activa en el apartado 4.9 cambiando dos líneas.

```
# Homelab, Fase 1: SSH de emergencia.
# Debian incluye esta carpeta al principio de sshd_config y sshd se queda con el
# PRIMER valor que lee: lo de aquí manda sobre el archivo principal.
# Comprobar siempre con: sudo sshd -T

# Quién puede entrar: solo el grupo sshusers y nunca root.
AllowGroups sshusers
PermitRootLogin no

# Cómo: solo clave pública. Sin contraseña ni métodos interactivos.
AuthenticationMethods publickey
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitEmptyPasswords no
HostbasedAuthentication no

# Sin reenvíos ni túneles: no se usan y amplían lo que puede hacer una clave robada.
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
AllowStreamLocalForwarding no
GatewayPorts no
PermitTunnel no
PermitUserEnvironment no

# Límites. La guía de referencia propone MaxAuthTries 2 y MaxSessions 2; se deja
# margen para un cliente que pruebe varias claves y para varias ventanas.
LoginGraceTime 30
MaxAuthTries 3
MaxSessions 4
MaxStartups 10:30:60

# Cierra conexiones muertas tras unos 10 minutos sin respuesta del cliente.
ClientAliveInterval 300
ClientAliveCountMax 2

# Registra la huella de la clave usada en cada acceso.
LogLevel VERBOSE
```

### 4.5 Instalar el servidor (servidor, `<USUARIO_ADMIN>`)

```bash
sudo apt install openssh-server
```
Instala tres paquetes (`openssh-server`, `openssh-sftp-server`, `runit-helper`) y arranca el servicio `ssh` ya con tu configuración.

```bash
systemctl is-active ssh
```
Salida esperada: `active`.

```bash
sudo sshd -t && echo config-ok
```
Valida la sintaxis de toda la configuración. Salida esperada: `config-ok`.

```bash
sudo sshd -T | grep -E '^(permitroot|passwordauth|allowgroups|authenticationmethods)'
```
`sshd -T` imprime la configuración **efectiva**, después de juntar todos los archivos. Salida esperada:
```
permitrootlogin no
passwordauthentication no
allowgroups sshusers
authenticationmethods publickey
```

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```
**Huella del servidor.** Apúntala: la necesitas en el paso siguiente para saber que te conectas al equipo correcto.

### 4.6 Primera entrada (PC, Símbolo del sistema)

```
ssh <USUARIO_ADMIN>@<IP_SERVIDOR>
```
1. Pregunta si te fías del servidor y muestra `ED25519 key fingerprint is SHA256:…`. **Compárala** con la del paso anterior. Solo si coincide, escribe `yes`.
2. Pide la frase de paso de la clave.
3. Entras: `<USUARIO_ADMIN>@homelab:~$`.

Comprueba que `sudo` funciona y sal:
```bash
sudo true && echo sudo-ok
```
```bash
exit
```

### 4.7 Lo indebido se rechaza (PC)

```
ssh -o PubkeyAuthentication=no <USUARIO_ADMIN>@<IP_SERVIDOR>
```
Simula a alguien sin la clave. Esperado: `Permission denied (publickey).` **sin pedir contraseña**.

```
ssh root@<IP_SERVIDOR>
```
Esperado: `Permission denied (publickey).` Ni siquiera pide la frase de paso: el servidor rechaza la clave para root antes de llegar ahí.

### 4.8 Preparar el TOTP (servidor, `<USUARIO_ADMIN>`)

```bash
timedatectl | grep synchronized
```
Debe decir `yes`. Los códigos dependen de la hora del servidor y del móvil.

```bash
sudo apt install libpam-google-authenticator
```
Instala el módulo de PAM y el programa que genera el secreto. Todavía no cambia cómo se entra.

```bash
google-authenticator -t -d -f -W -r 3 -R 30 -i homelab -l <USUARIO_ADMIN>
```
- `-t`: códigos por tiempo (TOTP), cada 30 segundos.
- `-d`: cada código sirve una sola vez.
- `-f`: escribe el archivo sin preguntar.
- `-W`: en modo TOTP deja la ventana **estándar de 3 códigos** (el anterior, el actual y el siguiente), que tolera unos 30 s de desfase de reloj. El manual dice "desactivar la ventana", pero el código fuente de esta versión escribe `WINDOW_SIZE 3` para TOTP. Lo que se evita es la ventana ampliada de 17 códigos, que el programa ofrece si no se le dice nada.
- `-r 3 -R 30`: como máximo 3 intentos cada 30 segundos.
- `-i homelab -l <USUARIO_ADMIN>`: el nombre que verás en la app.

Después:
1. Escanea el **código QR** con la app del móvil.
2. Escribe el código de 6 cifras que muestre la app, cuando lo pida: comprueba que funciona.
3. Guarda los **5 códigos de emergencia** en el gestor de contraseñas, fuera del servidor.

**No pegues esta salida en ningún sitio:** contiene el secreto y los códigos de emergencia.

```bash
stat -c '%a %n' ~/.google_authenticator
```
Salida esperada: `400 …/.google_authenticator` (solo tú puedes leerlo).

Opcional, para ver las opciones sin mostrar el secreto (solo imprime las líneas que empiezan por `"`):
```bash
grep '^"' ~/.google_authenticator
```
Debe incluir `" WINDOW_SIZE 3`, `" RATE_LIMIT 3 30`, `" DISALLOW_REUSE` y `" TOTP_AUTH`.

### 4.9 Activar el TOTP (servidor, `<USUARIO_ADMIN>`, con una sesión de respaldo abierta)

**Antes:** abre en el PC una sesión SSH y **no la cierres** hasta terminar. Si algo falla, desde ahí se deshace.

Copias de seguridad:
```bash
sudo cp -a /etc/pam.d/sshd /etc/pam.d/sshd.bak-AAAAMMDD
```
```bash
cd /etc/ssh
```
```bash
sudo cp -a sshd_config.d/10-homelab.conf 10-homelab.conf.bak-AAAAMMDD
```
La copia del `.conf` se guarda **fuera** de `sshd_config.d/`. Dentro, cualquier archivo `.conf` se cargaría.

**PAM.** En `/etc/pam.d/sshd`, sustituye la línea `@include common-auth` por:
```
# Homelab: sin contraseña de Unix en SSH; el segundo factor es el TOTP.
# @include common-auth
auth       required     pam_google_authenticator.so
```
```bash
sudo nano /etc/pam.d/sshd
```
- Sin `common-auth`, PAM ya no pide la contraseña de Unix: tras la clave, solo el código.
- **Sin `nullok`:** con `nullok`, un usuario sin TOTP configurado entraría sin código. Aquí el único usuario de `sshusers` ya está inscrito.
- Las secciones `account` y `session` no se tocan.

**sshd.** En `10-homelab.conf`, cambia el bloque "Cómo" por:
```
# Cómo: primero la clave y después el código. Sin la directiva
# AuthenticationMethods, entrar con clave no pediría el código.
# keyboard-interactive pasa por PAM, que solo pide el TOTP (sin contraseña).
AuthenticationMethods publickey,keyboard-interactive
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication yes
PermitEmptyPasswords no
HostbasedAuthentication no
```
```bash
sudo nano sshd_config.d/10-homelab.conf
```

Valida y aplica **sin cortar** las sesiones abiertas:
```bash
sudo sshd -t && echo config-ok
```
```bash
sudo systemctl reload ssh
```
`reload` relee la configuración para las conexiones nuevas; la sesión de respaldo sigue viva.

**Prueba en otra ventana del PC** (sin cerrar la de respaldo):
```
ssh <USUARIO_ADMIN>@<IP_SERVIDOR>
```
Frase de paso → `Verification code:` → código de la app → dentro.

### 4.10 Cada factor por separado no basta (PC)

| Prueba | Cómo | Esperado |
|---|---|---|
| Clave con código incorrecto | `ssh <USUARIO_ADMIN>@<IP_SERVIDOR>` y `000000` como código | Vuelve a pedirlo o `Permission denied`; no entras |
| Sin clave | `ssh -o PubkeyAuthentication=no <USUARIO_ADMIN>@<IP_SERVIDOR>` | `Permission denied (publickey).` sin pedir código |
| Código de emergencia | `ssh <USUARIO_ADMIN>@<IP_SERVIDOR>` y uno de los 5 códigos | Entras. Ese código queda gastado: márcalo en el gestor |

Espera 30 segundos entre la primera prueba y la última: el límite es de 3 intentos cada 30 segundos.

### 4.11 Fail2ban (servidor, `<USUARIO_ADMIN>`)

En Debian 13 no existe `/var/log/auth.log`: los registros van al *journal* de systemd. El paquete ya trae `/etc/fail2ban/jail.d/defaults-debian.conf` con la jaula `sshd` activada, leyendo el journal (`backend = systemd`) y bloqueando con nftables. Solo se añaden los valores propios.

```bash
sudo apt install fail2ban
```

```bash
sudo nano /etc/fail2ban/jail.d/homelab.local
```
```
# Homelab, Fase 1: Fail2ban solo para SSH.
# Debian ya activa la jaula sshd con el journal y bloquea con nftables,
# solo en el puerto de SSH: el escritorio remoto no se ve afectado.

[DEFAULT]
# Nunca se bloquea el PC de rescate (reserva DHCP) ni el propio servidor.
ignoreip = 127.0.0.1/8 ::1 <IP_PC>

[sshd]
enabled  = true
# aggressive también cuenta las conexiones cerradas antes de autenticarse,
# que es lo que deja un intento sin clave cuando solo se admite clave.
mode     = aggressive
maxretry = 5
findtime = 10m
bantime  = 1h
```

```bash
sudo systemctl restart fail2ban
```
```bash
sudo fail2ban-client status sshd
```
Esperado: `Status for the jail: sshd`, `Currently banned: 0`.

```bash
sudo fail2ban-client get sshd ignoreip
```
Debe aparecer `<IP_PC>`.

**Prueba de bloqueo** desde otro dispositivo de casa (aquí, el iPhone con Termius y el Wi-Fi encendido): conexión a `<IP_SERVIDOR>`, puerto 22, usuario `prueba`, varias veces. Después, en el servidor:

```bash
sudo fail2ban-client status sshd
```
Esperado: `Currently banned: 1` y la IP del móvil en `Banned IP list`.

```bash
sudo nft list ruleset | grep -A4 f2b
```
Muestra la regla: rechaza el TCP 22 **solo** desde las IP bloqueadas. El resto de puertos de ese equipo siguen abiertos.

**Desbloqueo:**
```bash
sudo fail2ban-client set sshd unbanip <IP_BLOQUEADA>
```
Esperado: `1`. Compruébalo **en el servidor**, no reintentando desde el móvil (cada intento fallido vuelve a contar):
```bash
sudo nft list set inet f2b-table addr-set-sshd
```
Esperado: el conjunto sin línea `elements`.

**El PC nunca se bloquea:** desde el PC, repite 6 veces `ssh -o PubkeyAuthentication=no <USUARIO_ADMIN>@<IP_SERVIDOR>` (todas deben dar `Permission denied`), y en el servidor:
```bash
sudo fail2ban-client status sshd
```
```bash
sudo grep -c 'Ignore <IP_PC>' /var/log/fail2ban.log
```
Esperado: `Currently banned: 0` y un número mayor que 0 (aquí, 8): Fail2ban vio los fallos y los ignoró a propósito. El contador `Total failed` no sube, porque las IP ignoradas no cuentan.

### 4.12 Volver a entrar y recuperar el acceso

Para retomar el trabajo desde el PC:
```
ssh <USUARIO_ADMIN>@<IP_SERVIDOR>
```
```bash
tmux attach -t homelab
```
(o `tmux new -s homelab` si no existe).

| Vía de acceso | Cuándo sirve |
|---|---|
| SSH desde el PC | Lo normal. El PC nunca lo bloquea Fail2ban |
| Escritorio remoto (Moonlight) | Si falla el SSH. Necesita la sesión gráfica iniciada |
| Teclado y pantalla del portátil | Si fallan los dos. En la pantalla de inicio entra con tu usuario, o `Ctrl+Alt+F3` para una consola de texto |

| Qué ha pasado | Qué hacer |
|---|---|
| Perdí el móvil y tengo códigos de emergencia | Entra con uno y ejecuta otra vez `google-authenticator` (4.8): invalida el secreto anterior y los códigos viejos |
| Perdí el móvil y no me quedan códigos | Entra por escritorio remoto o físicamente y ejecuta `google-authenticator` |
| Perdí el PC o me robaron la clave | Por escritorio remoto o físicamente: borra su línea de `~/.ssh/authorized_keys`, crea una clave nueva en otro equipo y cambia `<IP_PC>` en `homelab.local` y en la reserva DHCP |
| Fail2ban bloqueó un equipo mío | `sudo fail2ban-client set sshd unbanip <IP>` desde el PC o el escritorio remoto |
| Un cambio rompió el SSH | Escritorio remoto o físico: restaura las copias del apartado 4.9 (sección 7) |

## 5. Verificación

- [x] `sshd -T` muestra `permitrootlogin no`, `passwordauthentication no`, `allowgroups sshusers` y `authenticationmethods publickey,keyboard-interactive`.
- [x] Entra con clave + código, y `sudo` funciona.
- [x] Se rechazan: sin clave, código incorrecto, root y contraseña.
- [x] Un código de emergencia entra.
- [x] Fail2ban bloquea otro equipo solo en el puerto 22, lo desbloquea y nunca bloquea el PC.
- [x] Desde internet no llega: el router no reenvía el 22 al servidor ([auditoría, 4.9](../00-auditoria.md#49-hallazgo-el-router-tenía-su-propio-ssh-abierto-a-internet)).
- [ ] Limitar el 22 a la LAN con el firewall: paso 7.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `bash: -aG: command not found` o similar | Se pegó también una línea de la explicación | No pasa nada: ese texto no es un comando. Pega solo los bloques de código |
| `grep: … Usage` y `command not found` con un trozo del patrón | La línea larga se partió al pegarla | Pégala de nuevo entera o divídela en comandos cortos |
| Pide contraseña de Unix después de la clave | Falta comentar `@include common-auth` en `/etc/pam.d/sshd` | Revisa 4.9 |
| Entra solo con la clave, sin pedir código | Falta `AuthenticationMethods publickey,keyboard-interactive`, o `KbdInteractiveAuthentication` sigue en `no` | Comprueba con `sudo sshd -T` |
| `Verification code` siempre incorrecto | Hora desincronizada en el servidor o en el móvil | `timedatectl` en el servidor y hora automática en el móvil |
| Tras desbloquear un equipo, vuelve a estar bloqueado | Con `mode = aggressive` y un usuario inexistente, **un solo** intento genera varios fallos y llega a 5 | Comprueba el desbloqueo en el servidor con `nft list set`, no reintentando |
| La app del móvil dice "connection could not be established" | Puede ser un bloqueo de Fail2ban (rechazo con `port-unreachable`) | Mira `fail2ban-client status sshd` antes de suponer otra causa |
| `sudo sshd -t` da un error | Directiva mal escrita | **No** recargues: corrige y valida otra vez. Mientras no recargues, sigue la configuración anterior |

## 7. Cómo revertirlo

Desde el escritorio remoto o físicamente, con tu usuario.

**Quitar el TOTP** (vuelve a "solo clave"):
```bash
cd /etc/ssh
```
```bash
sudo cp -a /etc/pam.d/sshd.bak-AAAAMMDD /etc/pam.d/sshd
```
```bash
sudo cp -a 10-homelab.conf.bak-AAAAMMDD sshd_config.d/10-homelab.conf
```
```bash
sudo sshd -t && sudo systemctl reload ssh
```

**Quitar Fail2ban:**
```bash
sudo apt purge fail2ban
```

**Quitar el SSH del todo:**
```bash
sudo apt purge openssh-server
```
```bash
sudo rm /etc/ssh/sshd_config.d/10-homelab.conf
```
```bash
sudo groupdel sshusers
```

## 8. Seguridad

**Mitiga:**
- Contraseñas adivinadas: no se aceptan contraseñas.
- Clave privada robada: sin el móvil no basta, y la clave tiene frase de paso.
- Móvil robado: sin la clave del PC no basta.
- Intentos repetidos desde la LAN: Fail2ban.
- Root remoto: prohibido.
- Abuso de una sesión para hacer túneles: reenvíos desactivados.

**Queda:**
- Mientras no haya firewall (paso 7), el 22 es alcanzable desde **cualquier equipo de casa**, aunque exija clave y código.
- El secreto TOTP está en `~/.google_authenticator` (permisos 400). Root o quien tenga el disco lo puede leer: riesgo físico aceptado ([ADR 0001](../decisiones/0001-sin-cifrado-de-disco.md)).
- La exclusión de Fail2ban depende de la IP del PC, que se puede suplantar en la LAN. Solo evita bloqueos: no da acceso.
- Algoritmos de cifrado y `/etc/ssh/moduli` sin restringir todavía: se harán en el paso 9, tras comprobar que el PC, el iPad y el iPhone siguen pudiendo conectar. OpenSSH 10 ya prefiere por defecto intercambios de claves modernos.
- El TOTP no protege si el PC está comprometido **mientras** hay una sesión abierta.

## 9. Para la entrevista

**En 30 segundos:** "Monté SSH como acceso de emergencia con clave Ed25519 con frase de paso y TOTP, usando `AuthenticationMethods publickey,keyboard-interactive` y quitando `common-auth` de PAM para que nunca pida la contraseña de Unix. Solo entra un grupo, root no, sin reenvíos, y Fail2ban lee el journal porque Debian 13 ya no tiene `auth.log`. Configuré todo antes de instalar el paquete, para que arrancase ya protegido, y probé que cada factor por separado se rechaza."

**Preguntas posibles:**
- *¿Por qué no cambiaste el puerto 22?* Porque no está expuesto a internet, así que no hay ruido que reducir. Cambiarlo solo complica el rescate. La seguridad la dan la clave, el TOTP y el firewall.
- *¿Para qué sirve `AuthenticationMethods`?* Sin ella, OpenSSH acepta **cualquiera** de los métodos activos: entrar con clave no pediría el código. Con `publickey,keyboard-interactive` exige los dos, en ese orden.
- *¿Por qué quitar `common-auth`?* Porque `keyboard-interactive` pasa por PAM y PAM pediría también la contraseña de Unix. Quería clave + código, no tres factores, y que la contraseña de `sudo` no viaje por SSH.
- *¿Y `nullok`?* Deja entrar sin código a quien no tenga TOTP configurado. Sirve para una transición con varios usuarios; aquí solo hay uno y ya está inscrito.
- *¿Qué hace `sshd -T` y por qué usarlo?* Imprime la configuración efectiva tras juntar todos los archivos. En Debian manda el primer valor leído, así que un archivo mal ordenado puede anular otro sin dar error.
- *¿Fail2ban sustituye al firewall?* No. Solo reacciona a fallos en los registros. El firewall decide quién puede intentarlo.
- *¿Qué pasa si pierdes el móvil?* Códigos de emergencia o acceso físico, y regenerar el secreto. Está escrito y probado.

## Fuentes

- [How To Secure A Linux Server: The SSH Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server#the-ssh-server) (CC BY-SA 4.0), adaptada: ver [`../guia-securizacion.md`](../guia-securizacion.md).
- `man sshd_config` de OpenSSH 10.0 y la configuración por defecto de Debian (`/usr/share/openssh/sshd_config`).
- `man google-authenticator`, el archivo `FILEFORMAT` y el código fuente de libpam-google-authenticator 1.08.
- [Microsoft: autenticación por clave en OpenSSH para Windows](https://learn.microsoft.com/windows-server/administration/openssh/openssh_keymanagement).
- Configuración de Fail2ban incluida en el paquete de Debian (`jail.d/defaults-debian.conf`, `filter.d/sshd.conf`).
