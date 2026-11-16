# ADR 0009: acceso desde fuera de casa con Tailscale

- **Estado:** aceptada (2026-11-15).
- **Decide:** el dueño.

## Contexto

El dueño quiere usar SSH, Moonlight y Jellyfin fuera de casa. Ningún servicio se expone a internet y el router no se toca (CLAUDE.md, [ADR 0004](0004-tapar-ssh-del-router.md)). La conexión no tiene CGNAT.

## Opciones

| Opción | A favor | En contra |
|---|---|---|
| Abrir puertos en el router | Sin terceros | Expone servicios a internet: descartado de entrada |
| WireGuard propio | Sin terceros | Exige abrir un puerto UDP en el router y gestionar claves a mano |
| **Tailscale** | Sin tocar el router; claves gestionadas; reglas por dispositivo y puerto | Se confía en Tailscale como coordinador; cuenta en un tercero |

## Decisión

- Tailscale desde su repositorio oficial (clave comprobada en tres sitios), con pin a `tailscale` y en las actualizaciones automáticas.
- El servidor entra con la etiqueta `tag:server`: su clave no caduca y no pertenece a ningún usuario.
- Política con *grants*: los dispositivos del dueño solo llegan al servidor, y solo a SSH (TCP 22), Jellyfin (TCP 8096) y Sunshine (TCP 47984, 47989 y 48010; UDP 47998-48000). Sin el panel de Sunshine. El servidor no llega a ningún dispositivo. Sin las reglas por defecto, que lo permiten todo.
- Sin Tailscale SSH, sin rutas, sin nodo de salida y **sin el DNS de Tailscale** en el servidor (`--accept-dns=false`).
- UFW abre esos puertos solo en `tailscale0`, más UDP 41641 para intentar conexiones directas. En el router no se abre nada.
- Jellyfin se publica en todas las IPv4 y se limita en `DOCKER-USER` a la LAN y a la tailnet entrando por `tailscale0`. Cambia la forma de publicar del [ADR 0008](0008-docker-y-ufw.md), porque la IP de Tailscale puede no existir aún cuando arranca Docker. En Jellyfin, la tailnet cuenta como red local.

## Consecuencias

- **Comprobado** desde el iPhone con datos móviles: Jellyfin y Moonlight funcionan. La conexión va por el repetidor de Madrid (40 ms); la red móvil no permite la conexión directa.
- Con el DNS de Tailscale, el agente se queda sin DNS: su filtro de salida ([ADR 0007](0007-filtro-de-salida-del-agente.md)) bloquea `100.64.0.0/10`. Ocurrió al conectar y se corrigió con `tailscale set --accept-dns=false`.
- Riesgo que queda: quien controle la cuenta de Tailscale puede añadir dispositivos a la tailnet. Mitigación: la cuenta con verificación en dos pasos. Tailnet Lock queda como mejora (exige dos nodos firmantes).
- Si el dueño invita a otra persona a la tailnet, `autogroup:member` le daría el mismo acceso: habría que revisar la política.
- **Actualización (2026-11-16):** la política añade DNS (`udp:53` y `tcp:53`) hacia el servidor, que es el nameserver de la tailnet, y el servidor usa `--snat-subnet-routes=false` ([ADR 0010](0010-dns-de-casa-con-adguard.md)).
