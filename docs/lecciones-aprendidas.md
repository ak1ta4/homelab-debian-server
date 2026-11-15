# Lecciones aprendidas

Errores reales, con su causa y su solución. Se escribe mientras se trabaja, no al final.

## Pantalla negra en Moonlight con NVIDIA

**Cuándo:** antes del homelab, en un intento de usar la RTX 3050 y una tele por HDMI.

- **Síntoma:** al conectar con Moonlight, pantalla negra con el cursor visible; en los registros de Sunshine, errores al convertir la imagen (`Could not convert image`).
- **Causa real:** para que funcionase la tele (conectada a la NVIDIA) se activó `nvidia-drm modeset=1`. Desde entonces, Sunshine **codificaba con Vulkan en la NVIDIA** mientras **capturaba por KMS la pantalla del portátil, que va en la AMD**. Importar la imagen de una GPU a otra (DMA-BUF) fallaba. Antes del cambio, captura y codificación iban en la AMD y funcionaba.
- **Lo que no era:**
  - **NVENC:** falló por otro motivo. Esta versión de Sunshine exige CUDA 13.1, y el driver 550 llega a CUDA 12.4.
  - **`capture = portal`:** parecía arreglarlo, pero al lanzar el escritorio se quedaba esperando un diálogo de GNOME para permitir compartir pantalla y **bloqueaba el puerto HTTPS 47984**. Moonlight daba *timeout* al emparejar, y cada intento creaba un emparejamiento duplicado.
- **Solución:** capturar y codificar en la misma GPU, la AMD:
  ```
  capture = kms
  encoder = vaapi
  adapter_name = /dev/dri/renderD128
  ```
- **Otros tropiezos del mismo día:**
  - Sunshine no volvía tras reiniciar porque el servicio estaba deshabilitado.
  - Emparejamientos repetidos con el mismo certificado hacían que Sunshine rechazara al cliente (401): hay que dejar un solo registro y emparejar **una** vez.
  - Escalar pantallas con transformaciones de `xrandr` a espaldas de GNOME (más una extensión de panel) **rompió el escritorio**. Se revirtió, y la escala por monitor se hizo con la configuración de Mutter en Wayland.
- **Lección:**
  - No dar una causa por buena sin comprobar el flujo completo.
  - En un portátil híbrido, captura y codificación en la misma GPU.
  - Nada de cambios de pantalla por detrás del compositor.

## El router tenía su propio SSH abierto a internet

- **Síntoma:** reenvíos, DMZ y UPnP vacíos, y aun así un antivirus avisaba de un puerto 22 abierto en el router.
- **Causa:** un servicio del propio router (Dropbear), escuchando también en la interfaz de internet. La revisión inicial solo había probado desde fuera los puertos **del servidor**.
- **Solución:** reenvío del TCP 22 de internet a una IP sin uso ([ADR 0004](decisiones/0004-tapar-ssh-del-router.md)), verificado desde datos móviles.
- **Lección:** probar desde fuera también los servicios del router, y comparar huellas para saber **quién** responde.

## Una prueba "desde fuera" que no demostraba nada

- **Síntoma:** la app de SSH del móvil "pedía usuario", y eso se tomó por respuesta del router.
- **Causa:** la app pide el usuario **antes** de conectar si la conexión no lo tiene guardado.
- **Solución:** guardar el usuario en la conexión y mirar solo si pide **contraseña**, enseña una **huella** o da error.
- **Lección:** diseñar cada prueba para que sus dos resultados posibles signifiquen cosas distintas.

## La MAC "del PC" era de otro aparato

- **Síntoma:** la MAC que el servidor veía en la IP atribuida al PC tenía formato de MAC aleatoria.
- **Causa:** el PC había cambiado de IP desde la auditoría. Esa IP la tenía otro dispositivo, con MAC privada.
- **Solución:** sacar la MAC con `ipconfig /all` en el propio PC y reservar su IP actual.
- **Lección:** una IP prestada no identifica a un equipo. Por eso existen las reservas DHCP.

## Fail2ban volvía a bloquear justo después de desbloquear

- **Síntoma:** tras `unbanip`, un solo intento desde el móvil lo bloqueaba otra vez.
- **Causa:** con `mode = aggressive` y un usuario inexistente, **cada conexión** genera varias líneas que cuentan como fallo, y se llega a 5 en un intento.
- **Solución:** comprobar el desbloqueo en el servidor (`nft list set inet f2b-table addr-set-sshd`), no reintentando.
- **Lección:** para probar un sistema que reacciona a fallos, comprobar el estado desde el propio sistema, sin provocar más fallos.

## gitleaks-action y la primera subida

- **Síntoma:** el primer workflow de gitleaks en GitHub terminó en rojo.
- **Causa:** en un `push`, la acción revisa `primer_commit^..último_commit`. En la primera subida de un repositorio, el primer commit no tiene padre, git falla y **no se revisa nada**. En local, gitleaks 8.30.1 llegó a terminar con código 0 en esa situación: habría pasado por bueno.
- **Solución:** workflow propio que revisa todo el historial con el binario fijado por checksum y **exige haber revisado al menos todos los commits**. Probado: limpio pasa, token falso falla, fallo de git falla.
- **Lección:** un control de seguridad tiene que fallar cerrado. Comprobar que de verdad revisó algo, no solo que no se quejó.

## El llavero de GNOME guardó en memoria la clave de despliegue

- **Síntoma:** `ssh -T` a GitHub entraba sin pedir la frase de paso.
- **Causa:** la sesión gráfica usa el agente SSH del llavero de GNOME (`SSH_AUTH_SOCK=…/keyring/ssh`). La primera vez que se usó la clave, lo cargó en memoria.
- **Solución:** `ssh-add -d` para quitarla e `IdentityAgent none` en el alias. Se comprobó que la frase no quedó guardada en el llavero.
- **Lección:** comprobar con `ssh-add -l` qué claves tiene cargadas un agente.

## Líneas que se parten o explicaciones que se pegan

- **Síntoma:** `command not found` con trozos de texto, o un `grep` incompleto.
- **Causa:** al pegar en la terminal, una línea larga se partió, o se copió también la explicación.
- **Solución:** comandos cortos, de uno en uno, y cada directiva de configuración en su propia línea. Un error parecido en `~/.ssh/config` (dos directivas en la misma línea) impidió conectar a GitHub hasta separarlas.

## Un proceso detenido no muere con `kill`

- **Síntoma:** una sesión antigua de Claude Code seguía viva tras `kill <PID>`.
- **Causa:** el proceso estaba **detenido** (estado `T` en `ps`). La señal de terminar queda pendiente hasta que continúa; y al continuar, como estaba en segundo plano en su terminal, volvía a detenerse.
- **Solución:** `kill -KILL <PID>`, que no se puede ignorar ni posponer.
- **Lección:** mirar el estado con `ps -o pid,stat` antes de elegir la señal.

## Puesta en marcha de la cuenta del agente

- `adduser` en Debian 13 añade la cuenta al grupo `users` por defecto. Se retiró con `gpasswd -d`.
- `sudo -iu homelab-agent` conservó `DISPLAY` y `XAUTHORITY` (causa sin confirmar). Apuntaban a rutas inaccesibles; se eliminan con `unset` en el `~/.profile` del agente.
- `claude --dangerously-skip-permissions` **se ignoró en vez de rechazarse** en la versión 2.1.309. Lo válido fue comprobar que el modo seguía en `default` y que un `touch` no se ejecutaba sin aprobación.
- **Lección:** verificar el efecto real de cada control, no el mensaje del programa.

## El asistente de Jellyfin, empezado en dos sitios

- **Síntoma:** "Ha habido un error procesando la solicitud" al crear el usuario, en la tele y en el PC. El registro decía `CustomAuthentication was forbidden` y `IsStartupWizardCompleted` seguía en `false`.
- **Causa probable:** el asistente se empezó en el PC, llegó a crear el usuario y la biblioteca, pero no se terminó. Al repetirlo desde la tele, Jellyfin no dejaba volver a crear el usuario.
- **Solución:** borrar solo la configuración, que estaba vacía, y hacer el asistente del tirón desde el PC.
- **Lección:** los asistentes de primera configuración, en un solo dispositivo y hasta el final. Si algo falla, el registro antes que las suposiciones.

## Un `*` con `sudo` en una carpeta que no puedes leer

- **Síntoma:** `sudo cat /carpeta/archivo*.json` decía que no existía.
- **Causa:** el `*` lo expande la shell de tu usuario **antes** de `sudo`. Sin permiso para leer la carpeta, no se expande y `cat` recibe el `*` literal.
- **Solución:** `sudo sh -c 'cat /carpeta/archivo*.json'`, o el nombre completo.

## El manual de una opción decía lo contrario de lo que hace

- `google-authenticator -W` aparece en el manual como "desactivar la ventana de códigos", pero en modo TOTP el código fuente escribe `WINDOW_SIZE 3`: la ventana estándar (código anterior, actual y siguiente).
- **Lección:** ante una duda de seguridad, leer el código o comprobar el archivo resultante (`grep '^"' ~/.google_authenticator`).
