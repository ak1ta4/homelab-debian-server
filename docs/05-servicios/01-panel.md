# Fase 5, paso 1: panel de inicio con Homepage

> **Versiones usadas:** Debian 13.7, Docker Engine 29.9.3, Compose 5.7.1, Homepage `v2.8.0`, AdGuard Home v0.107.80.

## 1. Qué vamos a hacer y por qué

**En sencillo:** una página de inicio con todos los servicios de casa: un clic para Jellyfin, Immich, Syncthing o AdGuard, sin recordar IPs ni puertos. Arriba muestra cómo va el servidor (CPU, RAM, discos), el tiempo y la hora. Se abre con un nombre fácil, `http://homelab.home.arpa`, que funciona en casa y fuera con Tailscale.

**Técnicamente:**
- Homepage en Docker, **sin root**, sin capacidades y **sin el socket de Docker**.
- Un nombre local, `homelab.home.arpa`, que AdGuard responde con la IP de casa o la de la tailnet según quién pregunte.
- Publicado en el 80 (sin número en la dirección), solo desde la LAN y la tailnet.
- Estilo gruvbox con un `custom.css`, como las páginas de inicio del dueño.

Después se añadieron puntos de estado y datos de cada servicio en las tarjetas (4.6 y 4.7). Beszel y Dozzle están en su [guía](01-monitorizacion.md), y los filtros de AdGuard, en la [Fase 4](../04-dns.md#48-filtros-de-amenazas).

Decisión y alternativas: [ADR 0013](../decisiones/0013-panel-con-homepage.md).

## 2. Conceptos previos

- **`home.arpa`:** el dominio reservado para redes de casa ([RFC 8375](https://www.rfc-editor.org/rfc/rfc8375)). No existe en internet, así que no choca con nada.
- **Reescritura de DNS:** AdGuard contesta con una IP fija para un nombre, en vez de preguntar fuera. Con `$client`, la regla solo vale para ciertos equipos.
- **Un enlace, dos redes:** en casa el servidor es `<IP_SERVIDOR>`; fuera, `<IP_TAILSCALE_SERVIDOR>`. Homepage solo admite un enlace por servicio, así que el nombre resuelve el problema: cada equipo recibe la IP que le sirve.
- **`HOMEPAGE_ALLOWED_HOSTS`:** los nombres con los que se puede abrir el panel. Si se abre con otro, Homepage lo rechaza.

## 3. Requisitos y estado inicial

- Docker ([Fase 1, paso 8](../01-base-y-endurecimiento/08-docker.md)), Tailscale ([Fase 3](../03-acceso-remoto.md)) y AdGuard como DNS de la casa y de la tailnet ([Fase 4](../04-dns.md)).
- Nada escuchando en el puerto 80 del servidor:
  ```bash
  ss -ltn | grep ':80 '
  ```
  **Resultado:** no sale nada.

## 4. Pasos

En el servidor, con `<USUARIO_ADMIN>`. Se hizo con dos scripts revisados antes de ejecutarlos (instalación y estilo); estos son sus pasos.

### 4.1 El nombre `homelab.home.arpa`

En AdGuard (`http://<IP_SERVIDOR>:3000`), **Filtros → Reglas de filtrado personalizadas**, al final:
```
||homelab.home.arpa^$client=<RED_LAN>,dnsrewrite=NOERROR;A;<IP_SERVIDOR>
||homelab.home.arpa^$client=100.64.0.0/10,dnsrewrite=NOERROR;A;<IP_TAILSCALE_SERVIDOR>
```
**Aplicar.** La documentación de AdGuard dice que, si varias reescrituras casan con la misma consulta, devuelve **todas**. Por eso cada regla lleva su `$client` y no se solapan.

Prueba desde el PC:
```
nslookup homelab.home.arpa <IP_SERVIDOR>
```
**Resultado:** **una sola** dirección, `<IP_SERVIDOR>`. "Servidor: UnKnown" es normal.

El propio servidor no usa AdGuard ([Fase 4, 4.5](../04-dns.md#45-el-servidor-no-usa-adguard)), así que en él este nombre no resuelve. No hace falta.

### 4.2 Usuario, carpetas y Compose

```bash
sudo adduser --system --group --no-create-home --home /nonexistent homepage
```
```bash
sudo install -d -m 755 /srv/homepage
```
```bash
sudo install -d -o homepage -g homepage -m 750 /srv/homepage/config
```
Copia [`compose.yaml`](../../stacks/homepage/compose.yaml) en `/srv/homepage/`, crea `.env` a partir de [`.env.example`](../../stacks/homepage/.env.example) y copia los archivos de [`config/`](../../stacks/homepage/config/) en `/srv/homepage/config/`, de `homepage` y con permisos `640`. Sustituye los marcadores de `widgets.yaml` (la ciudad y sus coordenadas, con dos decimales: las del pueblo, no las de casa) y de `bookmarks.yaml`.

Diferencias con el ejemplo oficial:
- `user:` con el usuario `homepage`, en vez de root con `PUID` y `PGID`. El script de arranque intenta un `chown`, avisa de que no puede y sigue.
- **Sin el socket de Docker:** la propia documentación lo desaconseja, y quien accede a él es root en el servidor. Sin él, el panel no ve los contenedores.
- `cap_drop: ALL` y `no-new-privileges`.
- `LOG_TARGETS: stdout`: los registros van a `docker logs`, no a la carpeta de configuración.
- `/srv/datos` montado `:ro`, solo para mostrar su espacio libre.
- La versión fijada (`v2.8.0`), no `latest`.

### 4.3 Reglas en `DOCKER-USER`

`/etc/systemd/system/homelab-homepage-lan.service`, con el mismo esquema que el de Syncthing ([paso 3, 4.2](03-nube.md#42-reglas-en-docker-user)):
```ini
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctstate DNAT --ctorigdstport 80 --ctdir ORIGINAL ! -s <RED_LAN> -j DROP
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctstate DNAT --ctorigdstport 80 --ctdir ORIGINAL -i tailscale0 -s 100.64.0.0/10 -j RETURN
```
Con su `[Unit]` (`After`, `Requires` y `PartOf=docker.service`), un `ExecStop=-… -D` por regla y `WantedBy=docker.service`.

```bash
sudo systemctl daemon-reload
```
```bash
sudo systemctl enable --now homelab-homepage-lan.service
```
En la política de Tailscale, `"tcp:80"` en la lista `ip` ([Fase 3, 4.1](../03-acceso-remoto.md#41-política-de-acceso)). Y `"tcp:8384"`, para que el enlace de Syncthing abra también fuera de casa. **Comprueba la política antes de seguir:** sin `"tcp:80"`, fuera de casa el panel no carga ni por la IP.

### 4.4 Arrancar

Con las reglas ya puestas:
```bash
sudo docker compose -f /srv/homepage/compose.yaml up -d
```
```bash
sudo docker top homepage -o pid,user
```
**Resultado:** `homepage`, no `root`. Y la portada, preguntando con un nombre permitido:
```bash
curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: <IP_SERVIDOR>' http://127.0.0.1/
```
**Resultado:** `200`. En `docker logs homepage`, el aviso `kubernetes.yaml was copied to the config folder` es normal.

En el PC, abre `http://homelab.home.arpa`, **con** el `http://`: sin él, algunos navegadores lo buscan en internet. Para ver un cambio de `custom.css`, recarga con **Ctrl + F5**.

### 4.5 El nombre en el propio servidor

El servidor no pregunta a AdGuard, así que en su escritorio el nombre no resuelve. A petición del dueño, una línea en `/etc/hosts`, con copia antes:
```bash
sudo cp -a /etc/hosts /etc/hosts.bak-AAAAMMDD
```
```bash
echo '127.0.0.1 homelab.home.arpa' | sudo tee -a /etc/hosts
```
```bash
getent hosts homelab.home.arpa
```
**Resultado:** `127.0.0.1  homelab.home.arpa`. Apunta al propio equipo y no a `<IP_SERVIDOR>`, para que siga valiendo aunque cambie la red.

### 4.6 La red `panel` y los puntos de estado

Homepage comprueba cada servicio y pide sus datos **desde su contenedor**. Por los puertos publicados no puede: `DOCKER-USER` solo deja entrar a la LAN y la tailnet. Por eso hay una red de Docker, `panel`, con Homepage y **solo la web** de cada servicio. Las bases de datos y la IA de Immich se quedan fuera.

```bash
sudo docker network create panel
```
En el `compose.yaml` de Immich (solo en `immich-server`), Jellyfin, AdGuard, Syncthing y Homepage (y después Beszel y Dozzle):
```yaml
    networks:
      - default
      - panel
```
Y al final de cada archivo:
```yaml
networks:
  panel:
    external: true
```
Un `up -d` en cada uno recrea el contenedor unos segundos. **AdGuard el último:** mientras se recrea, la casa usa el DNS de respaldo.

En `settings.yaml`, `statusStyle: dot`. En `services.yaml`, cada servicio lleva su `siteMonitor` con el nombre del contenedor: `http://immich_server:2283`, `http://jellyfin:8096`, `http://adguard:3000`, `https://syncthing:8384`, `http://beszel:8090` y `http://dozzle:8080`. Syncthing va por HTTPS con su certificado propio, y Homepage lo acepta.

### 4.7 Datos en las tarjetas

| Servicio | Credencial | Qué permite | Dónde se crea |
|---|---|---|---|
| Immich | Clave de API con **solo** `server.statistics` | Leer estadísticas. El código exige además que su dueño sea administrador | **Ajustes de la cuenta → Claves de API** |
| Jellyfin | Clave de API | Administrador completo: su documentación no ofrece claves limitadas | **Panel de control → Claves de API** |
| AdGuard | Un usuario propio, `panel`, con contraseña aleatoria | Administrador completo: AdGuard no tiene usuarios de solo lectura | `users:` de `AdGuardHome.yaml`, con AdGuard parado |

Syncthing se queda sin datos a propósito: su clave da **control total**, y con ella se podrían compartir las carpetas del PC con otro equipo.

Las claves van en `/srv/homepage/secretos/` (carpeta `700` y archivos `400`, de `homepage`), escritas **sin salto de línea** (`printf '%s'`): Homepage mete el archivo tal cual. Se montan en `/app/secretos:ro` y se nombran en `compose.yaml`:
```yaml
      HOMEPAGE_FILE_IMMICH: /app/secretos/immich
```
En `services.yaml`, `key: "{{HOMEPAGE_FILE_IMMICH}}"`. Así no aparecen en los YAML, ni en `docker inspect`, ni en el repositorio.

El usuario `panel` de AdGuard se creó con un script: genera la contraseña con `openssl rand`, la guarda en `secretos/adguard`, calcula su hash con `mkpasswd -m bcrypt` y lo añade a `users:` con AdGuard parado, validando el YAML antes de arrancarlo. **Nadie llega a ver esa contraseña.**

En Jellyfin, `enableMediaControl: false`: el panel no puede pausar lo que se esté viendo.

## 5. Verificación

- [x] `homelab.home.arpa` da una sola IP desde el PC, la de casa.
- [x] Contenedor `healthy`, con el usuario `homepage`; portada `200`.
- [x] Reglas del 80 arriba del todo en `DOCKER-USER`.
- [x] En el PC: recursos, tiempo, hora, grupos con iconos, marcadores y estilo gruvbox. Los enlaces abren.
- [x] Desde el iPhone fuera de casa, con datos y Tailscale: el panel abre por el nombre y los enlaces funcionan (tras añadir `"tcp:80"` y `"tcp:8384"` a la política).
- [x] En el escritorio del servidor, el nombre abre el panel y Beszel.
- [x] Seis puntos verdes: Jellyfin, Immich, Syncthing, AdGuard, Beszel y Dozzle.
- [x] Datos en las tarjetas: películas en Jellyfin; fotos y vídeos en Immich; consultas, bloqueos y latencia en AdGuard.
- [ ] Tras un reinicio: las reglas vuelven y el panel arranca. En la sesión en persona.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| Un icono sale como el texto "logo" | No existe con ese nombre en Dashboard Icons (pasó con `samba.png`) | Un icono `mdi-…`, como `mdi-folder-network-outline` |
| Un icono casi no se ve | Es negro sobre fondo oscuro (pasó con Tailscale) | Simple Icons con color: `si-tailscale-#ebdbb2` |
| El estilo no cambia tras editar `custom.css` | El navegador guarda la hoja de estilo | **Ctrl + F5** |
| `homelab.home.arpa` abre un buscador | Escrito sin `http://` | Escribir `http://homelab.home.arpa` |
| `nslookup` devuelve dos IP | Dos reescrituras casan con el mismo equipo | Cada regla con su `$client`, sin solaparse |
| Fuera de casa no carga, ni por el nombre ni por la IP de la tailnet | Falta `"tcp:80"` en la política (pasó aquí) | Añadirlo en **Access controls** |
| Abre por la IP pero Homepage rechaza el nombre | El nombre no está en `HOMEPAGE_ALLOWED_HOSTS` | Añadirlo en `.env` y recrear el contenedor (`up -d`) |
| Fuera de casa, el enlace de Syncthing no abre | Falta `"tcp:8384"` en la política | Añadirlo |
| `Skipping ownership changes` en el registro | Corre sin root, como se quiere | Nada |
| Tras reiniciar AdGuard, "Server not found" con el nombre, pero la IP abre | Windows pasó al DNS de respaldo y se queda con él un rato (pasó aquí) | Desactivar y activar la red del PC, o esperar unos 15 minutos |
| En el escritorio del servidor el nombre no abre | El servidor no usa AdGuard | La línea de `/etc/hosts` (4.5) |
| Immich dice `0 B` de almacenamiento | Solo cuenta lo subido a Immich; la biblioteca externa del PC no | Nada |
| La latencia de AdGuard sale como "4,392 ms" | La coma es el decimal: 4,4 milisegundos | Nada |

## 7. Cómo revertirlo

```bash
sudo docker compose -f /srv/homepage/compose.yaml down
```
```bash
sudo systemctl disable --now homelab-homepage-lan.service
```
```bash
sudo rm /etc/systemd/system/homelab-homepage-lan.service
```
Quita `"tcp:80"` (y, si no se quiere el panel de Syncthing desde fuera, `"tcp:8384"`) de la política de Tailscale y las dos reglas de AdGuard. `/srv/homepage` solo tiene configuración; se puede borrar. Para volver a un estilo anterior, los scripts dejaron copias en `/srv/homepage/config.bak-<fecha>`.

Para quitar solo los datos de las tarjetas: borra las claves `panel` en Immich y Jellyfin, el usuario `panel` en AdGuard (copia `AdGuardHome.yaml.bak-<fecha>`) y `/srv/homepage/secretos`. Para quitar la red `panel`, cada servicio tiene su `compose.yaml.bak-<fecha>`. La línea de `/etc/hosts` se quita restaurando su copia.

## 8. Seguridad

**Mitiga:**
- Sin socket de Docker: un fallo en el panel no da el control de los contenedores ni del servidor.
- Usuario sin privilegios, sin capacidades y `no-new-privileges`; el disco de datos, en solo lectura.
- Solo desde la LAN y la tailnet; nada sale a internet salvo la consulta del tiempo.
- Claves fuera de los YAML y del entorno, en archivos `400` montados `:ro`. Cada servicio tiene su credencial `panel`, que se puede borrar sin tocar las demás.
- La red `panel` solo une las webs: desde el panel no se llega a las bases de datos.

**Queda:**
- **Quien comprometa el panel tiene las claves** de Jellyfin y AdGuard, que son de administrador. La de Immich solo lee estadísticas. Es la contrapartida de ver los datos; el dueño la aceptó.
- **Sin contraseña:** cualquiera en el wifi de casa o en la tailnet lo ve. Solo muestra enlaces y el uso de recursos; cada servicio sigue pidiendo la suya.
- **HTTP**, hasta el dominio y Caddy (paso 5.0).
- El contenedor consulta el tiempo a Open-Meteo con las coordenadas del pueblo: sabe la zona, no la casa.
- Homepage tuvo avisos de seguridad en 2026 (el más grave, saltarse la autenticación, arreglado en v2.6.0). Aquí no hay autenticación que saltar, pero hay que actualizarlo.

## 9. Para la entrevista

**En 30 segundos:** "Monté Homepage sin root y sin el socket de Docker, que es como dar root a quien comprometa el panel. El problema era que un servicio solo admite un enlace y el servidor tiene dos IP, la de casa y la de la tailnet. Lo resolví con un nombre `home.arpa` en AdGuard y reglas por cliente: cada equipo recibe la IP que le sirve. El firewall está en `DOCKER-USER`, porque Docker se salta UFW."

**Preguntas posibles:**
- *¿Por qué `home.arpa` y no `.local`?* `.local` es de mDNS y da problemas con los equipos de Apple. `home.arpa` está reservado para esto.
- *¿Por qué no el socket con un proxy?* Daría el estado de los contenedores, pero no hace falta: los puntos de estado van por la red `panel`, sin tocar Docker.
- *¿Dónde guardas las claves?* En archivos `400` montados `:ro`, que Homepage lee con `HOMEPAGE_FILE_*`. Variables de entorno no: salen en `docker inspect`.
- *¿Cómo cambiaste los colores?* Homepage define cada paleta como variables CSS en el elemento `html`. `custom.css` sustituye las del tema `stone` por las de gruvbox.

## Fuentes

- Homepage: [instalación con Docker](https://gethomepage.dev/installation/docker/), [ajustes](https://gethomepage.dev/configs/settings/), [servicios](https://gethomepage.dev/configs/services/), [marcadores](https://gethomepage.dev/configs/bookmarks/), [CSS propio](https://gethomepage.dev/configs/custom-css-js/), [recursos](https://gethomepage.dev/widgets/info/resources/), [Open-Meteo](https://gethomepage.dev/widgets/info/openmeteo/), [Docker](https://gethomepage.dev/configs/docker/) y [avisos de seguridad](https://github.com/gethomepage/homepage/security/advisories).
- [Código de la v2.8.0](https://github.com/gethomepage/homepage/tree/v2.8.0): `docker-entrypoint.sh` y `src/styles/theme.css`.
- AdGuard: [sintaxis de reglas](https://adguard-dns.io/kb/general/dns-filtering-syntax/) (`$dnsrewrite` y `$client`).
- [RFC 8375](https://www.rfc-editor.org/rfc/rfc8375): `home.arpa`.
