# Immich: fotos

> **Versión usada:** Immich `v3.4.1`, Valkey 9 y Postgres 14 con VectorChord 0.4.3 (las imágenes de la versión, con su digest). Docker Engine 29.9.3, Compose 5.7.1.

La guía completa, con el porqué, está en la [Fase 5, paso 6](../../docs/05-servicios/06-fotos.md). Aquí, solo los archivos del stack.

## Archivos

- [`compose.yaml`](compose.yaml): va en `/srv/immich/`. Los cuatro contenedores con el usuario `immich`, sin capacidades y con `no-new-privileges`. Parte del `docker-compose.yml` de la versión y de `docker-compose.rootless.yml`.
- [`.env.example`](.env.example): cópialo como `.env` (permisos `600`) con los UID y GID, y una contraseña aleatoria para la base de datos.
- `postgres/`, `redis/` y `ml-model-cache/` (de `immich`, permisos `700`): base de datos, caché y modelos de IA. **No van al repositorio.**

## Rutas

| En el servidor | En el contenedor | Qué |
|---|---|---|
| `/srv/datos/immich` | `/data` | Lo subido a Immich, miniaturas, vídeos convertidos y volcados de la base de datos (`backups/`) |
| `/srv/immich/postgres` | `/var/lib/postgresql/data` | Base de datos, en el NVMe: Immich la quiere en disco local y rápido |
| `/srv/datos/archivos/pc/imagenes` | `/mnt/pc-imagenes` (`:ro`) | Imágenes del PC, como biblioteca externa en solo lectura |

## Puertos

| Fuera (servidor) | Dentro | Para qué | Quién llega |
|---|---|---|---|
| 2283 TCP | 2283 | Web y app | `<RED_LAN>` y la tailnet por `tailscale0` |

El filtro está en `DOCKER-USER`, con `homelab-immich-lan.service` ([guía, apartado 4.2](../../docs/05-servicios/06-fotos.md#42-reglas-en-docker-user)). Postgres y Valkey no publican nada.

## Actualizar

La versión está fijada en `.env` y las imágenes de Valkey y Postgres, por digest. **Antes de actualizar**, lee las [notas de la versión](https://github.com/immich-app/immich/releases) y compara el `docker-compose.yml` nuevo con este: a veces cambian las imágenes de la base de datos. No se puede volver a una versión anterior. Después:
```bash
sudo docker compose -f /srv/immich/compose.yaml pull
```
```bash
sudo docker compose -f /srv/immich/compose.yaml up -d
```
La app del móvil tiene que ser de la misma versión mayor (v3) que el servidor.
