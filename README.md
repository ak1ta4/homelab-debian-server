# Homelab en un portátil con Debian

Servidor doméstico privado montado sobre un portátil con Debian 13 y GNOME: sincronización de archivos, notas de Obsidian, fotos, DNS con filtrado para toda la casa y acceso remoto seguro. Todo funciona con Docker y **nada se expone a internet**: se accede desde la red de casa o desde una tailnet.

El repositorio documenta **lo que se hizo de verdad**, paso a paso. Así se puede reconstruir a mano, un principiante puede seguirlo y se entiende el porqué de cada decisión.

## Estado

Ver [`docs/PROGRESO.md`](docs/PROGRESO.md).

## Arquitectura (prevista)

| Área | Pieza |
|---|---|
| Acceso remoto | Tailscale en el host, con ACL de mínimo privilegio |
| Acceso de emergencia | SSH solo desde la LAN, clave + TOTP, Fail2ban |
| Contenedores | Docker Engine oficial y Compose, un stack por servicio |
| HTTPS | Caddy con certificado comodín por DNS-01 |
| DNS | AdGuard Home para la casa y la tailnet |
| Archivos | Syncthing + File Browser |
| Notas | Obsidian con Self-hosted LiveSync (CouchDB, cifrado de extremo a extremo) |
| Fotos | Immich |
| Copias | restic hacia el PC, con `rest-server` en modo `append-only` |
| Panel | Homepage, Beszel y Dozzle |
| Escritorio remoto | Sunshine bajo demanda, cliente Moonlight |
| IA | Claude Code con un usuario sin privilegios y cambios revisados por una persona |

## Cómo leerlo

1. [`docs/glosario.md`](docs/glosario.md): términos y marcadores como `<IP_SERVIDOR>`.
2. `docs/00-conceptos.md` y las guías por fase (`docs/01-…` a `docs/08-…`), en orden.
3. `docs/decisiones/`: un ADR por decisión, con las alternativas descartadas.
4. `stacks/<servicio>/`: `compose.yaml`, `.env.example` y su README.
5. `scripts/`: scripts comentados, cada uno con su equivalente manual en la guía.

## Seguridad del repositorio

- No contiene secretos ni datos identificativos: los valores reales viven fuera de Git, en el servidor.
- gitleaks revisa cada commit, cada parche aplicado y cada subida, con una configuración local que también bloquea datos de la red de casa. En GitHub lo repite un workflow.

## Fuentes

La securización se basa en [How To Secure A Linux Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server), de imthenachoman, bajo licencia [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). Las guías están escritas con palabras propias y la citan como fuente.
