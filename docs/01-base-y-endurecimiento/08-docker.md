# Paso 8: Docker Engine y Compose

> **Versiones usadas:** Debian 13.7, Docker Engine 29.9.3, containerd 2.4.2, Docker Compose 5.7.1, Buildx 0.39.0.

## 1. Qué vamos a hacer y por qué

**En sencillo:** Docker permite instalar cada servicio (fotos, notas, DNS…) en su propia "caja" con todo lo que necesita, sin mezclarlo con el sistema. Lo instalamos desde los creadores de Docker y lo configuramos para que, por defecto, nada de lo que pongamos en esas cajas quede abierto a la red de casa por descuido.

**Técnicamente:**
- Docker Engine y el plugin de Compose desde el repositorio oficial de Docker.
- `daemon.json`: registros con rotación (`local`), `live-restore` y puertos publicados en `127.0.0.1` por defecto ([ADR 0008](../decisiones/0008-docker-y-ufw.md)).
- Docker entra en las actualizaciones automáticas.
- Siempre con `sudo`: el grupo `docker` queda vacío.

## 2. Conceptos previos

- **Contenedor:** un proceso aislado con su propio sistema de archivos, creado a partir de una **imagen**.
- **Compose:** describe uno o varios contenedores en un archivo `compose.yaml`.
- **Publicar un puerto (`-p`):** hace accesible un puerto del contenedor desde el equipo. Docker crea sus propias reglas de firewall y **se salta UFW**.
- **Grupo `docker`:** quien puede hablar con Docker puede montar `/` en un contenedor y hacer lo que quiera: equivale a root.

## 3. Requisitos y estado inicial

- Sin paquetes en conflicto: `docker.io`, `docker-compose`, `docker-doc`, `docker-buildx`, `podman-docker`, `containerd` y `runc`, ninguno instalado.
- UFW activo ([paso 7](07-firewall.md)).
- Debian 13 está soportado por Docker.

## 4. Pasos

En el servidor, con `<USUARIO_ADMIN>`. Pega los comandos **de uno en uno**.

### 4.1 Repositorio oficial

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```
```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
```
```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
```bash
gpg --show-keys --with-fingerprint /etc/apt/keyrings/docker.asc
```
Aquí, `9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`. La guía actual de Docker ya no publica la huella: se comparó con una segunda descarga por otro camino, y era idéntica byte a byte.

`/etc/apt/sources.list.d/docker.sources` (`sudo nano`), en formato deb822 como en la guía oficial:
```
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: trixie
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
```

### 4.2 Instalar

```bash
sudo apt update
```
```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
```bash
docker --version
```
```bash
docker compose version
```
Docker arranca solo y queda habilitado. **No** añadas tu usuario al grupo `docker`.

### 4.3 Configurar el servicio

`/etc/docker/daemon.json`:
```json
{
  "log-driver": "local",
  "live-restore": true,
  "ip": "127.0.0.1"
}
```
- `log-driver: local`: el de serie (`json-file`) no rota los registros y puede llenar el disco.
- `live-restore`: los contenedores siguen vivos si Docker se reinicia o se actualiza a una versión de parche.
- `ip: 127.0.0.1`: un `-p 8080:80` sin IP escucha solo en el propio equipo. Para publicar en la LAN hay que escribir la IP: `-p <IP_SERVIDOR>:8080:80`.

```bash
sudo dockerd --validate --config-file /etc/docker/daemon.json
```
✅ `configuration OK`.
```bash
sudo systemctl restart docker
```
```bash
sudo docker info --format 'logs={{.LoggingDriver}} live-restore={{.LiveRestoreEnabled}}'
```
✅ `logs=local live-restore=true`.

### 4.4 Actualizaciones automáticas

`/etc/apt/apt.conf.d/53homelab-unattended-upgrades-docker`:
```
// Homelab, Fase 1, paso 8: Docker también se actualiza solo.
Unattended-Upgrade::Origins-Pattern {
        "origin=Docker,label=Docker CE,codename=trixie";
};
```
La lista se **suma** a los orígenes de Debian, no los sustituye:
```bash
sudo unattended-upgrade --dry-run --debug 2>&1 | grep "Allowed origins"
```
✅ Los tres orígenes de Debian y `origin=Docker,label=Docker CE,codename=trixie`.

### 4.5 Prueba

```bash
sudo docker run -d --name prueba-homelab -p 8088:8080 busybox httpd -f -p 8080
```
```bash
ss -tlnH 'sport = :8088'
```
✅ `127.0.0.1:8088`, y **no** `0.0.0.0:8088`.
```bash
sudo docker rm -f prueba-homelab
```
```bash
sudo docker image rm busybox
```

## 5. Verificación

- [x] Clave del repositorio: la misma huella por dos caminos.
- [x] Docker 29.9.3 y Compose 5.7.1, activos y habilitados.
- [x] `dockerd --validate`: configuración correcta; `docker info`: `local` y `live-restore`.
- [x] Un puerto publicado sin IP escucha en `127.0.0.1`.
- [x] Docker figura entre los orígenes de las actualizaciones automáticas.
- [x] Grupo `docker` vacío.
- [x] El filtro de salida del agente sigue funcionando con Docker activo, incluido el rechazo hacia `172.17.0.1`.
- [ ] Tras un reinicio: Docker arranca. En la sesión en persona.
- [ ] Escaneo de puertos desde otro equipo de la LAN: Fase 8.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `permission denied while trying to connect to the Docker daemon socket` | Sin `sudo` y sin grupo `docker` | Usa `sudo docker …`, como está decidido |
| Un servicio no se ve desde otro equipo de casa | Publicado sin IP: escucha en `127.0.0.1` | Publica con `<IP_SERVIDOR>:puerto` y limita el origen en `DOCKER-USER` |
| Un puerto se ve desde la LAN aunque UFW no lo permite | Publicado con IP: Docker se salta UFW | Es lo esperado: limita en `DOCKER-USER` (ver ADR 0008) |
| Tras una actualización, los contenedores se reiniciaron | Actualización grande (`AA.MM`): `live-restore` no la cubre | Normal, de madrugada |

## 7. Cómo revertirlo

```bash
sudo apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
```bash
sudo rm /etc/apt/sources.list.d/docker.sources /etc/apt/keyrings/docker.asc
```
```bash
sudo rm /etc/apt/apt.conf.d/53homelab-unattended-upgrades-docker
```
Los datos de Docker (`/var/lib/docker`, `/var/lib/containerd` y `/etc/docker`) no se borran solos. Bórralos solo si no queda nada que conservar.

## 8. Seguridad

**Mitiga:**
- Puertos abiertos a la LAN por descuido: por defecto, solo `127.0.0.1`.
- Disco lleno por los registros: rotación.
- Escalada a root a través de Docker: nadie en el grupo `docker`; el agente no tiene acceso al socket.
- Versiones con vulnerabilidades: actualización automática.

**Queda:**
- Lo publicado con IP se salta UFW: hay que limitarlo en `DOCKER-USER` servicio a servicio.
- Las imágenes de terceros pueden traer vulnerabilidades: se fijarán versiones y se revisarán en cada fase.
- Docker no da soporte oficial a tablas nftables propias, como el filtro del agente: se repite la prueba tras cada actualización grande.
- Rootless Docker no se usa, para no complicar Immich ni el acceso a `/srv/datos`.

## 9. Para la entrevista

**En 30 segundos:** "Instalé Docker desde su repositorio oficial y comprobé la clave descargándola por dos caminos, porque la documentación ya no publica la huella. Como los puertos de Docker se saltan UFW, puse `"ip": "127.0.0.1"` en `daemon.json`: lo publicado sin IP no sale del equipo, y abrir algo a la LAN exige escribir la IP y limitarlo en `DOCKER-USER`. Además activé la rotación de registros y `live-restore`, y no metí a nadie en el grupo `docker`, porque equivale a root."

**Preguntas posibles:**
- *¿Por qué Docker se salta UFW?* Porque redirige el tráfico de los puertos publicados con DNAT en la tabla `nat`, antes de las cadenas de entrada donde UFW filtra.
- *¿Por qué no usar el grupo `docker`?* Con acceso al socket se puede montar `/` en un contenedor con root dentro. El administrador ya tiene `sudo`, así que el grupo no aporta nada y amplía el riesgo.
- *¿Para qué `live-restore`?* Para que una actualización de parche de Docker no reinicie los contenedores.
- *¿Por qué no el backend nftables de Docker?* Es experimental en Docker 29 según su propia documentación, y no tiene `DOCKER-USER`.

## Fuentes

- [Install Docker Engine on Debian](https://docs.docker.com/engine/install/debian/).
- [Packet filtering and firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/), [iptables](https://docs.docker.com/engine/network/firewall-iptables/) y [nftables](https://docs.docker.com/engine/network/firewall-nftables/).
- [Live restore](https://docs.docker.com/engine/daemon/live-restore/) y [Configure logging drivers](https://docs.docker.com/engine/logging/configure/).
- `dockerd --help` de Docker 29.9.3.
- [How To Secure A Linux Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server) (CC BY-SA 4.0), adaptada: ver [`../guia-securizacion.md`](../guia-securizacion.md).
