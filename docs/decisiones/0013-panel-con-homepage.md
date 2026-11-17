# ADR 0013: panel con Homepage, sin socket de Docker y con un nombre `home.arpa`

- **Estado:** aceptada (2026-11-17).
- **Decide:** el dueño.

## Contexto

Con cinco servicios, cada uno en su IP y su puerto, hace falta una página de inicio. El servidor tiene dos direcciones: la de casa y la de la tailnet. Fuera de casa solo vale la segunda, y en casa la VPN se desconecta sola ([ADR 0012](0012-fotos-con-immich.md)). Todavía no hay dominio ni HTTPS. El dueño ya hace páginas de inicio propias en gruvbox.

## Opciones

**Panel:**

| Opción | A favor | En contra |
|---|---|---|
| **Homepage** | Configuración en YAML, que va a Git; muchos widgets; ligero | Se edita a mano; un enlace por servicio |
| Homarr | Se edita desde la web | La configuración vive en su base de datos, fuera de Git |
| Una página propia | Control total | Hay que mantenerla; sin widgets |

**Enlaces que funcionen en casa y fuera:**

| Opción | A favor | En contra |
|---|---|---|
| **Un nombre con AdGuard, por cliente** | Un enlace sirve en los dos sitios | Toca el DNS de casa (dos reglas) |
| Solo las IP de casa | Lo más sencillo | Fuera de casa los enlaces no abren |
| Dos grupos de enlaces | Sin tocar el DNS | Todo duplicado |

**Socket de Docker:** con él, el panel ve el estado de cada contenedor; pero quien accede al socket es root en el servidor. Hay proxies que dejan solo lectura.

## Decisión

- Homepage `v2.8.0` como el usuario `homepage`, `cap_drop: ALL`, `no-new-privileges` y **sin socket de Docker**. Si más adelante hace falta, con un proxy de solo lectura.
- Nombre `homelab.home.arpa` (RFC 8375): dos reescrituras en AdGuard, una con `$client` para `<RED_LAN>` y otra para `100.64.0.0/10`. Amplía el [ADR 0010](0010-dns-de-casa-con-adguard.md).
- Puerto 80, desde la LAN y la tailnet (`DOCKER-USER` y `"tcp:80"` en la política). Amplía el [ADR 0009](0009-tailscale.md). Con el dominio, pasará detrás de Caddy.
- Estilo gruvbox con `custom.css`; el tiempo con Open-Meteo (sin clave), con las coordenadas del pueblo.

## Consecuencias

- **Comprobado:** el nombre da una sola IP en el PC; el contenedor corre como `homepage`; el dueño lo ve y usa desde el PC, y desde el iPhone fuera de casa por la tailnet.
- Para abrir desde fuera el panel de Syncthing, la política también lleva `"tcp:8384"`.
- Sin contraseña: lo ve cualquiera de la LAN o de la tailnet. Solo hay enlaces y uso de recursos.
- **Actualización (2026-11-17):** puntos de estado y datos de Immich, Jellyfin y AdGuard por una red de Docker, `panel`, que solo une las webs. Las claves, en archivos `400` fuera de Git (`HOMEPAGE_FILE_*`). Las de Jellyfin y AdGuard son de administrador; la de Syncthing no se usa, porque da control total.
- Al actualizar Homepage, revisar `custom.css` si cambian las variables de color.
- El servidor no resuelve `homelab.home.arpa` (no usa AdGuard): sus scripts siguen usando IP.
