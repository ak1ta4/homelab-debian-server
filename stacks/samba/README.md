# Samba: carpeta de red

> **Versión usada:** Samba 4.22.11 de Debian 13.7, instalado en el host (no en Docker).

La guía completa, con el porqué, está en la [Fase 5, paso 3](../../docs/05-servicios/03-nube.md#44-samba). Aquí, solo la configuración.

## Archivos

- [`smb.conf`](smb.conf): va en `/etc/samba/smb.conf`. Sustituye los marcadores (ver el [glosario](../../docs/glosario.md)) y valida con `testparm -s`.

## Carpetas compartidas

| Nombre | Ruta | Usuario `<USUARIO_WINDOWS>` |
|---|---|---|
| `pc` | `/srv/datos/archivos/pc` | Lee y descarga. **No escribe**: es el espejo de Syncthing |
| `subidas` | `/srv/datos/archivos/subidas` | Lee, descarga y sube. Fuera de Syncthing |

El usuario de Samba es del sistema, sin shell ni carpeta personal, y está en el grupo `syncthing` (que solo puede leer la copia). Su contraseña de Samba es propia y se pone con `smbpasswd -a`.

## Puertos

| Puerto | Quién llega |
|---|---|
| 445 TCP | `<RED_LAN>` y la tailnet por `tailscale0` (UFW), y `"tcp:445"` en la política de Tailscale |

Sin NetBIOS: `nmbd` está enmascarado y no se abren el 137, el 138 ni el 139.

## Actualizar

Llega con las actualizaciones de seguridad de Debian. Tras cambiar `smb.conf`:
```bash
sudo testparm -s
```
```bash
sudo systemctl reload smbd
```
