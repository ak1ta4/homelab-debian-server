# Homepage: panel de inicio

> **Versión usada:** Homepage `v2.8.0`. Docker Engine 29.9.3, Compose 5.7.1.

La guía completa, con el porqué, está en la [Fase 5, paso 1](../../docs/05-servicios/01-panel.md). Aquí, solo los archivos del stack.

## Archivos

- [`compose.yaml`](compose.yaml): va en `/srv/homepage/`. Un contenedor con el usuario `homepage`, sin capacidades, con `no-new-privileges` y **sin el socket de Docker**.
- [`.env.example`](.env.example): cópialo como `.env` con los UID y GID de `homepage` y los nombres permitidos.
- [`config/`](config/): va en `/srv/homepage/config/` (de `homepage`, carpeta `750` y archivos `640`):
  - `settings.yaml`: título, tema oscuro, grupos y sus iconos;
  - `custom.css`: la paleta gruvbox, cambiando las variables de color del tema `stone`;
  - `services.yaml`: los enlaces a cada servicio, con `homelab.home.arpa`;
  - `widgets.yaml`: CPU, RAM, discos, el tiempo (Open-Meteo, sin clave) y la hora;
  - `bookmarks.yaml`: los marcadores.

Homepage relee los YAML y `custom.css` al recargar la página. Al arrancar crea los que falten (`docker.yaml`, `kubernetes.yaml`…) vacíos; no se usan.

## Rutas

| En el servidor | En el contenedor | Qué |
|---|---|---|
| `/srv/homepage/config` | `/app/config` | Configuración |
| `/srv/datos` | `/mnt/datos` (`:ro`) | Solo para mostrar el espacio libre del disco de datos |

## Puertos

| Fuera (servidor) | Dentro | Para qué | Quién llega |
|---|---|---|---|
| 80 TCP | 3000 | El panel | `<RED_LAN>` y la tailnet por `tailscale0` |

El filtro está en `DOCKER-USER`, con `homelab-homepage-lan.service` ([guía, apartado 4.3](../../docs/05-servicios/01-panel.md#43-reglas-en-docker-user)).

## Actualizar

La versión está fijada en `compose.yaml`. Antes, lee las [notas de la versión](https://github.com/gethomepage/homepage/releases): a veces cambian los widgets. Si cambian los nombres de las variables de color (`src/styles/theme.css`), hay que revisar `custom.css`. Después:
```bash
sudo docker compose -f /srv/homepage/compose.yaml pull
```
```bash
sudo docker compose -f /srv/homepage/compose.yaml up -d
```
Y cambia la versión en la insignia del `README.md` del repositorio.
