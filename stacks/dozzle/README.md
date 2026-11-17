# Dozzle: registros de los contenedores

> **Versión usada:** Dozzle `v11.4.3` y `tecnativa/docker-socket-proxy` (`lts`), los dos fijados por digest. Docker Engine 29.9.3, Compose 5.7.1.

La guía completa, con el porqué, está en la [Fase 5, paso 1](../../docs/05-servicios/01-monitorizacion.md). Aquí, solo los archivos.

## Archivos

- [`compose.yaml`](compose.yaml): va en `/srv/dozzle/`. Dos contenedores:
  - `dozzle_proxy`: el único con el socket de Docker. Solo deja **leer** (`CONTAINERS`, `INFO` y `EVENTS`; `POST=0`). Vive en la red `interna`, sin salida y sin puertos publicados.
  - `dozzle`: usuario `dozzle`, sin capacidades, con usuario y contraseña, sin acciones, sin consola y sin analítica.
- [`.env.example`](.env.example): cópialo como `.env` con los UID y GID de `dozzle`.
- `datos/` (de `dozzle`, `700`): `users.yml` (usuario y hash bcrypt) y el secreto de las sesiones. **No va al repositorio.**

`users.yml` se crea con la propia imagen, pasando la contraseña por la entrada (no queda en el historial ni en la lista de procesos):
```bash
printf '%s\n' "$CONTRASEÑA" | sudo docker run -i --rm --network none <imagen> generate <usuario>
```

## Puertos

| Fuera (servidor) | Dentro | Para qué | Quién llega |
|---|---|---|---|
| 8080 TCP | 8080 | La web | `<RED_LAN>` y la tailnet por `tailscale0` |

El filtro está en `DOCKER-USER`, con `homelab-dozzle-lan.service`.

## Actualizar

Lee las [notas de la versión](https://github.com/amir20/dozzle/releases) y los [avisos de seguridad](https://github.com/amir20/dozzle/security/advisories): en 2026 tuvo varios graves. Cambia los digest en `compose.yaml` y:
```bash
sudo docker compose -f /srv/dozzle/compose.yaml pull
```
```bash
sudo docker compose -f /srv/dozzle/compose.yaml up -d
```
Comprueba después que el proxy sigue negando la escritura (`403`, apartado 5 de la guía). Cambia también las insignias del `README.md`.
