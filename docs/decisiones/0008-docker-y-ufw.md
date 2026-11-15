# ADR 0008: Docker y UFW, puertos solo en el propio equipo por defecto

- **Estado:** aceptada (2026-11-15).
- **Decide:** el dueño.

## Contexto

La documentación oficial de Docker avisa: "when you expose container ports using Docker, these ports bypass your firewall rules". Un `-p 8080:80` publica en todas las interfaces y llega desde la LAN aunque UFW no lo permita. Los servicios previstos (Caddy, AdGuard, Immich, Syncthing…) publicarán puertos. Además, Docker dice que no da soporte a reglas creadas directamente con `nft`, que es como está hecho el filtro de salida del agente ([ADR 0007](0007-filtro-de-salida-del-agente.md)).

## Opciones

| Opción | A favor | En contra |
|---|---|---|
| Publicar sin más y confiar en UFW | Cómodo | UFW no se aplica: todo queda abierto a la LAN |
| `ufw-docker` (script de terceros) | Integra Docker con las reglas de UFW | Dependencia externa que modifica `after.rules` |
| Backend nftables de Docker 29 | Más moderno | Experimental según la documentación; sin `DOCKER-USER` |
| **`"ip": "127.0.0.1"` en `daemon.json` y publicar en la LAN a propósito** | Lo publicado por defecto no sale del equipo; abrir algo exige escribir la IP | Hay que recordarlo en cada servicio que deba verse desde casa |

## Decisión

- `/etc/docker/daemon.json` con `"ip": "127.0.0.1"`, `"log-driver": "local"` y `"live-restore": true`.
- Un servicio que deba verse desde casa publica con la IP del servidor de forma explícita y, en su fase, se añade una regla en `DOCKER-USER` que limita el origen a `<RED_LAN>`.
- Docker se usa siempre con `sudo`: el grupo `docker` queda vacío, porque controlar Docker equivale a ser root.
- Docker entra en las actualizaciones automáticas (`origin=Docker,label=Docker CE,codename=trixie`).

## Consecuencias

- **Comprobado:** un contenedor con `-p 8088:8080` escucha solo en `127.0.0.1:8088`.
- **Comprobado:** con Docker activo, el filtro de salida del agente sigue funcionando, incluido el rechazo hacia la red de Docker (`172.17.0.1`). La combinación no está soportada oficialmente: se repite la prueba tras cada actualización grande de Docker.
- Una actualización grande de Docker (`AA.MM`) reinicia los contenedores: `live-restore` solo cubre las de parche (`AA.MM.x`). Ocurrirá de madrugada.
- Escaneo de puertos desde otro equipo de la LAN: en la Fase 8.
- **Actualización (2026-11-15):** Jellyfin se publica en `0.0.0.0` (solo IPv4) para servir también a la tailnet, y su regla de `DOCKER-USER` limita el origen a `<RED_LAN>` y a `100.64.0.0/10` por `tailscale0` ([ADR 0009](0009-tailscale.md)).
