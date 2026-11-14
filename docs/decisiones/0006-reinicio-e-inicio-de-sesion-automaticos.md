# ADR 0006: reinicio automático tras actualizar e inicio de sesión automático con bloqueo

- **Estado:** aceptada (2026-11-14).
- **Decide:** el dueño.

## Contexto

Las actualizaciones de seguridad se aplican solas con `unattended-upgrades` (paso 6). Algunas, como un núcleo nuevo, solo surten efecto tras reiniciar. El escritorio remoto (Sunshine) es un servicio de usuario: sin una sesión iniciada no arranca, y GDM no tenía inicio de sesión automático. Un reinicio sin nadie delante dejaba el equipo sin Moonlight hasta entrar en persona. El disco no está cifrado ([ADR 0001](0001-sin-cifrado-de-disco.md)).

## Opciones

| Opción | A favor | En contra |
|---|---|---|
| Sin reinicio automático; reinicia el dueño | Ningún reinicio por sorpresa | El núcleo nuevo no se usa hasta que alguien se acuerda |
| **Reinicio automático sobre las 6:00, solo si hace falta** | El sistema corre siempre con los parches | Cada reinicio cierra la sesión de tmux del agente y depende de que el driver de NVIDIA se recompile bien |
| Sin inicio de sesión automático | Nadie entra sin contraseña | Tras reiniciar no hay Moonlight |
| Inicio de sesión automático sin más | Moonlight disponible tras reiniciar | Con el portátil delante, se entra directamente en el escritorio |
| **Inicio de sesión automático con bloqueo inmediato** | Sunshine arranca y el escritorio sigue pidiendo contraseña | Unos segundos de escritorio sin bloquear en cada arranque; el llavero de GNOME queda cerrado hasta abrirlo |

## Decisión

- `unattended-upgrades` con `Automatic-Reboot "true"`, `Automatic-Reboot-WithUsers "true"` y `Automatic-Reboot-Time "now"`. Lo lanza `apt-daily-upgrade.timer` entre las 6:00 y las 7:00, y reinicia solo si existe `/run/reboot-required`.
- GDM con `AutomaticLoginEnable` para `<USUARIO_ADMIN>`.
- Un programa de inicio de GNOME (`~/.config/autostart/bloquear-al-entrar.desktop`) ejecuta `loginctl lock-session` a los 3 segundos de entrar. Comprobado a mano que ese comando bloquea la sesión.

## Consecuencias

- Tras un reinicio, por Moonlight se ve la pantalla de bloqueo y se entra con la contraseña. **Pendiente de comprobar** en el primer reinicio.
- **Riesgo aceptado:** quien tenga el portátil delante durante el arranque dispone de unos segundos de escritorio sin bloquear. Con el disco sin cifrar, el acceso físico ya da acceso a los datos, así que el riesgo añadido es pequeño.
- El llavero de GNOME no se abre al entrar: alguna aplicación puede pedir la contraseña una vez.
- Cada reinicio automático cierra la sesión de tmux del agente: hay que relanzarla a mano hasta que arranque sola (Fase 7).
- Si el driver de NVIDIA no se recompila con un núcleo nuevo, la salida HDMI deja de funcionar. Sunshine captura y codifica en la AMD, así que Moonlight no depende de ello.
- Sin correo, nadie se entera de un reinicio ni de un fallo hasta mirar el registro (`/var/log/unattended-upgrades/`). Los avisos llegan en la Fase 5.
