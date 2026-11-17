# ADR 0012: fotos con Immich, sin root y con las fotos del PC en solo lectura

- **Estado:** aceptada (2026-11-17).
- **Decide:** el dueño.

## Contexto

El dueño quiere una galería con las fotos del PC (ya en el servidor por Syncthing, [ADR 0011](0011-nube-con-syncthing-y-samba.md)) y poder subir fotos del iPhone a mano, sin copia automática. Quiere reconocimiento de caras y búsqueda. Todavía no hay HTTPS. El servidor es un portátil con 14 GB de RAM y una NVIDIA de 4 GB reservada (no se tocan sus drivers).

## Opciones

**Cómo entran las fotos del PC:**

| Opción | A favor | En contra |
|---|---|---|
| Subirlas a Immich | Immich las gestiona del todo | Duplica 40 GB; dos copias que pueden divergir |
| **Biblioteca externa en solo lectura** | Sin duplicar; Immich no puede borrarlas | No se borran ni editan desde Immich; se hace en el PC |
| Biblioteca externa con escritura | Se podrían borrar desde Immich | Borraría en el espejo de Syncthing, no en el PC, y Syncthing lo marcaría como cambio local |

**IA (caras y búsqueda):**

| Opción | A favor | En contra |
|---|---|---|
| **En la CPU** | Sin tocar la NVIDIA; local | Primer procesado de horas, con la CPU al 100 % |
| En la NVIDIA | Más rápida | Exige el runtime de NVIDIA en Docker; cambio de drivers y gráficos, aplazado |
| Sin IA | Ligero | Sin caras ni búsqueda por contenido |

**Usuario de los contenedores:** el compose oficial corre como root. Immich documenta un ejemplo sin root (`docker-compose.rootless.yml`).

## Decisión

- Immich `v3.4.1` en Docker, con los cuatro contenedores como el usuario `immich`, `cap_drop: ALL` y `no-new-privileges`.
- Base de datos en el NVMe (`/srv/immich/postgres`) y lo subido en el disco de datos (`/srv/datos/immich`).
- Imágenes del PC como biblioteca externa con montaje `:ro`; el contenedor lee gracias al grupo `syncthing` (`group_add`). Se excluye la carpeta de Manga.
- IA en la CPU, con la concurrencia por defecto.
- Web en el 2283 por HTTP, desde la LAN y la tailnet (`DOCKER-USER` y `"tcp:2283"` en la política). Amplía el [ADR 0009](0009-tailscale.md).
- En el iPhone: subida manual, cambio automático de URL según el wifi y Tailscale con VPN On Demand fuera de casa.

## Consecuencias

- **Comprobado:**
  - Los cuatro contenedores corren como `immich`.
  - La biblioteca del PC está escaneada.
  - La app entra en casa sin VPN y fuera por la tailnet.
- Hasta tener HTTPS, la contraseña viaja sin cifrar por el wifi de casa.
- Lo subido desde el iPhone solo está en un disco hasta la Fase 6: no se borra del móvil antes.
- Cada actualización exige revisar el `docker-compose.yml` nuevo (imágenes de la base de datos) y las notas; no se puede volver atrás.
- Se puede pasar la IA a la NVIDIA más adelante, si el dueño lo pide.
