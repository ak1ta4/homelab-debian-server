# Fase 1: base y endurecimiento

Convierte el portátil en un servidor con un acceso de rescate que no dependa del escritorio remoto, con la documentación en Git, el disco de datos preparado, actualizaciones de seguridad, firewall y Docker.

| Paso | Guía | Estado |
|---|---|---|
| 1. Reservas DHCP del servidor y del PC | [`01-reservas-dhcp.md`](01-reservas-dhcp.md) | Hecho |
| 2. SSH de emergencia: clave, TOTP y Fail2ban | [`02-ssh-de-emergencia.md`](02-ssh-de-emergencia.md) | Hecho |
| 3. Repositorio, gitleaks y GitHub | [`03-repositorio.md`](03-repositorio.md) | Hecho |
| 4. Portátil como servidor (tapa, suspensión, batería) | [`04-portatil-como-servidor.md`](04-portatil-como-servidor.md) | En curso |
| 5. Disco de datos (btrfs) | [`05-disco-de-datos.md`](05-disco-de-datos.md) | Hecho. Falta la prueba de arranque |
| 6. Actualizaciones de seguridad | [`06-actualizaciones-de-seguridad.md`](06-actualizaciones-de-seguridad.md) | Hecho. Falta la prueba de reinicio |
| 7. Firewall (UFW) y filtro de salida del agente | [`07-firewall.md`](07-firewall.md) | Hecho. Falta la prueba de reinicio |
| 8. Docker | [`08-docker.md`](08-docker.md) | Hecho. Falta la prueba de reinicio |
| 9. Resto de la base y Lynis | [`09-resto-de-la-base.md`](09-resto-de-la-base.md) | Hecho. Falta la prueba de reinicio y Termius |

**Orden:** el SSH va antes que cualquier otro cambio que pueda cortar el acceso. A partir del paso 2, los pasos que puedan cortar el escritorio remoto se lanzan **por SSH desde el PC, dentro de tmux**. Las reservas DHCP se adelantaron al SSH porque la IP del servidor podía cambiar cada 12 horas y Fail2ban necesita una IP fija del PC de rescate.

**Salida de la fase:** reinicio con la tapa cerrada; el equipo vuelve solo, con el disco montado, SSH accesible solo con clave y TOTP, y Moonlight funcionando con la tapa entreabierta (decisión del dueño: sin pantalla virtual ni enchufe HDMI falso).
