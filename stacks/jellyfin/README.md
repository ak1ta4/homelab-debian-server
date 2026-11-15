# Jellyfin: películas y series en casa

> **Versiones usadas:** Debian 13.7, Docker Engine 29.9.3, Compose 5.7.1, imagen `jellyfin/jellyfin:12.4` (ffmpeg 8.1.3), Mesa 26.0.8 dentro de la imagen.
>
> **Fuera de orden:** pedido por el dueño durante la Fase 1 (2026-11-15), antes que el resto de servicios.

## 1. Qué vamos a hacer y por qué

**En sencillo:** un "Netflix casero". Las películas se guardan en el disco de datos y se ven desde la tele, el móvil o el PC, solo dentro de casa.

**Técnicamente:**
- Imagen oficial `jellyfin/jellyfin` con Compose, en red *bridge* y con un usuario de sistema propio (`jellyfin`), sin capacidades.
- Vídeos en un subvolumen btrfs propio (`/srv/datos/Videos`), montado **solo lectura** en el contenedor.
- Conversión de vídeo por hardware con VA-API en la gráfica AMD integrada (`renderD128`, la misma que usa Sunshine). La NVIDIA no se toca.
- Puerto 8096 publicado en todas las IPv4 y limitado a `<RED_LAN>` y a la tailnet en la cadena `DOCKER-USER` ([ADR 0008](../../docs/decisiones/0008-docker-y-ufw.md), [ADR 0009](../../docs/decisiones/0009-tailscale.md)). Al principio se publicaba solo en `<IP_SERVIDOR>`; el cambio y la regla actual están en la [Fase 3](../../docs/03-acceso-remoto.md#44-firewall).

## 2. Conceptos previos

- **Transcodificar:** convertir el vídeo al vuelo cuando el dispositivo no admite el formato. Con la gráfica gasta mucho menos procesador.
- **Reproducción directa:** el dispositivo admite el archivo tal cual y no se convierte nada.
- **VA-API:** la interfaz de Linux para usar la gráfica al decodificar y codificar vídeo.
- **`DOCKER-USER`:** la cadena del firewall donde Docker deja poner reglas propias; los puertos publicados por Docker no pasan por UFW.

## 3. Requisitos y estado inicial

- Docker y su `daemon.json` del [paso 8](../../docs/01-base-y-endurecimiento/08-docker.md).
- Disco de datos montado en `/srv/datos` ([paso 5](../../docs/01-base-y-endurecimiento/05-disco-de-datos.md)).
- `/dev/dri/renderD128` es la AMD, del grupo `render` ([auditoría](../../docs/00-auditoria.md)).

## 4. Pasos

En el servidor, con `<USUARIO_ADMIN>`. Se hizo con dos scripts revisados antes de ejecutarlos; estos son sus pasos.

### 4.1 Usuario, carpetas y configuración

```bash
sudo adduser --system --group --no-create-home --home /nonexistent jellyfin
```
```bash
sudo btrfs subvolume create /srv/datos/Videos
```
```bash
sudo mkdir /srv/datos/Videos/Películas
```
```bash
sudo chown -R <USUARIO_ADMIN>: /srv/datos/Videos
```
El administrador copia las películas; Jellyfin solo las lee (permisos `755`).

```bash
sudo install -d -m 755 /srv/jellyfin
```
```bash
sudo install -d -o jellyfin -g jellyfin -m 700 /srv/jellyfin/config /srv/jellyfin/cache
```
Copia [`compose.yaml`](compose.yaml) en `/srv/jellyfin/` y crea `.env` a partir de [`.env.example`](.env.example). Después:
```bash
sudo docker compose -f /srv/jellyfin/compose.yaml config --quiet
```
```bash
sudo docker compose -f /srv/jellyfin/compose.yaml pull
```

### 4.2 Limitar el puerto a la LAN y arrancar

`/etc/systemd/system/homelab-jellyfin-lan.service` pone la regla cada vez que arranca Docker (las reglas de `iptables` no sobreviven a un reinicio):
```ini
[Unit]
Description=Homelab: Jellyfin (8096) solo desde la LAN, en DOCKER-USER
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctorigdstport 8096 --ctdir ORIGINAL ! -s <RED_LAN> -j DROP
ExecStop=-/usr/sbin/iptables -D DOCKER-USER -p tcp -m conntrack --ctorigdstport 8096 --ctdir ORIGINAL ! -s <RED_LAN> -j DROP

[Install]
WantedBy=docker.service
```
`--ctorigdstport` mira el puerto **original**, antes de que Docker lo redirija al contenedor.

```bash
sudo systemctl daemon-reload
```
```bash
sudo systemctl enable --now homelab-jellyfin-lan.service
```
```bash
sudo iptables -S DOCKER-USER
```
✅ La regla con `DROP`. Solo entonces:
```bash
sudo docker compose -f /srv/jellyfin/compose.yaml up -d
```

### 4.3 Asistente inicial

Desde el navegador del PC, en `http://<IP_SERVIDOR>:8096`, **del tirón y hasta pulsar "Terminar"**:
- Usuario administrador con contraseña fuerte, guardada en el gestor de contraseñas.
- Biblioteca de películas: elige la carpeta con clics (`/media/Películas`), sin escribirla.
- Desmarca la asignación automática de puertos (UPnP).

### 4.4 Gráfica

**Panel de control** → **Reproducción** → **Transcodificación**:
- VAAPI, dispositivo `/dev/dri/renderD128`.
- Decodificación: H264, HEVC, MPEG2, VC1, VP9, HEVC 10 bits y VP9 10 bits. **Sin AV1**: la Vega de los Ryzen 5000 no lo decodifica.
- Permitir codificación HEVC.

### 4.5 Añadir películas

En `/srv/datos/Videos/Películas`, con el nombre `Título (Año)/Título (Año).mkv`:
- desde el PC: `scp "Título (Año).mkv" <USUARIO_ADMIN>@<IP_SERVIDOR>:/srv/datos/Videos/Películas/` (pide el TOTP);
- o con Moonlight, desde el gestor de archivos del servidor.

Después, **Panel de control** → **Bibliotecas** → **Escanear todas las bibliotecas**.

## 5. Verificación

- [x] La regla de `DOCKER-USER` está puesta y el contenedor queda `healthy`, publicado solo en `<IP_SERVIDOR>:8096`.
- [x] `vainfo` dentro del contenedor abre la AMD (`radeonsi`, `renoir`):
  ```bash
  sudo docker exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128
  ```
- [x] Una película se ve en la tele LG (app webOS) con reproducción directa.
- [x] Jellyfin ve la IP real del cliente de la LAN, no la de Docker.
- [x] Al bajar la calidad a 8 Mbps en la tele, el panel muestra "Transcodificación" y se ve sin cortes.
- [ ] Tras un reinicio: la regla vuelve y Jellyfin arranca. En la sesión en persona.
- [ ] Desde fuera de la LAN no llega: escaneo de la Fase 8.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| "Ha habido un error procesando la solicitud" al crear el usuario del asistente | Asistente empezado en un dispositivo y repetido en otro: el primero ya había creado el usuario sin terminar el asistente (causa probable) | Empezar de cero (abajo) y hacerlo del tirón en un solo dispositivo |
| El asistente "no encuentra la ruta" de la biblioteca | Ruta escrita a mano (la tilde) | Elegirla con clics en el selector |
| `sudo cat …/passwordreset*.json`: no existe | El `*` lo expande tu usuario, que no puede leer `config/` | `sudo sh -c 'cat …/passwordreset*.json'`. Aquí ni así apareció el archivo; se optó por empezar de cero |
| Cortes en reproducción directa por Wi-Fi | El enlace Wi-Fi con la tele bajó a ~6 Mbit/s (el router mostraba `866/6`) con buena señal; la película pide 17 de media | Medir dónde está el atasco (abajo). Aquí, por cable va bien. En el router, 5 GHz en un canal sin DFS (36) a 80 MHz |
| Un dispositivo de casa no conecta | IP del servidor cambiada o la regla con otra red | `sudo iptables -S DOCKER-USER` y `ss -tlnH 'sport = :8096'` |

**Dónde está el atasco.** Con la película en marcha:
```bash
sudo nsenter -n -t <PID de jellyfin> ss -tin
```
Si la conexión de vídeo tiene mucho `Send-Q`/`notsent` y un `rtt` de segundos frente a un `minrtt` de milisegundos, el servidor envía pero el camino o el cliente no da abasto. Si `tc -s qdisc show dev <interfaz>` no tiene `backlog`, la cola no está en el servidor. El tráfico de Jellyfin se mide en el `veth` del contenedor (`/sys/class/net/veth…/statistics/rx_bytes`).

**Empezar de cero** (borra solo la configuración de Jellyfin, no los vídeos):
```bash
sudo docker compose -f /srv/jellyfin/compose.yaml down
```
```bash
sudo rm -rf /srv/jellyfin/config /srv/jellyfin/cache
```
```bash
sudo install -d -o jellyfin -g jellyfin -m 700 /srv/jellyfin/config /srv/jellyfin/cache
```
```bash
sudo docker compose -f /srv/jellyfin/compose.yaml up -d
```

## 7. Cómo revertirlo

```bash
sudo docker compose -f /srv/jellyfin/compose.yaml down
```
```bash
sudo systemctl disable --now homelab-jellyfin-lan.service
```
```bash
sudo rm /etc/systemd/system/homelab-jellyfin-lan.service
```
`/srv/jellyfin` y `/srv/datos/Videos` no se borran solos.

## 8. Seguridad

**Mitiga:**
- Exposición: solo la LAN, con regla en `DOCKER-USER`, y nada en el router.
- Contenedor comprometido: usuario sin privilegios, sin capacidades, `no-new-privileges` y vídeos en solo lectura.
- Sin NVIDIA ni su kit para Docker: no se tocan esos drivers.

**Queda:**
- HTTP sin cifrar dentro de casa: las contraseñas viajan en claro por la LAN hasta que llegue Caddy (Fase 4).
- La imagen está fijada en `12.4`: las actualizaciones son manuales hasta definir cómo se actualizan los contenedores.
- Sin autodescubrimiento (UDP 7359) ni DLNA: los dispositivos se conectan con la dirección a mano.
- Cuando llegue Tailscale (Fase 3) habrá que decidir si Jellyfin se ve también desde la tailnet.

## 9. Para la entrevista

**En 30 segundos:** "Monté Jellyfin con la imagen oficial, con un usuario propio sin capacidades y los vídeos en solo lectura. Para la conversión de vídeo usé VA-API en la gráfica integrada AMD en vez de la NVIDIA, para no tocar sus drivers. Como Docker se salta UFW, publiqué el puerto solo en la IP de la LAN y añadí una regla en `DOCKER-USER`, que un servicio de systemd repone cada vez que arranca Docker."

**Preguntas posibles:**
- *¿Por qué `--ctorigdstport` y no `--dport`?* En `DOCKER-USER` el paquete ya pasó el DNAT: `--dport` vería el puerto del contenedor. Aquí coinciden, pero así la regla sigue valiendo si cambian.
- *¿Por qué no `network_mode: host`?* Solo hace falta para DLNA, y con host el puerto lo controlaría UFW pero el contenedor vería todas las interfaces del equipo.

## Fuentes

- [Jellyfin: Container](https://jellyfin.org/docs/general/installation/container/).
- [Jellyfin: AMD hardware acceleration](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/amd/).
- [Docker: Packet filtering and firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/).
- [Lanzamiento de Jellyfin 12.4](https://github.com/jellyfin/jellyfin/releases/latest) y las etiquetas en Docker Hub (2026-11-15).
