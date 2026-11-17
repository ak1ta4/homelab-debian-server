# Homelab en un portátil con Debian

![Debian 13.7](https://img.shields.io/badge/Debian-13.7-A81D33?logo=debian&logoColor=white)
![Docker Engine 29.9.3](https://img.shields.io/badge/Docker%20Engine-29.9.3-2496ED?logo=docker&logoColor=white)
![Compose 5.7.1](https://img.shields.io/badge/Compose-5.7.1-2496ED?logo=docker&logoColor=white)
![Tailscale 1.104.7](https://img.shields.io/badge/Tailscale-1.104.7-242424?logo=tailscale&logoColor=white)

![AdGuard Home v0.107.80](https://img.shields.io/badge/AdGuard%20Home-v0.107.80-68BC71?logo=adguard&logoColor=white)
![Jellyfin 12.4](https://img.shields.io/badge/Jellyfin-12.4-00A4DC?logo=jellyfin&logoColor=white)
![Syncthing 2.1.7](https://img.shields.io/badge/Syncthing-2.1.7-0891D1?logo=syncthing&logoColor=white)
![Samba 4.22.11](https://img.shields.io/badge/Samba-4.22.11-555555)
![Immich v3.4.1](https://img.shields.io/badge/Immich-v3.4.1-4250AF?logo=immich&logoColor=white)
![Homepage v2.8.0](https://img.shields.io/badge/Homepage-v2.8.0-FE8019)
![Beszel 0.21.2](https://img.shields.io/badge/Beszel-0.21.2-0EA5E9)
![Dozzle v11.4.3](https://img.shields.io/badge/Dozzle-v11.4.3-F59E0B)

![Expuesto a internet: nada](https://img.shields.io/badge/expuesto%20a%20internet-nada-2E7D32)
![Secretos: gitleaks 8.30.1](https://img.shields.io/badge/secretos-gitleaks%208.30.1-2E7D32)
![Fase 5 en curso](https://img.shields.io/badge/fase-5%20en%20curso-F0A30A)

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
| Archivos | Syncthing del PC al servidor + Samba |
| Notas | Obsidian con Self-hosted LiveSync (CouchDB, cifrado de extremo a extremo) |
| Fotos | Immich |
| Películas y series | Jellyfin |
| Copias | restic hacia el PC, con `rest-server` en modo `append-only` |
| Panel | Homepage, Beszel y Dozzle |
| Escritorio remoto | Sunshine bajo demanda, cliente Moonlight |
| IA | Claude Code con un usuario sin privilegios y cambios revisados por una persona |

## Cómo leerlo

1. [`docs/glosario.md`](docs/glosario.md): términos y marcadores como `<IP_SERVIDOR>`.
2. [`docs/00-conceptos.md`](docs/00-conceptos.md): las ideas básicas, explicadas desde cero.
3. Las guías, en orden:
   - [Fase 0: auditoría](docs/00-auditoria.md)
   - [Fase 1: base y endurecimiento](docs/01-base-y-endurecimiento/README.md)
   - [Fase 2: escritorio remoto](docs/02-escritorio-remoto.md)
   - [Fase 3: acceso desde fuera](docs/03-acceso-remoto.md)
   - [Fase 4: DNS con bloqueo de anuncios](docs/04-dns.md)
   - [Fase 5: servicios](docs/05-servicios/README.md) (en curso)
   - Las siguientes fases, `docs/06-…` a `docs/08-…`, a medida que se hagan.
4. [`docs/decisiones/`](docs/decisiones/README.md): un ADR por decisión, con las alternativas descartadas.
5. [`docs/guia-securizacion.md`](docs/guia-securizacion.md): qué se aplicó, adaptó o descartó de la guía de referencia.
6. [`docs/lecciones-aprendidas.md`](docs/lecciones-aprendidas.md) y [`docs/entrevista.md`](docs/entrevista.md).
7. `stacks/<servicio>/`: `compose.yaml`, `.env.example` y su README.
8. `scripts/`: scripts comentados, cada uno con su equivalente manual en la guía.

Cada guía tiene las mismas secciones:
1. Qué y por qué.
2. Conceptos.
3. Requisitos.
4. Pasos con cada comando explicado.
5. Verificación.
6. Errores frecuentes.
7. Cómo revertirlo.
8. Seguridad.
9. Para la entrevista.

## Seguridad del repositorio

- No contiene secretos ni datos identificativos: los valores reales viven fuera de Git, en el servidor.
- gitleaks revisa cada commit, cada parche aplicado y cada subida, con una configuración local que también bloquea datos de la red de casa. En GitHub lo repite un workflow.

## Fuentes

La securización se basa en [How To Secure A Linux Server](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server), de imthenachoman, bajo licencia [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). Las guías están escritas con palabras propias y la citan como fuente.
