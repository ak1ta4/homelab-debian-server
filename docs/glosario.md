# Glosario y marcadores

## Marcadores

Las guías nunca muestran valores reales. Usan estos marcadores; los valores de verdad están en un inventario local del servidor, fuera de Git.

| Marcador | Qué es |
|---|---|
| `<IP_SERVIDOR>` | IP del servidor en la red de casa (reserva DHCP) |
| `<IP_PC>` | IP del PC principal (reserva DHCP). Es la IP de rescate que Fail2ban nunca bloquea |
| `<IP_ROUTER>` | Puerta de enlace de la red de casa |
| `<RED_LAN>` | Red de casa en notación CIDR, por ejemplo `x.x.x.0/24` |
| `<IP_SUMIDERO>` | IP de la LAN sin uso, fuera de los rangos DHCP. No se asigna nunca a ningún equipo |
| `<IP_PUBLICA>` | IP pública de la conexión |
| `<USUARIO_ADMIN>` | Cuenta administradora del servidor, la única en `sshusers` por ahora |
| `<USUARIO_GITHUB>` | Usuario de GitHub dueño del repositorio |
| `<DOMINIO>` | Dominio propio (Fase 4) |

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
