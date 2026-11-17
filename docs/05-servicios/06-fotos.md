# Fase 5, paso 6: fotos con Immich

> **Versiones usadas:** Debian 13.7, Docker Engine 29.9.3, Compose 5.7.1, Immich `v3.4.1` (servidor, IA, Valkey 9 y Postgres 14 con VectorChord 0.4.3), app de Immich para iOS.

## 1. Qué vamos a hacer y por qué

**En sencillo:** una galería de fotos tipo Google Fotos, pero en el servidor de casa. Ordena por fecha y lugar, reconoce caras y permite buscar por lo que sale en la foto ("playa", "perro"). Muestra las fotos del PC sin copiarlas otra vez y deja subir fotos del iPhone a mano.

**Técnicamente:**
- Immich en Docker, con los cuatro contenedores (servidor, IA, Valkey y Postgres) **sin root** y sin capacidades.
- La IA funciona en la CPU, sin la NVIDIA.
- Imágenes del PC como **biblioteca externa en solo lectura**.
- Web y app por HTTP en el 2283, desde la LAN y la tailnet, hasta que haya dominio y HTTPS.

Decisión y alternativas: [ADR 0012](../decisiones/0012-fotos-con-immich.md).

## 2. Conceptos previos

- **Biblioteca externa:** una carpeta que Immich lee pero no gestiona. Con `:ro`, no puede borrar ni modificar nada en ella. Lo que se borra en el origen desaparece de Immich en el siguiente escaneo.
- **Biblioteca propia:** lo que se sube a Immich (desde la app o la web). Immich lo guarda en su carpeta y sí puede borrarlo: va a la papelera y se elimina del todo a los 30 días.
- **Trabajos:** tareas en segundo plano (metadatos, miniaturas, caras, búsqueda inteligente, OCR). La primera vez procesan todo; después, solo lo nuevo.
- **Primer usuario:** quien entra primero en la web se convierte en administrador. Hay que crearlo justo después de arrancar.

## 3. Requisitos y estado inicial

- Docker ([Fase 1, paso 8](../01-base-y-endurecimiento/08-docker.md)), disco de datos y Tailscale.
- Nube del [paso 3](03-nube.md): las imágenes del PC en `/srv/datos/archivos/pc/imagenes`.
- Immich pide 6 GB de RAM como mínimo (8 recomendados) y 2 núcleos (4 recomendados). El servidor tiene 14 GB y 12 hilos. La IA necesita una CPU `x86-64-v2` o superior:
  ```bash
  /lib64/ld-linux-x86-64.so.2 --help | grep x86-64-v2
  ```
  ✅ `x86-64-v2 (supported, searched)`.
- Base de datos en disco local y rápido: `/` es ext4 en el NVMe.

## 4. Pasos

En el servidor, con `<USUARIO_ADMIN>`. Se hizo con un script revisado antes de ejecutarlo; estos son sus pasos.

### 4.1 Usuario, carpetas y Compose

```bash
sudo adduser --system --group --no-create-home --home /nonexistent immich
```
```bash
sudo install -d -m 755 /srv/immich
```
```bash
sudo install -d -o immich -g immich -m 700 /srv/immich/postgres /srv/immich/redis /srv/immich/ml-model-cache
```
```bash
sudo install -d -o immich -g immich -m 750 /srv/datos/immich
```
Prueba: con el grupo `syncthing`, `immich` lee las imágenes del PC y no puede escribir:
```bash
sudo -u immich -g syncthing touch /srv/datos/archivos/pc/imagenes/.prueba
```
✅ `Permission denied`.

Copia [`compose.yaml`](../../stacks/immich/compose.yaml) en `/srv/immich/` y crea `.env` (permisos `600`) a partir de [`.env.example`](../../stacks/immich/.env.example), con una contraseña aleatoria:
```bash
openssl rand -hex 24
```

Diferencias con el ejemplo oficial sin root:
- El usuario es `immich`, no `1000:1000`.
- `cap_drop: ALL`, en vez de solo `NET_RAW`.
- `group_add` con el GID de `syncthing`: Docker no aplica al contenedor los grupos secundarios del usuario, así que hay que dárselo.
- La versión fijada (`v3.4.1`), no `v3`.

### 4.2 Reglas en `DOCKER-USER`

`/etc/systemd/system/homelab-immich-lan.service`, con el mismo esquema que el de Syncthing ([paso 3, 4.2](03-nube.md#42-reglas-en-docker-user)):
```ini
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctstate DNAT --ctorigdstport 2283 --ctdir ORIGINAL ! -s <RED_LAN> -j DROP
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctstate DNAT --ctorigdstport 2283 --ctdir ORIGINAL -i tailscale0 -s 100.64.0.0/10 -j RETURN
```
Con su `[Unit]` (`After`, `Requires` y `PartOf=docker.service`), un `ExecStop=-… -D` por regla y `WantedBy=docker.service`.

```bash
sudo systemctl daemon-reload
```
```bash
sudo systemctl enable --now homelab-immich-lan.service
```
En la política de Tailscale, `"tcp:2283"` en la lista `ip` ([Fase 3, 4.1](../03-acceso-remoto.md#41-política-de-acceso)).

### 4.3 Arrancar y crear el administrador

```bash
sudo docker compose -f /srv/immich/compose.yaml up -d
```
```bash
curl -s http://127.0.0.1:2283/api/server/ping
```
✅ `{"res":"pong"}`. Para ver el usuario de cada contenedor (`docker top` necesita la columna PID):
```bash
sudo docker top immich_postgres -o pid,user
```
✅ `immich` en todas las líneas, igual en los otros tres.

**Enseguida**, desde el PC, `http://<IP_SERVIDOR>:2283` → **Comenzar**: correo, contraseña (en el gestor) y nombre. El asistente, con lo que trae.

### 4.4 Fotos del PC

En la web, **Administración → Bibliotecas externas → Crear biblioteca**:
1. **Propietario:** tu usuario. No se puede cambiar después.
2. **Renombrar:** `PC`.
3. **Rutas de importación:** `/mnt/pc-imagenes`, la ruta dentro del contenedor.
4. **Patrones de exclusión:** `**/Manga/**`. No son fotos y la IA buscaría caras en ellas.
5. **Escanear.**

En **Ajustes → Biblioteca externa**, el escaneo periódico (diario) viene activado y se deja así; la vigilancia `[EXPERIMENTAL]`, apagada.

✅ **Colas de trabajo** con unos 2.200 recursos en marcha.

### 4.5 App del iPhone

1. **Servidor:** en casa, `http://<IP_SERVIDOR>:2283`, y entrar con la cuenta.
2. **Ajustes de la app → Red → Cambio automático de URL:**
   - **Red local:** **Usar la conexión actual** (el nombre del wifi de casa y `http://<IP_SERVIDOR>:2283`). iOS pide la ubicación para leer el nombre del wifi: **Al usar la app** basta.
   - **Red externa:** `http://<IP_TAILSCALE_SERVIDOR>:2283`. En casa sale con ❗ porque Tailscale está desconectado; es normal.
3. **Sin copia de seguridad automática:** las fotos se suben a mano (seleccionar → **Subir**).
4. En la app de Tailscale, **VPN On Demand**: desconectar en el wifi de casa y conectar en el resto.

## 5. Verificación

- [x] `{"res":"pong"}` y los cuatro contenedores con el usuario `immich`.
- [x] Reglas del 2283 arriba del todo en `DOCKER-USER`.
- [x] `immich` lee las imágenes del PC y no puede escribirlas.
- [x] Biblioteca externa escaneada, sin la carpeta de Manga; trabajos de IA en marcha.
- [x] App del iPhone en casa, sin VPN, por `<IP_SERVIDOR>`.
- [x] App del iPhone con datos: Tailscale se conecta solo y entra por la tailnet.
- [ ] Trabajos terminados y carga del servidor normal.
- [ ] Tras un reinicio: las reglas vuelven y Immich arranca. En la sesión en persona.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `docker top … -o user`: `Couldn't find PID field` | `docker top` necesita la columna PID | `-o pid,user` |
| La app: "Server is not reachable" en casa | Dirección de la tailnet con Tailscale apagado | `http://<IP_SERVIDOR>:2283` en casa, o el cambio automático de URL |
| La app pide la ubicación | iOS la exige para leer el nombre del wifi | **Al usar la app**, no "Siempre" |
| Fuera de casa no entra con `<IP_SERVIDOR>` | Esa IP solo existe en la red de casa | Tailscale y `<IP_TAILSCALE_SERVIDOR>` |
| No se puede borrar una foto del PC desde Immich (o vuelve) | La biblioteca externa es de solo lectura | Borrarla en el PC; Immich la retira en el siguiente escaneo |
| Servidor al 100 % durante horas | Primer procesado de la IA en CPU | Esperar, o bajar la concurrencia en **Ajustes → Trabajos** |
| En Ajustes no aparece "Crear biblioteca" | Ahí están las opciones generales | **Bibliotecas externas**, en el menú de la izquierda |

## 7. Cómo revertirlo

```bash
sudo docker compose -f /srv/immich/compose.yaml down
```
```bash
sudo systemctl disable --now homelab-immich-lan.service
```
```bash
sudo rm /etc/systemd/system/homelab-immich-lan.service
```
Quita `"tcp:2283"` de la política de Tailscale. `/srv/immich` y `/srv/datos/immich` no se borran solos: en `/srv/datos/immich` están las fotos subidas desde el móvil. Las imágenes del PC no se tocan en ningún caso.

## 8. Seguridad

**Mitiga:**
- Fotos en casa, sin servicios de terceros. La IA es local.
- Contenedores comprometidos: usuario sin privilegios, sin capacidades y `no-new-privileges`. Postgres y Valkey no publican puertos.
- Las fotos del PC no se pueden borrar ni modificar desde Immich (`:ro` y permisos de solo lectura).
- Nada sale a internet; la web solo llega desde la LAN y la tailnet.

**Queda:**
- **HTTP en la LAN:** la contraseña y la sesión viajan sin cifrar por el wifi de casa. Por la tailnet van cifradas. Se arregla con el dominio y Caddy (paso 5.0).
- La app guarda el nombre del wifi de casa y pide la ubicación mientras se usa.
- Lo subido desde el iPhone vive en un solo disco hasta la Fase 6. **No borrar fotos del móvil** hasta tener copias.
- El volcado diario de la base de datos (`/srv/datos/immich/backups`) solo tiene metadatos, no fotos.
- La primera vez, la IA deja la CPU al 100 % durante horas (carga de 12 a 26 con 12 hilos).

## 9. Para la entrevista

**En 30 segundos:** "Instalé Immich con sus cuatro contenedores sin root y sin capacidades, partiendo del ejemplo rootless oficial y endureciéndolo más. La base de datos va en el NVMe y las fotos en el disco de datos. Las fotos del PC entran como biblioteca externa de solo lectura: el contenedor tiene el grupo de Syncthing con `group_add`, y el montaje es `:ro`. En el iPhone, la app cambia de dirección según el wifi, y Tailscale se activa solo fuera de casa."

**Preguntas posibles:**
- *¿Por qué `group_add`?* Docker arranca el proceso con el UID y el GID indicados, sin los grupos secundarios del usuario en el host.
- *¿Por qué la base de datos en otro disco?* Immich la quiere en disco local y rápido; el NVMe lo es, y las fotos van al disco grande.
- *¿Qué pasa si borras desde Immich una foto del PC?* No se puede: el montaje es de solo lectura. Se borra en el PC y Syncthing e Immich lo propagan.

## Fuentes

- [Immich: requisitos](https://docs.immich.app/install/requirements), [Docker Compose](https://docs.immich.app/install/docker-compose), [FAQ](https://docs.immich.app/FAQ) (sin root), [bibliotecas externas](https://docs.immich.app/features/libraries) y [copias de seguridad](https://docs.immich.app/administration/backup-and-restore).
- [`docker-compose.rootless.yml`](https://github.com/immich-app/immich/blob/main/docker/docker-compose.rootless.yml) y las [notas de la versión](https://github.com/immich-app/immich/releases).
