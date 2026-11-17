# Fase 5, paso 1: monitorización con Beszel y Dozzle

> **Versiones usadas:** Debian 13.7, Docker Engine 29.9.3, Compose 5.7.1, Beszel `0.21.2` (hub y agente), Dozzle `v11.4.3`, `tecnativa/docker-socket-proxy` (`lts`, fijado por digest).

## 1. Qué vamos a hacer y por qué

**En sencillo:** el [panel](01-panel.md) dice cómo está el servidor **ahora**. Beszel guarda **el historial**: gráficas de CPU, RAM, discos, red, temperatura, batería y la gráfica AMD, y más adelante avisará por Telegram (paso 5.2). Dozzle enseña en la web **los registros de los contenedores**, en directo y con buscador, en vez de `sudo docker logs`.

**Técnicamente:**
- **Beszel:** la web ("hub") en Docker, sin root. El agente que mide es un **servicio del sistema** con su propio usuario, **sin acceso a Docker**.
- **Dozzle:** sin root y con usuario y contraseña. **No toca el socket de Docker:** habla con un proxy que solo deja leer.
- Las dos webs, solo desde la LAN y la tailnet.

Decisión y alternativas: [ADR 0014](../decisiones/0014-beszel-y-dozzle.md).

## 2. Conceptos previos

- **Socket de Docker** (`/var/run/docker.sock`): quien habla con él manda en Docker, y por tanto es root en el servidor. Montarlo `:ro` **no** limita nada: solo marca el archivo, no la API.
- **Proxy del socket:** un contenedor que sí tiene el socket y solo deja pasar ciertas peticiones. Con `POST=0`, solo lecturas (`GET` y `HEAD`): nada de crear, parar ni borrar.
- **Hub y agente:** el hub guarda los datos y hace las gráficas; el agente mide el equipo y se los manda. Aquí el agente se conecta al hub por WebSocket, con un token.

## 3. Requisitos y estado inicial

- La red `panel` del [panel, 4.6](01-panel.md#46-la-red-panel-y-los-puntos-de-estado).
- Puertos 8090 y 8080 libres:
  ```bash
  ss -ltn | grep -E ':(8090|8080) '
  ```
  **Resultado:** no sale nada.
- En la política de Tailscale, `"tcp:8090"` y `"tcp:8080"` en la lista `ip`.

## 4. Pasos

En el servidor, con `<USUARIO_ADMIN>`. Se hizo con tres scripts revisados antes de ejecutarlos; estos son sus pasos.

### 4.1 Beszel: el hub

```bash
sudo adduser --system --group --no-create-home --home /nonexistent beszel-hub
```
```bash
sudo install -d -m 755 /srv/beszel
```
```bash
sudo install -d -o beszel-hub -g beszel-hub -m 700 /srv/beszel/datos
```
Copia [`compose.yaml`](../../stacks/beszel/compose.yaml) y crea `.env` a partir de [`.env.example`](../../stacks/beszel/.env.example). Diferencias con el ejemplo oficial:
- `user:` con `beszel-hub`. La imagen es `scratch` y no declara usuario, así que por defecto sería root. **Probado:** funciona sin root.
- `tmpfs: /tmp`, porque la imagen no trae `/tmp`.
- `cap_drop: ALL`, `no-new-privileges` y la imagen fijada por digest.

Reglas en `DOCKER-USER` con `homelab-beszel-lan.service`, igual que las del [panel](01-panel.md#43-reglas-en-docker-user) pero con el 8090. Después:
```bash
sudo docker compose -f /srv/beszel/compose.yaml up -d
```
```bash
sudo docker top beszel -o pid,user
```
**Resultado:** `beszel-hub`. En el PC, `http://homelab.home.arpa:8090`: crear la cuenta y **Añadir sistema** (nombre `homelab`, host `<IP_SERVIDOR>`, el puerto que trae). De esa ventana solo se copian la **KEY** (*Public Key*, empieza por `ssh-ed25519`) y el **TOKEN**. **No se ejecuta el comando que propone.**

### 4.2 Beszel: el agente

**No se usa el `.deb` oficial.** Su `postinst` mete al usuario `beszel` en el grupo `docker`, que equivale a root. Se instala el binario a mano:
```bash
curl -fsSLO https://github.com/henrygd/beszel/releases/download/v0.21.2/beszel-agent_linux_amd64.tar.gz
```
```bash
curl -fsSL https://github.com/henrygd/beszel/releases/download/v0.21.2/beszel_0.21.2_checksums.txt | grep 'agent_linux_amd64.tar.gz' | sha256sum -c
```
**Resultado:** `OK`.
```bash
tar -xzf beszel-agent_linux_amd64.tar.gz beszel-agent
```
```bash
sudo install -o root -g root -m 755 beszel-agent /usr/local/bin/beszel-agent
```
```bash
sudo adduser --system --group --no-create-home --home /nonexistent beszel
```
`/etc/beszel-agent.conf` (de `beszel`, `600`, lleva el token) a partir de [`beszel-agent.conf.example`](../../stacks/beszel/beszel-agent.conf.example):
- `HUB_URL=http://127.0.0.1:8090` y `DISABLE_SSH=true`: el agente no abre ningún puerto; se conecta él al hub.
- `DOCKER_HOST=` vacío: sin Docker.
- `GPU_COLLECTOR=amd_sysfs`: solo la AMD. Preguntar a la NVIDIA cada pocos segundos la despertaría y gastaría batería.
- `EXTRA_FILESYSTEMS=/srv/datos__Datos`: el disco de datos.

[`beszel-agent.service`](../../stacks/beszel/beszel-agent.service) en `/etc/systemd/system/`. Es el del `.deb`, con `User=beszel`, `NoNewPrivileges`, sin capacidades y con más protecciones de systemd.
```bash
sudo systemctl daemon-reload
```
```bash
sudo systemctl enable --now beszel-agent.service
```
**Resultado:** el sistema sale en verde en la web.

### 4.3 Dozzle

```bash
sudo adduser --system --group --no-create-home --home /nonexistent dozzle
```
```bash
sudo install -d -m 755 /srv/dozzle
```
```bash
sudo install -d -o dozzle -g dozzle -m 700 /srv/dozzle/datos
```
`users.yml` con la propia imagen. La contraseña va por la entrada estándar, así que no queda en el historial ni en la lista de procesos:
```bash
printf '%s\n' "$CONTRASEÑA" | sudo docker run -i --rm --network none <imagen de Dozzle> generate <usuario> | sudo tee /srv/dozzle/datos/users.yml >/dev/null
```
El script pedía la contraseña dos veces sin mostrarla y comprobaba que el resultado era YAML con un hash bcrypt.

Copia [`compose.yaml`](../../stacks/dozzle/compose.yaml) y `.env`. Dos contenedores:
- **`dozzle_proxy`:** el único con el socket. `CONTAINERS=1`, `INFO=1`, `EVENTS=1` y `POST=0`. Solo está en la red `interna` (`internal: true`): sin salida y sin puertos.
- **`dozzle`:** `DOZZLE_REMOTE_HOST=tcp://dozzle_proxy:2375`, `DOZZLE_AUTH_PROVIDER=simple`, sin acciones, sin consola y sin analítica. Usuario `dozzle` y `cap_drop: ALL`. Redes `default`, `interna` y `panel`.

Reglas en `DOCKER-USER` con `homelab-dozzle-lan.service` (8080). Después:
```bash
sudo docker compose -f /srv/dozzle/compose.yaml up -d
```

## 5. Verificación

Que el proxy no deja escribir. Se intenta reiniciar un contenedor a través de él:
```bash
sudo docker exec dozzle_proxy wget -q -O /dev/null -S --post-data= http://127.0.0.1:2375/containers/dozzle/restart
```
**Resultado:** `HTTP/1.0 403 Forbidden`.

- [x] Hub `healthy`, con el usuario `beszel-hub`; web `200`.
- [x] Agente activo como `beszel`, sin grupos extra, sin puertos abiertos y con `CapabilityBoundingSet` vacío. El sistema, en verde.
- [x] Dozzle `healthy`, como `dozzle`; sin sesión redirige a la entrada (`307`).
- [x] El proxy responde `403` a un `POST`.
- [x] Dozzle ve los 11 contenedores, con CPU, memoria y registros.
- [x] Puntos verdes de Beszel y Dozzle en el panel.
- [ ] Beszel y Dozzle desde el iPhone fuera de casa.
- [ ] Tras un reinicio: las reglas vuelven, el agente arranca y el hub lo ve. En la sesión en persona.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| El script del agente se para en "Pega la KEY" | No se sabía dónde estaba | En **Añadir sistema**, el campo *Public Key*; el TOKEN, en su campo o en el comando que propone, tras `TOKEN` o `-t` |
| El script de Dozzle se niega: "Ya existe /srv/dozzle" | Un intento anterior falló al repetir la contraseña y dejó la carpeta (fallo del script, corregido) | Mirar que solo haya `.env` y `datos` vacía, y `sudo rm -r /srv/dozzle` |
| El agente sale en rojo | Token o KEY mal copiados | `journalctl -u beszel-agent` y revisar `/etc/beszel-agent.conf` |
| Dozzle no muestra registros | El proxy no deja una lectura que hace falta | Revisar las variables del proxy; nunca `POST=1` |

## 7. Cómo revertirlo

```bash
sudo systemctl disable --now beszel-agent.service
```
```bash
sudo rm /etc/systemd/system/beszel-agent.service /etc/beszel-agent.conf /usr/local/bin/beszel-agent
```
```bash
sudo docker compose -f /srv/beszel/compose.yaml down
```
```bash
sudo docker compose -f /srv/dozzle/compose.yaml down
```
```bash
sudo systemctl disable --now homelab-beszel-lan.service homelab-dozzle-lan.service
```
Borra las dos unidades de `/etc/systemd/system/`, quita `"tcp:8090"` y `"tcp:8080"` de la política y, si ya no se quieren, `/srv/beszel` (el historial) y `/srv/dozzle`.

## 8. Seguridad

**Mitiga:**
- **Nada nuevo tiene poder de root sobre Docker.** El agente no está en el grupo `docker`, y Dozzle pasa por un proxy que solo deja leer.
- El proxy no tiene salida ni puertos; solo le habla Dozzle.
- Usuarios propios, sin capacidades y con `no-new-privileges`. El agente no abre puertos.
- Las imágenes van fijadas por digest, y el binario del agente, comprobado con SHA-256.

**Queda:**
- **Dozzle ve los registros de todo**, y un registro puede contener datos sensibles. Va con usuario y contraseña.
- **Dozzle tuvo varios avisos graves en 2026.** Para algunos, el aviso no dice en qué versión se arreglaron. Por eso va siempre con autenticación y sin acciones ni consola. Hay que actualizarlo.
- **Beszel tuvo 4 avisos en 2026.** Uno no consta como arreglado (permisos de las alertas entre usuarios), pero solo afecta con varios usuarios. Aquí hay uno.
- La web de Beszel incluye `/_/`, la administración de PocketBase, protegida por la cuenta de administrador.
- HTTP en la LAN, hasta el dominio y Caddy.

## 9. Para la entrevista

**En 30 segundos:** "Para monitorizar no di a nada el socket de Docker. El agente de Beszel corre como servicio del sistema con su usuario, sin el grupo `docker`: descarté el `.deb` oficial porque lo añadía. Dozzle sí necesita Docker, así que va detrás de un proxy que solo deja hacer `GET`, en una red interna sin salida. Lo comprobé pidiendo un reinicio a través del proxy: `403`."

**Preguntas posibles:**
- *¿Por qué el agente fuera de Docker?* Para ver temperaturas, batería y discos sin `network_mode: host`, sin privilegios y sin socket.
- *¿Por qué no `:ro` en el socket?* Solo marca el archivo; la API sigue aceptando órdenes. Lo dice la documentación de Dozzle.
- *¿Por qué no monitorizar la NVIDIA?* Está reservada y en reposo. Consultarla a menudo la despierta y gasta batería.

## Fuentes

- Beszel: [instalación del hub](https://beszel.dev/guide/hub-installation), [del agente](https://beszel.dev/guide/agent-installation), [variables](https://beszel.dev/guide/environment-variables), [GPU](https://beszel.dev/guide/gpu), [discos](https://beszel.dev/guide/additional-disks), [seguridad](https://beszel.dev/guide/security), [versiones](https://github.com/henrygd/beszel/releases) y [avisos](https://github.com/henrygd/beszel/security/advisories). El `.deb` de la 0.21.2, inspeccionado con `dpkg-deb -e`.
- Dozzle: [inicio](https://dozzle.dev/guide/getting-started), [hosts remotos](https://dozzle.dev/guide/remote-hosts), [autenticación](https://dozzle.dev/guide/authentication/simple), [variables](https://dozzle.dev/guide/supported-env-vars), [versiones](https://github.com/amir20/dozzle/releases) y [avisos](https://github.com/amir20/dozzle/security/advisories).
- [docker-socket-proxy](https://github.com/Tecnativa/docker-socket-proxy).
