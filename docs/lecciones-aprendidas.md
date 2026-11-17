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

## AdGuard no arrancaba con `cap_drop: ALL`

- **Síntoma:** el contenedor se reiniciaba sin parar con `exec /opt/adguardhome/AdGuardHome: operation not permitted`.
- **Causa:** el Dockerfile de la imagen marca el binario con `setcap cap_net_bind_service=+eip`. Si el archivo pide una capacidad que no está en el conjunto límite del contenedor, el núcleo se niega a ejecutarlo.
- **Solución:** `cap_add: NET_BIND_SERVICE`. Con un usuario sin privilegios y `no-new-privileges`, el proceso no llega a tenerla.
- **Después:** en el primer arranque, AdGuard exigía root (`checkNetworkPermissions`, en su código). El asistente se hizo como root y después se volvió al usuario `adguard`.
- **Lección:** con `cap_drop: ALL`, mirar si la imagen tiene binarios con `setcap` (`getcap`).

## Una regla de entrada que bloqueaba la salida del contenedor

- **Síntoma:** en AdGuard, "Probar servidores de subida" fallaba, y los filtros no se descargaban.
- **Causa:** la regla `DOCKER-USER` "puerto original 53 y origen fuera de la LAN → `DROP`" también cazaba las consultas del contenedor a `9.9.9.9:53`. Esa salida pasa por `FORWARD`, su puerto original también es 53 y su origen es la red de Docker, no la LAN.
- **Solución:** añadir `--ctstate DNAT` para filtrar solo lo que entra por un puerto publicado. Después, `docker restart adguard` descargó los filtros.
- **Lección:** en `FORWARD` pasa el tráfico en las dos direcciones. Una regla por puerto tiene que distinguir entrada y salida.

## Las consultas de la tailnet llegaban como `172.19.0.1`

- **Síntoma:** con el DNS de Tailscale apuntando al servidor, AdGuard no mostraba la IP `100.x` del iPhone, sino la puerta de su red Docker.
- **Causa:** Tailscale marca lo que reenvía desde `tailscale0` y le hace NAT a la salida (`NoSNAT: false`). El tráfico hacia un contenedor se reenvía, así que también se le aplica.
- **Solución:** `tailscale set --snat-subnet-routes=false`. El firewall ya veía la IP real, porque `DOCKER-USER` actúa antes del NAT de salida.
- **Lección:** antes de dar una prueba por fallida, comprobar si la IP que falta aparece con otra cara.

## Samba trajo un controlador de dominio y winbind al login

- **Síntoma:** al instalar `samba` entraron 38 paquetes, entre ellos `samba-ad-dc`, `winbind`, `libpam-winbind` y `libnss-winbind`. Este último añadió `winbind` a `/etc/nsswitch.conf`, y `libpam-winbind` puso `pam_winbind` en todos los `/etc/pam.d/common-*`.
- **Por qué importa:** `sudo` incluye `common-auth` y `common-account`, y el SSH incluye `common-account` y `common-session` (no `common-auth`). Un servicio que no hacía falta quedaba en la cadena de autenticación del servidor.
- **Causa:** `samba-ad-dc` es un paquete *recomendado* de `samba`, y depende de `winbind`.
- **Solución:** con una copia de `/etc/pam.d` y una sesión de root abierta, `apt-get purge` de esos cinco paquetes (antes, `apt-get -s` para comprobar que `samba` se quedaba). Sus scripts de desinstalación limpiaron PAM y `nsswitch`. Después, `sudo` y SSH probados en ventanas nuevas.
- **Lección:** en un servidor, instalar con `--no-install-recommends` y leer la lista de paquetes **antes** de aceptar. Después de instalar, mirar si algo ha tocado `/etc/pam.d` o `/etc/nsswitch.conf`.

## El iPhone veía la carpeta de red, pero no copiaba nada

- **Síntoma:** desde la app Archivos, la carpeta compartida se veía, pero no se podía ni subir ni descargar. Windows funcionaba bien.
- **Causa:** la app Archivos espera las extensiones SMB de Apple, que Samba no trae activadas.
- **Solución:** `vfs objects = catia fruit streams_xattr` con `fruit:metadata = stream` en `[global]`, y `fruit:nfs_aces = no` para que el cliente no cambie permisos. Después, desconectar el servidor en el iPhone y **cerrar la app**: la sesión anterior seguía abierta sin `fruit`.
- **Además:** al principio se intentó subir dentro de `pc`, que es de solo lectura a propósito. Antes de buscar un fallo, comprobar que se está probando en el sitio correcto.

## Una carpeta "en uso" en Windows

- **Síntoma:** `Move-Item` no podía mover la carpeta de Syncthing: "está en uso".
- **Causas:** primero, la propia PowerShell estaba **dentro** de la carpeta. Después, Syncthing seguía abierto con dos procesos: un vigilante y el programa. Si se cierra el programa, el vigilante lo vuelve a abrir.
- **Solución:** `cd $HOME`, `Stop-Process -Name syncthing -Force` (cierra los dos) y copiar con `Copy-Item -Recurse` en vez de mover.
- **Lección:** antes de mover un programa en Windows, salir de su carpeta y comprobar con `Get-Process` que no queda nada en marcha.

## Windows entra en la carpeta de red con su propio usuario

- **Síntoma:** "La contraseña de red especificada no es correcta", sin haber escrito nada.
- **Causa:** Windows prueba primero el usuario con el que inició sesión. Después, si hay una conexión con otras credenciales, da el error 1219.
- **Solución:** `net use \\<IP_SERVIDOR>\pc /user:homelab\<usuario> *` (el `*` pide la contraseña sin mostrarla), y `net use * /delete /y` para cerrar las anteriores.

## La IP de la tailnet solo existe con Tailscale encendido

- **Síntoma:** la app de Immich decía "Server is not reachable" en casa, con `http://<IP_TAILSCALE_SERVIDOR>:2283`.
- **Causa:** la dirección `100.x` la da Tailscale. Con Tailscale apagado, el iPhone no sabe llegar a ella aunque esté en la misma red que el servidor. En casa, el servidor es `<IP_SERVIDOR>`.
- **Solución:** cambio automático de URL en la app (wifi de casa → `<IP_SERVIDOR>`; fuera → tailnet) y VPN On Demand en Tailscale. iOS pide la ubicación para leer el nombre del wifi: con **Al usar la app** basta.
- **Lección:** una dirección es de una red concreta. Para cada servicio, dejar claro qué dirección se usa en casa y cuál fuera.

## Docker no da al contenedor los grupos secundarios del usuario

- **Contexto:** Immich tenía que leer las imágenes del PC, que son del grupo `syncthing`. Con `user: immich`, el proceso no tiene ese grupo, aunque el usuario esté en él en el host.
- **Solución:** `group_add` con el GID de `syncthing` en el servicio, y el montaje `:ro`. Probado antes con `sudo -u immich -g syncthing touch …` (debe fallar).
- **Otro detalle:** `docker top <contenedor> -o user` falla con `Couldn't find PID field`; necesita `-o pid,user`.

## El manual de una opción decía lo contrario de lo que hace

- `google-authenticator -W` aparece en el manual como "desactivar la ventana de códigos", pero en modo TOTP el código fuente escribe `WINDOW_SIZE 3`: la ventana estándar (código anterior, actual y siguiente).
- **Lección:** ante una duda de seguridad, leer el código o comprobar el archivo resultante (`grep '^"' ~/.google_authenticator`).

## Un paquete oficial que daba root a su usuario

- **Contexto:** el agente de Beszel se iba a instalar con su `.deb` oficial.
- **Lo que se vio al inspeccionarlo** (`dpkg-deb -e`, sin instalar): el `postinst` crea el usuario `beszel` y, si existe el grupo `docker`, lo mete en él. Estar en `docker` equivale a ser root.
- **Solución:** el binario del mismo release, comprobado con el `checksums.txt`, y un servicio propio con `User=beszel` y sin capacidades.
- **Lección:** antes de instalar un paquete de fuera de Debian, leer sus scripts de mantenimiento.

## Windows se quedó con el DNS de respaldo

- **Síntoma:** tras recrear AdGuard, el navegador del PC daba "Server not found" con `homelab.home.arpa`, pero abría por la IP.
- **Causa:** mientras AdGuard estuvo parado, Windows pasó al DNS secundario que reparte el router (uno público) y se quedó con él un rato. El público no conoce `home.arpa`.
- **Solución:** desactivar y activar la red del PC, o esperar unos 15 minutos. `nslookup homelab.home.arpa` dice a qué servidor se está preguntando.
- **Lección:** cada reinicio de AdGuard deja la casa unos minutos sin filtrado. Reiniciarlo lo menos posible, y el último.

## Un script que creaba cosas antes de preguntar

- **Síntoma:** al repetir mal la contraseña, el script de Dozzle terminó, pero ya había creado `/srv/dozzle`. Al volver a ejecutarlo, se negaba porque la carpeta existía.
- **Solución:** pedir los datos al usuario **antes** de crear nada, y borrar lo creado si algo falla (`trap … ERR`).

## La tecla AltGr atascada

- **Síntoma:** en el escritorio del servidor, "hola" salía como `ħøłæ` y no funcionaban los atajos de GNOME, como cambiar de escritorio. Parecía un cambio de idioma.
- **Lo que no era:** la distribución seguía en `es` y no había teclas especiales de accesibilidad activas. Se comprobó con `gsettings` por SSH (con `DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<UID>/bus`).
- **Causa:** AltGr (Alt derecha) estaba pulsada para GNOME: con ella, `h` da `ħ`, `o` da `ø`, `l` da `ł` y `a` da `æ`.
- **Solución:** pulsar y soltar AltGr. Si el atasco es del teclado virtual de Sunshine, reiniciar su servicio de usuario lo suelta.
