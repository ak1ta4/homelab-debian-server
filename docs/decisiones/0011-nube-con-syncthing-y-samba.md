# ADR 0011: nube con Syncthing del PC al servidor y Samba

- **Estado:** aceptada (2026-11-16).
- **Decide:** el dueño.

## Contexto

El dueño quiere tener en el servidor sus documentos, imágenes y vídeos del PC (unos 77 GB), verlos desde el iPhone y el iPad, y subir cosas a mano desde el móvil. Nada puede exponerse a internet. Todavía no hay HTTPS: el dominio y Caddy están aplazados (paso 5.0). El iPhone no se sincroniza: las subidas desde él son siempre manuales.

## Opciones

**Cómo llegan los archivos del PC al servidor:**

| Opción | A favor | En contra |
|---|---|---|
| Nextcloud | Todo en uno: sincronización, web y apps | Pesado: base de datos, PHP y web; mucha superficie para copiar carpetas |
| **Syncthing** | Pequeño, cifrado entre equipos, sin servidor central; versiones en el receptor | Solo sincroniza: para ver los archivos desde el móvil hace falta otra pieza |
| Copias programadas (robocopy o rsync) | Simple | Sin versiones ni detección de cambios en tiempo real; hay que abrir el PC o el servidor |

**Sentido de la sincronización:**

| Opción | A favor | En contra |
|---|---|---|
| **PC → servidor** (enviar / recibir solamente) | El PC manda; un error en el servidor no toca el PC | Lo que se cambia en el servidor no vuelve al PC |
| En los dos sentidos | Se puede editar en cualquier lado | Un borrado o un ransomware en el servidor también llegaría al PC |

**Cómo se ven desde el móvil:**

| Opción | A favor | En contra |
|---|---|---|
| File Browser | Web cómoda | Archivado el 2026-09-01; su continuación (Quantum) tuvo varios avisos de seguridad graves en octubre y noviembre de 2026; sin HTTPS todavía |
| **Samba** (SMB3) | La app Archivos del iPhone y el Explorador lo usan sin instalar nada; paquete de Debian con sus actualizaciones de seguridad | Un servicio más en el host; el iPhone necesita `vfs_fruit` |

## Decisión

- Syncthing en Docker en el servidor y el zip oficial en Windows, que arranca al iniciar sesión.
- El PC **envía solamente** y el servidor **recibe solamente**, con versiones escalonadas de 365 días.
- Sin relays, sin descubrimiento y sin NAT traversal. El PC llama al servidor por la LAN y no abre puertos.
- Samba en el host, solo SMB3, con dos carpetas: `pc` (el espejo) en **solo lectura** y `subidas` (fuera de Syncthing) con escritura, para un único usuario sin shell.
- Samba desde la LAN y la tailnet (`"tcp:445"` en la política de Tailscale). Amplía el [ADR 0009](0009-tailscale.md).

## Consecuencias

- **Comprobado:**
  - Las tres carpetas, al día (76,8 GiB) en el disco de datos.
  - Windows con SMB 3.1.1 cifrado.
  - El iPhone sube y descarga en casa y por la tailnet.
- No es una copia de seguridad: un borrado o un ransomware en el PC se sincroniza. Las versiones lo cubren en parte; las copias fuera de este disco, en la Fase 6.
- Lo que se sube desde el móvil va a `subidas`, no al PC. Para fotos, la herramienta prevista es Immich (paso 5.6).
- Si el servidor cambia de IP, el PC deja de encontrarlo: depende de la reserva DHCP.
- Cuando haya HTTPS, se puede reconsiderar una web de archivos.
