# Fase 5, paso 3: nube con Syncthing y Samba

> **Versiones usadas:** Debian 13.7, Docker Engine 29.9.3, Compose 5.7.1, imagen `syncthing/syncthing:2.1.7`, Syncthing 2.1.7 para Windows (zip oficial), Samba 4.22.11 de Debian.

## 1. Qué vamos a hacer y por qué

**En sencillo:** Documentos, Imágenes y Vídeos del PC se copian solos al servidor y se mantienen al día. Si algo se borra o se estropea en el PC, el servidor guarda versiones antiguas durante un año. Desde el iPhone, el iPad o Windows se ven esos archivos como una carpeta de red, y hay otra carpeta para subir cosas a mano.

**Técnicamente:**
- **Syncthing** en Docker en el servidor y el zip oficial en Windows. El PC **envía solamente** y el servidor **recibe solamente**, con versiones escalonadas.
- Sin relays, sin descubrimiento y sin NAT traversal: el PC llama directamente a `<IP_SERVIDOR>:22000`.
- **Samba** en el host, con dos carpetas compartidas: `pc` (el espejo, de solo lectura) y `subidas` (lectura y escritura).

Decisión y alternativas: [ADR 0011](../decisiones/0011-nube-con-syncthing-y-samba.md).

## 2. Conceptos previos

- **Sincronizar no es hacer copia de seguridad.** El servidor es un espejo: lo que se borra en el PC se borra en el servidor. Las versiones salvan los despistes; las copias de verdad son la Fase 6.
- **Tipos de carpeta en Syncthing:** *Enviar solamente* manda los cambios y no acepta los de fuera. *Recibir solamente* acepta los cambios y no manda nada; si algo cambia en el servidor, lo marca como "cambio local".
- **Versiones escalonadas:** al borrar o sustituir un archivo, el servidor guarda la versión anterior en `.stversions`. Una cada 30 s durante el primer día, una por día durante un mes y una por semana hasta el año.
- **ID de dispositivo:** la huella de la clave de cada equipo. Solo se conectan los equipos que se han aceptado el uno al otro.
- **SMB y Samba:** SMB es el protocolo de carpetas de red de Windows; Samba es el servidor SMB de Linux. La app Archivos del iPhone y el Explorador de Windows lo usan.
- **`vfs_fruit`:** módulo de Samba que añade las extensiones que esperan los dispositivos de Apple.

## 3. Requisitos y estado inicial

- Docker del [paso 8 de la Fase 1](../01-base-y-endurecimiento/08-docker.md) y el disco de datos montado en `/srv/datos` ([paso 5](../01-base-y-endurecimiento/05-disco-de-datos.md)).
- Tailscale de la [Fase 3](../03-acceso-remoto.md).
- Espacio: unos 77 GB entre las tres carpetas, con 1,8 TB libres.
- Nada escuchando en los puertos que se usan:
  ```bash
  sudo ss -lntup | grep -E ':(22000|8384|21027|139|445) '
  ```
  **Resultado:** Vacío.

## 4. Pasos

En el servidor, con `<USUARIO_ADMIN>`. Se hizo con scripts revisados antes de ejecutarlos; estos son sus pasos.

### 4.1 Usuario, carpetas y Compose

```bash
sudo adduser --system --group --no-create-home --home /nonexistent syncthing
```
```bash
sudo install -d -m 755 /srv/syncthing
```
```bash
sudo install -d -o syncthing -g syncthing -m 700 /srv/syncthing/var
```
```bash
sudo install -d -o syncthing -g syncthing -m 750 /srv/datos/archivos/pc
```
Copia [`compose.yaml`](../../stacks/syncthing/compose.yaml) en `/srv/syncthing/` y crea `.env` a partir de [`.env.example`](../../stacks/syncthing/.env.example).

Con `user:`, la imagen no pasa por su `su-exec` ni por `setcap`: ejecuta Syncthing directamente. Por eso funciona con `cap_drop: ALL` sin añadir nada.

### 4.2 Reglas en `DOCKER-USER`

`/etc/systemd/system/homelab-syncthing-lan.service`, con el mismo esquema que el de AdGuard ([Fase 4, 4.2](../04-dns.md#42-reglas-en-docker-user)):
```ini
[Unit]
Description=Homelab: Syncthing (sync LAN y tailnet; panel PC y tailnet), en DOCKER-USER
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctstate DNAT --ctorigdstport 8384 --ctdir ORIGINAL ! -s <IP_PC> -j DROP
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p udp -m conntrack --ctstate DNAT --ctorigdstport 22000 --ctdir ORIGINAL ! -s <RED_LAN> -j DROP
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctstate DNAT --ctorigdstport 22000 --ctdir ORIGINAL ! -s <RED_LAN> -j DROP
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctstate DNAT --ctorigdstport 8384 --ctdir ORIGINAL -i tailscale0 -s 100.64.0.0/10 -j RETURN
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p udp -m conntrack --ctstate DNAT --ctorigdstport 22000 --ctdir ORIGINAL -i tailscale0 -s 100.64.0.0/10 -j RETURN
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctstate DNAT --ctorigdstport 22000 --ctdir ORIGINAL -i tailscale0 -s 100.64.0.0/10 -j RETURN
```
Lleva también un `ExecStop=-… -D DOCKER-USER …` por cada regla y `WantedBy=docker.service` en `[Install]`.

```bash
sudo systemctl daemon-reload
```
```bash
sudo systemctl enable --now homelab-syncthing-lan.service
```
**Las reglas antes que el contenedor:** así no hay un momento con los puertos abiertos.
```bash
sudo docker compose -f /srv/syncthing/compose.yaml up -d
```
```bash
sudo docker top syncthing -o user,pid,args
```
**Resultado:** El proceso con el UID de `syncthing`.

El navegador del **propio servidor** también entra en el panel: lo que sale del mismo equipo no pasa por `FORWARD`.

### 4.3 Syncthing: servidor y PC

**Panel del servidor**, desde el PC en `http://<IP_SERVIDOR>:8384`:
1. Rechaza el informe de uso anónimo.
2. **Acciones → Ajustes → GUI:** usuario y contraseña (en el gestor de contraseñas) y **Usar HTTPS para la GUI**. El aviso naranja ("la dirección la fija una opción de arranque") es normal: la imagen fija `0.0.0.0:8384`.
3. **Conexiones:** desmarca **NAT traversal**, **Descubrimiento local**, **Descubrimiento global** y **Relays**. Guarda.

**Resultado:** Sin aviso rojo, **Oyentes 2/2** (TCP y QUIC; con relay serían 3/3) y, desde el servidor:
```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://127.0.0.1:8384/rest/system/status
```
**Resultado:** `403`: responde por HTTPS y sin credenciales no deja pasar.

**Syncthing en Windows** (no hay instalador oficial). En PowerShell, sin administrador:
1. Descarga de la [última versión](https://github.com/syncthing/syncthing/releases/latest) `syncthing-windows-amd64-v2.X.Y.zip` y comprueba su hash contra `sha256sum.txt.asc`, de la misma versión:
   ```
   Get-FileHash .\syncthing-windows-amd64-*.zip
   ```
2. Descomprímelo y pon la carpeta en `$env:LOCALAPPDATA\Programs\Syncthing`, fuera de Descargas. La configuración va aparte, en `%LOCALAPPDATA%\Syncthing`.
3. Acceso directo en la carpeta **Inicio** del usuario, sin ventana ni navegador:
   ```
   $ini = [Environment]::GetFolderPath('Startup')
   $ws = New-Object -ComObject WScript.Shell
   $s = $ws.CreateShortcut("$ini\Syncthing.lnk")
   $s.TargetPath = "$env:LOCALAPPDATA\Programs\Syncthing\syncthing.exe"
   $s.Arguments = "--no-console --no-browser"
   $s.WorkingDirectory = "$env:LOCALAPPDATA\Programs\Syncthing"
   $s.Save()
   Start-Process "$ini\Syncthing.lnk"
   ```
   **Resultado:** `Get-Process syncthing` muestra **dos** procesos: el vigilante y el programa.
4. En `127.0.0.1:8384`, lo mismo que en el servidor: usuario y contraseña **distintos**, HTTPS, sin **Iniciar navegador** y las cuatro opciones de **Conexiones** desmarcadas.
5. **Sin reglas en el firewall de Windows:** el PC solo llama hacia fuera.
   ```
   Get-NetFirewallRule -DisplayName *syncthing*
   ```
   **Resultado:** Nada.

**Emparejar:**
- En el PC, **Añadir dispositivo remoto** con el ID del servidor (**Acciones → Mostrar ID**) y, en **Avanzado → Direcciones**, `tcp://<IP_SERVIDOR>:22000`.
- En el servidor, acepta el aviso del PC comprobando el principio de su ID. Deja **Direcciones** en `dynamic`: con el descubrimiento apagado, el servidor nunca llama al PC.
- **Resultado:** Al cabo de un minuto, más o menos: **Conectado (Sin uso)** en los dos lados.

**Carpetas**, una a una ([tabla del stack](../../stacks/syncthing/README.md#carpetas)):
- **En el PC:** **Añadir carpeta**, con su ID, su ruta, compartida con el servidor y **Enviar solamente**. En **Patrones a ignorar**:
  - Documentos: [`documentos.stignore`](../../stacks/syncthing/documentos.stignore);
  - Imágenes y Vídeos: solo `(?d)desktop.ini` y `(?d)Thumbs.db`.

  Los patrones van **antes** de que el servidor acepte la carpeta: hasta entonces no sale nada.
- **En el servidor**, al aceptar:
  - **Folder Path**: `/datos/<nombre>`. **Nunca vacío ni con `~`**: iría a `/var/syncthing`, en el disco del sistema.
  - **Staggered**, con 365 días.
  - **Receive Only**.

### 4.4 Samba

**Confirmación expresa del dueño antes de abrir el 445 en el firewall.**

1. Instalar **sin los paquetes recomendados**:
   ```bash
   sudo apt-get install --no-install-recommends samba
   ```
   Con los recomendados entran `samba-ad-dc` y `winbind`, que añaden `pam_winbind` a `/etc/pam.d/common-*` y `winbind` a `/etc/nsswitch.conf` (ver [lecciones](../lecciones-aprendidas.md#samba-trajo-un-controlador-de-dominio-y-winbind-al-login)).

   **Lo que se hizo de verdad:** se instaló con los recomendados y después se quitaron, con una copia de `/etc/pam.d` y una sesión de root abierta:
   ```bash
   sudo apt-get purge samba-ad-dc samba-ad-provision winbind libnss-winbind libpam-winbind
   ```
   ```bash
   grep -n winbind /etc/pam.d/common-* /etc/nsswitch.conf
   ```
   **Resultado:** Nada, y `smbd` sigue `active`. Después, `sudo` y SSH probados en ventanas nuevas.
2. Parar y apagar NetBIOS:
   ```bash
   sudo systemctl stop smbd
   ```
   ```bash
   sudo systemctl mask --now nmbd
   ```
3. Guarda el original y pon [`smb.conf`](../../stacks/samba/smb.conf) con los marcadores sustituidos:
   ```bash
   sudo cp -a /etc/samba/smb.conf /etc/samba/smb.conf.orig-debian
   ```
   ```bash
   sudo testparm -s
   ```
   **Resultado:** `Loaded services file OK`, `ROLE_STANDALONE`, y `[pc]` y `[subidas]`.
4. Usuario de Samba, del sistema, sin shell, en el grupo `syncthing`, y la carpeta `subidas`:
   ```bash
   sudo adduser --system --group --no-create-home --home /nonexistent --shell /usr/sbin/nologin <USUARIO_WINDOWS>
   ```
   ```bash
   sudo usermod -aG syncthing <USUARIO_WINDOWS>
   ```
   ```bash
   sudo install -d -o <USUARIO_WINDOWS> -g <USUARIO_WINDOWS> -m 750 /srv/datos/archivos/subidas
   ```
   Prueba de verdad, antes de abrir nada:
   ```bash
   sudo -u <USUARIO_WINDOWS> touch /srv/datos/archivos/pc/.prueba
   ```
   **Resultado:** `Permission denied`. Y en `subidas`, el mismo `touch` funciona.
5. Contraseña de Samba, propia (no la de Windows), en el gestor de contraseñas:
   ```bash
   sudo smbpasswd -a <USUARIO_WINDOWS>
   ```
6. Firewall:
   ```bash
   sudo ufw allow from <RED_LAN> to any port 445 proto tcp comment 'Samba desde casa'
   ```
   ```bash
   sudo ufw allow in on tailscale0 to any port 445 proto tcp comment 'Samba tailnet'
   ```
   Y en la política de Tailscale, `"tcp:445"` en la lista `ip` del grant ([Fase 3, 4.1](../03-acceso-remoto.md#41-política-de-acceso)).
7. Arrancar:
   ```bash
   sudo systemctl enable --now smbd
   ```
   ```bash
   sudo ss -ltnp | grep -E ':(139|445) '
   ```
   **Resultado:** Solo el 445, de `smbd`.

**Conectarse:**
- **Windows:** con el usuario de Samba, no con el de Windows.
  ```
  net use \\<IP_SERVIDOR>\pc /user:homelab\<USUARIO_WINDOWS> *
  ```
  El `*` pide la contraseña sin mostrarla.
- **iPhone y iPad:** Archivos → **Explorar** → los tres puntos → **Conectarse al servidor** → `smb://<IP_TAILSCALE_SERVIDOR>` → **Usuario registrado**. Con la IP de la tailnet funciona igual en casa y fuera, con Tailscale activo. `<IP_SERVIDOR>` solo existe dentro de casa.

## 5. Verificación

- [x] El contenedor corre con el UID de `syncthing` y las reglas están encima de las de AdGuard y Jellyfin.
- [x] Panel del servidor: `403` sin credenciales por HTTPS. Sin relays ni descubrimiento (**Oyentes 2/2**).
- [x] El hash del zip de Windows coincide con la lista oficial, comprobada también desde el servidor.
- [x] Las tres carpetas, **Actualizado / Up to Date** en los dos lados: 13.756 archivos, 828 carpetas y 76,8 GiB.
- [x] Todo en el disco de datos (de 19 G a 89 G usados); el del sistema, igual. `/srv/syncthing/var`: 135 M.
- [x] En el servidor no hay `.env`, claves, `node_modules`, `My Games` ni `Restore Firefox`.
- [x] Windows: `smbstatus -b` muestra `SMB3_11` con `AES-128-GCM`. En `pc` se abren archivos y crear una carpeta da error.
- [x] iPhone en casa: descarga de `pc` y sube a `subidas`.
- [x] iPhone con datos y Tailscale: entra por `smb://<IP_TAILSCALE_SERVIDOR>`.
- [x] Tras quitar `winbind`: `sudo` y SSH (clave + TOTP) entran en ventanas nuevas.
- [ ] Tras un reinicio: las reglas vuelven, Syncthing y Samba arrancan, y el PC se reconecta solo. En la sesión en persona.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| Los dos equipos, "Desconectado" justo después de emparejar | Syncthing reintenta cada minuto, más o menos | Esperar |
| El panel del servidor avisa en rojo ("Danger!") | La GUI no tiene usuario ni contraseña | Ponerlos en **Ajustes → GUI** |
| `Move-Item`: "está en uso" al mover la carpeta de Syncthing | PowerShell está dentro de la carpeta, o Syncthing sigue abierto (el vigilante reabre el programa) | `cd $HOME`; `Stop-Process -Name syncthing -Force`; copiar con `Copy-Item -Recurse` |
| Windows pide credenciales con su propio usuario y falla | Prueba primero el usuario de Windows | **Más opciones** o `net use … /user:homelab\<USUARIO_WINDOWS> *` |
| Windows: error 1219 | Ya hay una conexión con otras credenciales | `net use * /delete /y` y repetir |
| iPhone: ve las carpetas pero no sube ni descarga | Faltan las extensiones de Apple | `vfs objects = catia fruit streams_xattr` (ya en `smb.conf`); desconectar y cerrar Archivos |
| No deja subir a `pc` ni a sus carpetas | Es el espejo, de solo lectura | Subir a `subidas` |
| Fuera de casa no entra con `<IP_SERVIDOR>` | Esa IP solo existe en la red de casa | `<IP_TAILSCALE_SERVIDOR>`, con Tailscale activo |
| El log de Samba se llena de `sys_get_quota … failed` | Samba consulta cuotas que btrfs no tiene | Ruido, sin efecto |
| `du /srv/datos/archivos/pc/*` no encuentra nada | El `*` lo expande tu usuario, que no puede leer la carpeta | `sudo sh -c 'du -sh /srv/datos/archivos/pc/*'` |

## 7. Cómo revertirlo

1. **Samba:**
   ```bash
   sudo apt purge samba
   ```
   Borra las dos reglas de UFW con comentario "Samba" (`sudo ufw status numbered` y `sudo ufw delete <número>`) y quita `"tcp:445"` de la política de Tailscale. Los usuarios de Samba se borran con `smbpasswd -x` antes de desinstalar.
2. **PC:** **Acciones → Apagar**, borra el acceso directo de Inicio y la carpeta de `Programs`. La configuración está en `%LOCALAPPDATA%\Syncthing`.
3. **Servidor:**
   ```bash
   sudo docker compose -f /srv/syncthing/compose.yaml down
   ```
   ```bash
   sudo systemctl disable --now homelab-syncthing-lan.service
   ```
   ```bash
   sudo rm /etc/systemd/system/homelab-syncthing-lan.service
   ```
   `/srv/syncthing` y `/srv/datos/archivos` no se borran solos.

## 8. Seguridad

**Mitiga:**
- Pérdida de archivos del PC por un despiste: espejo en otro disco, con versiones durante un año.
- Exposición: nada sale a internet. Sin relays ni descubrimiento, Syncthing no habla con servidores públicos. El panel solo llega desde el PC y la tailnet; la sincronización, desde la LAN y la tailnet. El PC no abre ningún puerto.
- Contenedor comprometido: usuario sin privilegios, sin capacidades y `no-new-privileges`.
- Fugas desde el PC: los patrones de Documentos dejan fuera claves, `.env`, perfiles del navegador y dependencias.
- Samba: solo SMB3 y cifrado, sin invitados ni NetBIOS, `hosts allow` además de UFW. El usuario de Samba no tiene shell y **no puede escribir en el espejo**; `fruit:nfs_aces = no` impide que un cliente de Apple cambie permisos.

**Queda:**
- **No es una copia de seguridad.** Un ransomware que cifre el PC se sincroniza; las versiones guardan lo anterior, pero en el mismo disco. Las copias fuera de este disco son la Fase 6.
- El disco de datos **no está cifrado** ([ADR 0001](../decisiones/0001-sin-cifrado-de-disco.md)) y Documentos puede tener papeles sensibles.
- Los paneles de Syncthing usan un certificado propio: el navegador avisa y hay que aceptarlo.
- `UMASK=027` no se aplica: los archivos llegan con `644` y las carpetas con `755`. No se nota porque `/srv/datos/archivos/pc` es `750`. Causa probable: la imagen solo aplica `UMASK` cuando arranca como root.
- En `subidas`, una cuenta de Samba robada puede borrar o cifrar lo subido. Entra en las copias de la Fase 6.
- El PC se actualiza solo; el servidor, a mano.

## 9. Para la entrevista

**En 30 segundos:** "Monté una nube propia sin exponer nada: Syncthing envía tres carpetas del PC al servidor, que solo recibe y guarda versiones durante un año. Quité relays y descubrimiento, así que el PC llama directamente al servidor y no abre ningún puerto. Para verlo desde el iPhone puse Samba con dos carpetas: el espejo, de solo lectura, y otra para subir. Me encontré dos sorpresas: los paquetes recomendados de Samba metieron `winbind` en la autenticación del sistema, y lo quité probando antes sudo y SSH; y el iPhone no copiaba nada hasta activar `vfs_fruit`."

**Preguntas posibles:**
- *¿Por qué no Nextcloud?* Para copiar carpetas del PC basta Syncthing, que es más pequeño y no necesita base de datos ni web. Ver [ADR 0011](../decisiones/0011-nube-con-syncthing-y-samba.md).
- *¿Por qué solo lectura en el espejo?* En una carpeta *Recibir solamente*, lo que se escribe en el servidor no llega al PC y se pierde al revertir. Y un error desde el iPhone no puede tocar los archivos del PC.
- *¿Por qué el servidor no llama al PC?* Con el PC llamando siempre, el PC no necesita reglas de entrada en su firewall.
- *¿Qué pasa si borras algo en el PC?* Se borra en el espejo, pero queda en `.stversions` hasta un año.

## Fuentes

- [Syncthing: Docker](https://github.com/syncthing/syncthing/blob/main/README-Docker.md), [arranque automático](https://docs.syncthing.net/users/autostart.html), [versiones](https://docs.syncthing.net/users/versioning.html), [patrones a ignorar](https://docs.syncthing.net/users/ignoring.html) y [tipos de carpeta](https://docs.syncthing.net/users/foldertypes.html).
- [Samba: smb.conf](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html) y [vfs_fruit](https://www.samba.org/samba/docs/current/man-html/vfs_fruit.8.html).
- [Tailscale: grants](https://tailscale.com/kb/1324/grants).
