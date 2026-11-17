# ADR 0010: DNS de casa y de la tailnet con AdGuard Home

- **Estado:** aceptada (2026-11-16).
- **Decide:** el dueño.

## Contexto

El dueño quiere bloquear anuncios y rastreo en todos los dispositivos, en casa y fuera. En la Fase 4 se aplazan Caddy, HTTPS y el dominio propio: solo AdGuard. El DNS de la casa lo reparte el router por DHCP. El agente de IA no puede usar DNS de la LAN ni de Docker: su filtro de salida lo bloquea ([ADR 0007](0007-filtro-de-salida-del-agente.md)).

## Opciones

**Qué DNS reparte el router:**

| Opción | A favor | En contra |
|---|---|---|
| Solo AdGuard | Todo pasa por el filtro | Si el servidor se apaga, la casa se queda sin internet |
| **AdGuard + uno público de respaldo** | La casa sigue navegando si el servidor falla | Algunos dispositivos usan a veces el secundario y se cuela algún anuncio |
| Dejar el router y configurar AdGuard en cada dispositivo | Sin tocar el router | Hay que hacerlo uno a uno; la tele y otros aparatos no lo permiten |

**DNS del propio servidor:**

| Opción | A favor | En contra |
|---|---|---|
| El que reparta el router (AdGuard) | El servidor también filtra | El agente se queda sin DNS; al arrancar no hay DNS hasta que Docker levanta AdGuard |
| **Fijo en los del operador** | Independiente del contenedor y compatible con el filtro del agente | El servidor no filtra anuncios (no navega) |

## Decisión

- AdGuard Home en Docker, con un usuario sin privilegios, DoH hacia Quad9 y Cloudflare, y DNSSEC.
- Router, solo en el pool DHCP de la casa: primero `<IP_SERVIDOR>` y de respaldo `9.9.9.9`. Los pools del decodificador de Movistar TV y de los aparatos de Telefónica no se tocan.
- El servidor fija sus DNS en los del operador (`ipv4.ignore-auto-dns`), igual que antes.
- Tailscale: el servidor como único nameserver global con **Override DNS servers**, y `udp:53` y `tcp:53` en la política. Sin respaldo público: Tailscale pregunta a todos los nameservers en paralelo y el público anularía el filtro.
- `--snat-subnet-routes=false` en el servidor, para que AdGuard y Jellyfin vean la IP de cada dispositivo de la tailnet. Amplía el [ADR 0009](0009-tailscale.md).

## Consecuencias

- **Comprobado:** cuatro dispositivos de casa y el iPhone con datos pasan por AdGuard, y se bloquean dominios de publicidad. El servidor y el agente siguen resolviendo.
- Fuera de casa, si el servidor se cae, un dispositivo con Tailscale conectado se queda sin DNS hasta desconectar Tailscale.
- Con el respaldo, el filtrado en casa no es completo. Si molesta, se puede quitar el secundario del router (a cambio de depender del servidor).
- La red de casa no tiene IPv6 global. Si el operador la activa, hay que revisar que el router no anuncie su propio DNS por IPv6.
- Si el servidor cambia de IP, la casa pierde el DNS principal: depende de la reserva DHCP.
- **Actualización (2026-11-17):**
  - AdGuard responde además el nombre `homelab.home.arpa`, con una IP para la casa y otra para la tailnet ([ADR 0013](0013-panel-con-homepage.md)).
  - Se añaden las listas de HaGeZi contra amenazas (TIF), la Pro y la de DNS cifrados, y el registro de consultas pasa a 7 días ([Fase 4, 4.8](../04-dns.md#48-filtros-de-amenazas)). La memoria de AdGuard pasa de 92 a 427 MB.
  - Hay un segundo usuario de AdGuard, `panel`, para los datos del panel.
  - Windows se queda un rato con el DNS de respaldo cuando AdGuard se reinicia: durante ese tiempo no hay filtrado ni `homelab.home.arpa`.
