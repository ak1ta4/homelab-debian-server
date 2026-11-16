# Syncthing: copia del PC en el servidor

> **Versión usada:** imagen `syncthing/syncthing:2.1.7` en el servidor y el zip oficial `2.1.7` en Windows. Docker Engine 29.9.3, Compose 5.7.1.

La guía completa, con el porqué, está en la [Fase 5, paso 3](../../docs/05-servicios/03-nube.md). Aquí, solo los archivos del stack.

## Archivos

- [`compose.yaml`](compose.yaml): va en `/srv/syncthing/`. Usuario `syncthing` sin privilegios, sin capacidades y con `no-new-privileges`. La imagen, arrancada como no-root, ejecuta Syncthing directamente, sin `su-exec` ni `setcap`.
- [`.env.example`](.env.example): cópialo como `.env` junto a `compose.yaml`, con el UID y el GID de `syncthing`.
- `var/` (de `syncthing`, permisos `700`): configuración, claves del dispositivo y base de datos. **No va al repositorio**: guarda la clave privada del dispositivo y el hash de la contraseña del panel.
- [`documentos.stignore`](documentos.stignore): los patrones que se pegan en el PC, en la carpeta Documentos (pestaña **Patrones a ignorar**). Imágenes y Vídeos solo ignoran `desktop.ini` y `Thumbs.db`.

## Puertos

| Fuera (servidor) | Dentro (contenedor) | Para qué | Quién llega |
|---|---|---|---|
| 22000 TCP y UDP | 22000 | Sincronización | `<RED_LAN>` y la tailnet por `tailscale0` |
| 8384 TCP | 8384 | Panel web (HTTPS) | Solo `<IP_PC>` y la tailnet |

El filtro está en `DOCKER-USER`, con `homelab-syncthing-lan.service` ([guía, apartado 4.2](../../docs/05-servicios/03-nube.md#42-reglas-en-docker-user)). El puerto 21027 (descubrimiento local) no se publica: el descubrimiento está apagado.

## Carpetas

| ID | En el PC (Enviar solamente) | En el servidor (Recibir solamente) |
|---|---|---|
| `pc-documentos` | `C:\Users\<USUARIO_WINDOWS>\Documents` | `/datos/documentos` |
| `pc-imagenes` | `C:\Users\<USUARIO_WINDOWS>\Pictures` | `/datos/imagenes` |
| `pc-videos` | `C:\Users\<USUARIO_WINDOWS>\Videos` | `/datos/videos` |

`/datos` es `/srv/datos/archivos/pc` en el disco de datos. En el servidor, cada carpeta guarda versiones escalonadas durante 365 días.

## Actualizar

La imagen está fijada. Para cambiar de versión, consulta las [notas de la versión](https://github.com/syncthing/syncthing/releases) y cambia la etiqueta en `compose.yaml`. Después:
```bash
sudo docker compose -f /srv/syncthing/compose.yaml pull
```
```bash
sudo docker compose -f /srv/syncthing/compose.yaml up -d
```
El PC se actualiza solo: es lo que trae el zip oficial de fábrica. Mientras las dos sean 2.x, se entienden.
