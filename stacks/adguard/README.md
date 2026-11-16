# AdGuard Home: DNS con bloqueo de anuncios

> **Versión usada:** imagen `adguard/adguardhome:v0.107.80`, Docker Engine 29.9.3, Compose 5.7.1.

La guía completa, con el porqué, está en la [Fase 4](../../docs/04-dns.md). Aquí, solo los archivos del stack.

## Archivos

- [`compose.yaml`](compose.yaml): va en `/srv/adguard/`. Usuario `adguard` sin privilegios, sin capacidades salvo `NET_BIND_SERVICE` y con `no-new-privileges`.
- [`.env.example`](.env.example): cópialo como `.env` junto a `compose.yaml`, con el UID y el GID de `adguard`.
- `work/` y `conf/` (propiedad de `adguard`, permisos `700`): datos y configuración. **No van al repositorio**: `conf/AdGuardHome.yaml` guarda el hash de la contraseña del panel.

## Puertos

| Fuera (servidor) | Dentro (contenedor) | Para qué | Quién llega |
|---|---|---|---|
| 53 UDP y TCP | 5353 | DNS | `<RED_LAN>` y la tailnet por `tailscale0` |
| 3000 TCP | 3000 | Panel web | Solo `<RED_LAN>` |

El filtro está en `DOCKER-USER`, con `homelab-adguard-lan.service` ([Fase 4, apartado 4.2](../../docs/04-dns.md#42-reglas-en-docker-user)).

## Actualizar

La imagen está fijada. Para cambiar de versión, consulta las [notas de la versión](https://github.com/AdGuardTeam/AdGuardHome/releases) y cambia la etiqueta en `compose.yaml`. Después:
```bash
sudo docker compose -f /srv/adguard/compose.yaml pull
```
```bash
sudo docker compose -f /srv/adguard/compose.yaml up -d
```
