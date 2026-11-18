# Glosario y marcadores

## Marcadores

Las guías nunca muestran valores reales. Usan estos marcadores; los valores de verdad están en un inventario local del servidor, fuera de Git.

| Marcador | Qué es |
|---|---|
| `<IP_SERVIDOR>` | IP del servidor en la red de casa (reserva DHCP) |
| `<IP_PC>` | IP del PC principal (reserva DHCP). Es la IP de rescate que Fail2ban nunca bloquea |
| `<IP_ROUTER>` | Puerta de enlace de la red de casa |
| `<RED_LAN>` | Red de casa en notación CIDR, por ejemplo `x.x.x.0/24` |
| `<RED_LAN_MASCARA>` | La misma red con la máscara escrita entera, `x.x.x.0/255.255.255.0`: es la forma que documenta `hosts allow` de Samba |
| `<IP_SUMIDERO>` | IP de la LAN sin uso, fuera de los rangos DHCP. No se asigna nunca a ningún equipo |
| `<IP_PUBLICA>` | IP pública de la conexión |
| `<USUARIO_ADMIN>` | Cuenta administradora del servidor. En `sshusers` está también `servercat`, sin `sudo`, para el widget del iPhone |
| `<USUARIO_GITHUB>` | Usuario de GitHub dueño del repositorio |
| `<USUARIO_WINDOWS>` | Usuario de Windows en el PC. El usuario de Samba tiene el mismo nombre, pero otra contraseña |
| `<IP_BLOQUEADA>` | IP que Fail2ban ha bloqueado, tal como sale en `fail2ban-client status sshd` |
| `<ID_DISCO_DATOS>` | Nombre del disco de datos en `/dev/disk/by-id/`. Incluye el número de serie |
| `<RANGO_PRIVADO_16>` | El rango privado `/16` de la RFC 1918 (el de los routers domésticos), entero. No identifica la casa, pero la regla de gitleaks que protege la IP de la LAN bloquea cualquier dirección de ese rango |
| `<UUID_DATOS>` | UUID del sistema de archivos btrfs del disco de datos (`lsblk -o NAME,UUID`) |
| `<IP_TAILSCALE_SERVIDOR>` | IP del servidor en la tailnet (`100.x.y.z`) |
| `<DOMINIO>` | Dominio propio (aplazado) |
| `<CIUDAD>`, `<LATITUD>`, `<LONGITUD>` | Pueblo del servidor y sus coordenadas con dos decimales (las del pueblo, no las de la casa), para el tiempo del panel |
| `<DNS_OPERADOR>` | Los dos DNS del operador, separados por coma, tal como los repartía el router |
| `<RED_LAN_REGEX>`, `<IP_PUBLICA_REGEX>`, `<OTROS_NOMBRES>` | Valores reales escritos como expresión regular, solo en la configuración local de gitleaks |
| `AAAAMMDD` | Fecha del día en las copias de seguridad, por ejemplo `20261108` |

Los marcadores se escriben con los `< >` y se sustituyen enteros, sin dejar los símbolos.

## Términos

| Término | Significado |
|---|---|
| ADR | *Architecture Decision Record*: documento corto con contexto, opciones, decisión y consecuencias |
| Clave de despliegue | Clave SSH que da acceso a un único repositorio de GitHub |
| Fail2ban | Programa que lee los registros y bloquea durante un tiempo las IP que fallan al entrar |
| gitleaks | Herramienta que busca secretos en un repositorio Git |
| Hook de Git | Script que Git ejecuta automáticamente en ciertos momentos, como antes de un commit |
| PAM | Sistema de Linux que decide cómo se autentica cada servicio |
| Reserva DHCP | Regla del router que da siempre la misma IP a un equipo, según su MAC |
| TOTP | Código de un solo uso que cambia cada 30 segundos, generado por una app del móvil |
| Tailnet | Red privada de Tailscale que une los dispositivos propios |
