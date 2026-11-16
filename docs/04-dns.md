# Fase 4: DNS con bloqueo de anuncios (AdGuard Home)

> **Versiones usadas:** Debian 13.7, Docker Engine 29.9.3, Compose 5.7.1, imagen `adguard/adguardhome:v0.107.80`, Tailscale 1.104.7.
>
> **Alcance:** solo AdGuard, por decisión del dueño (2026-11-15). Caddy, HTTPS y el dominio propio quedan para más adelante.

## 1. Qué vamos a hacer y por qué

**En sencillo:** un "filtro" de nombres para toda la casa. Cuando un móvil o la tele preguntan dónde está un servidor de anuncios o de rastreo, el servidor responde "no existe" y el anuncio no carga. Funciona en todos los dispositivos sin instalar nada en ellos, y también fuera de casa por Tailscale.

**Técnicamente:**
- AdGuard Home en Docker, con un usuario propio sin privilegios. Pregunta a internet por DNS cifrado (DoH) y valida DNSSEC.
- El router reparte por DHCP el servidor como DNS principal y uno público como respaldo.
- La tailnet usa el servidor como DNS global.
- El propio servidor **no** usa AdGuard.

Decisión y alternativas: [ADR 0010](decisiones/0010-dns-de-casa-con-adguard.md).

## 2. Conceptos previos

- **DNS:** traduce nombres (`debian.org`) a direcciones IP. Quien responde el DNS decide a dónde va cada dispositivo.
- **Bloqueo por DNS:** a los dominios de las listas de filtros se les responde `0.0.0.0`. Si un anuncio llega desde el mismo dominio que el contenido, como pasa en YouTube, no se puede bloquear así.
- **DoH (DNS sobre HTTPS):** las preguntas de AdGuard a internet van cifradas, y el operador no las ve.
- **Bootstrap:** DNS normal que solo sirve para averiguar la IP de los servidores DoH.
- **DNSSEC:** firma de las respuestas DNS. AdGuard comprueba que nadie las ha alterado.
- **Pool DHCP:** cada bloque de reparto de IP del router, con su rango y sus DNS. El router de Movistar tiene varios, según el tipo de aparato.

## 3. Requisitos y estado inicial

- Docker del [paso 8](01-base-y-endurecimiento/08-docker.md) y la reserva DHCP del servidor ([paso 1](01-base-y-endurecimiento/01-reservas-dhcp.md)). Si el servidor cambiara de IP, toda la casa perdería el DNS principal.
- Tailscale de la [Fase 3](03-acceso-remoto.md).
- Nada escuchando en el puerto 53 del servidor:
  ```bash
  sudo ss -lunpt 'sport = :53'
  ```
  ✅ Vacío.

## 4. Pasos

En el servidor, con `<USUARIO_ADMIN>`. Se hizo con scripts revisados antes de ejecutarlos; estos son sus pasos.

### 4.1 Usuario, carpetas y Compose

```bash
sudo adduser --system --group --no-create-home --home /nonexistent adguard
```
```bash
sudo install -d -m 755 /srv/adguard
```
```bash
sudo install -d -o adguard -g adguard -m 700 /srv/adguard/work /srv/adguard/conf
```
Copia [`compose.yaml`](../stacks/adguard/compose.yaml) en `/srv/adguard/` y crea `.env` a partir de [`.env.example`](../stacks/adguard/.env.example).

Dentro, AdGuard escucha en **puertos altos** (DNS en 5353 y panel en 3000), y Docker publica el 53 de fuera hacia el 5353. Así el proceso no necesita root.

`cap_add: NET_BIND_SERVICE` hace falta aunque no se use. El binario de la imagen lleva esa capacidad marcada con `setcap`, y con `cap_drop: ALL` el núcleo se niega a ejecutarlo (`operation not permitted`). Con un usuario sin privilegios y `no-new-privileges`, el proceso no llega a tenerla.

### 4.2 Reglas en `DOCKER-USER`

Como con Jellyfin, los puertos de Docker no pasan por UFW. `/etc/systemd/system/homelab-adguard-lan.service`:
```ini
[Unit]
Description=Homelab: AdGuard (DNS LAN y tailnet; panel solo LAN), en DOCKER-USER
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctstate DNAT --ctorigdstport 3000 --ctdir ORIGINAL ! -s <RED_LAN> -j DROP
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctstate DNAT --ctorigdstport 53 --ctdir ORIGINAL ! -s <RED_LAN> -j DROP
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p udp -m conntrack --ctstate DNAT --ctorigdstport 53 --ctdir ORIGINAL ! -s <RED_LAN> -j DROP
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctstate DNAT --ctorigdstport 53 --ctdir ORIGINAL -i tailscale0 -s 100.64.0.0/10 -j RETURN
ExecStart=/usr/sbin/iptables -I DOCKER-USER -p udp -m conntrack --ctstate DNAT --ctorigdstport 53 --ctdir ORIGINAL -i tailscale0 -s 100.64.0.0/10 -j RETURN
```
Lleva también un `ExecStop=-… -D DOCKER-USER …` por cada regla y `WantedBy=docker.service` en `[Install]`. Cada `-I` inserta arriba, así que los dos `RETURN` de la tailnet quedan primero.

**`--ctstate DNAT` es imprescindible.** Sin él, la regla del puerto 53 también caza las consultas que **el propio contenedor** hace a internet (el bootstrap a `9.9.9.9:53`), porque esa salida también pasa por `FORWARD` y su origen no es la LAN. Con `DNAT`, solo se filtra lo que entra por un puerto publicado.

```bash
sudo systemctl daemon-reload
```
```bash
sudo systemctl enable --now homelab-adguard-lan.service
```
```bash
sudo iptables -S DOCKER-USER
```
✅ Los dos `RETURN` con `tailscale0`, después los tres `DROP` y, más abajo, las reglas de Jellyfin.

### 4.3 Primer arranque y asistente

En el **primer** arranque, sin configuración, AdGuard exige ser root o tener `CAP_NET_BIND_SERVICE` como capacidad ambiental. Docker no da capacidades ambientales, así que con el usuario `adguard` se para con un aviso de que debe ejecutarse como administrador. Por eso el asistente se hace como root (con las mismas restricciones), y después se vuelve al usuario `adguard`.

1. En `compose.yaml`, comenta la línea `user:` y deja `work/` y `conf/` de root:
   ```bash
   sudo chown root:root /srv/adguard/work /srv/adguard/conf
   ```
   ```bash
   sudo docker compose -f /srv/adguard/compose.yaml up -d
   ```
2. Desde el PC, en `http://<IP_SERVIDOR>:3000`, **del tirón**:
   - Interfaz web: todas las interfaces, puerto **3000**.
   - Servidor DNS: todas las interfaces, puerto **5353**.
   - Usuario y contraseña del panel, guardados en el gestor de contraseñas.
3. De vuelta al usuario `adguard`:
   ```bash
   sudo docker compose -f /srv/adguard/compose.yaml down
   ```
   ```bash
   sudo chown -R adguard:adguard /srv/adguard/work /srv/adguard/conf
   ```
   Descomenta `user:` en `compose.yaml` y:
   ```bash
   sudo docker compose -f /srv/adguard/compose.yaml up -d
   ```
   ```bash
   sudo docker top adguard -o user,pid,args
   ```
   ✅ El proceso con el UID de `adguard`, no `root`. Con configuración ya creada, no repite la comprobación.

### 4.4 Ajustes de AdGuard

**Configuración** → **Configuración DNS**:
- **Servidores DNS de subida**, sustituyendo los que trae:
  ```
  https://dns.quad9.net/dns-query
  https://cloudflare-dns.com/dns-query
  ```
- **Solicitudes paralelas**.
- **Servidores DNS de arranque** (bootstrap), sustituyendo también los que trae: `9.9.9.9` y `1.1.1.1`.
- **Habilitar DNSSEC**.

Pulsa **Probar servidores de subida**: ✅ los dos en verde. Guarda.

**Filtros** → **Listas de bloqueo DNS**: la *AdGuard DNS filter* que trae. Si la lista sale sin reglas, reinicia el contenedor:
```bash
sudo docker restart adguard
```

Prueba desde el PC, preguntando directamente al servidor:
```
nslookup ad.doubleclick.net <IP_SERVIDOR>
```
✅ `0.0.0.0`. Y `nslookup debian.org <IP_SERVIDOR>` da su IP real.

### 4.5 El servidor no usa AdGuard

El servidor sigue con los DNS del operador, aunque el router reparta AdGuard:
- El filtro de salida del agente ([ADR 0007](decisiones/0007-filtro-de-salida-del-agente.md)) bloquea la LAN y Docker, y el agente se quedaría sin DNS.
- Si el servidor dependiera de su propio contenedor, al arrancar no tendría DNS hasta que Docker levantara AdGuard.

```bash
sudo nmcli con mod "Wired connection 1" ipv4.ignore-auto-dns yes ipv4.dns "<DNS_OPERADOR>"
```
```bash
sudo nmcli con mod "Wired connection 1" ipv6.ignore-auto-dns yes
```
```bash
sudo nmcli device reapply eno1
```
```bash
grep nameserver /etc/resolv.conf
```
✅ Los mismos DNS que antes, y `getent hosts debian.org` resuelve.

### 4.6 Router: DNS para toda la casa

**Confirmación expresa del dueño antes de tocarlo.**

En el menú avanzado del router: **LAN Setting** → **LAN DHCP**. Tiene **tres pools**; ábrelos uno a uno sin cambiar nada:

| Pool | *Pool Condition* | Qué es | ¿Se toca? |
|---|---|---|---|
| Vendor ID `[IAL]` | Marcado | Decodificador de Movistar TV, con su DNS interno | **No** |
| Vendor ID `TEF_IOT` | Marcado | Aparatos de Telefónica | **No** |
| **Sin condiciones** | Nada marcado | La red de casa | **Sí** |

En el de casa, anota los DNS actuales y cambia **solo** la fila **DNS**:
- primero: `<IP_SERVIDOR>`;
- segundo: `9.9.9.9` (Quad9), de respaldo.

**Apply.** En **Static Lease**, comprueba que la reserva del servidor sigue ahí, con la MAC de `eno1`:
```bash
ip -br link show eno1
```

Los dispositivos cogen el DNS nuevo al renovar la IP: apaga y enciende su Wi-Fi.

### 4.7 Tailscale: AdGuard también fuera de casa

1. En la política (**Access controls** → **JSON editor**), añade `"udp:53", "tcp:53"` a la lista `ip` del grant de la [Fase 3](03-acceso-remoto.md#41-política-de-acceso). **Antes que el paso 2**: si no, los dispositivos se quedan sin DNS.
2. **DNS** → **Nameservers** → **Add nameserver** → **Custom…**: `<IP_TAILSCALE_SERVIDOR>`, sin *Restrict to domain* ni *Use with exit node*. Guarda y activa **Override DNS servers**.

   **Solo ese.** Con varios nameservers, Tailscale pregunta a todos en paralelo y se queda con la respuesta más rápida, así que uno público dejaría pasar los anuncios.
3. Para que AdGuard vea la IP de cada dispositivo de la tailnet, y no la de la red de Docker:
   ```bash
   sudo tailscale set --snat-subnet-routes=false
   ```
   Tailscale marca lo que reenvía desde `tailscale0` y le hace NAT a la salida. Como el tráfico hacia un contenedor se reenvía, AdGuard veía `172.19.0.1`, la puerta de su red Docker. El servidor no anuncia rutas, así que solo cambia la IP que ven los contenedores.

El servidor sigue con `--accept-dns=false`:
```bash
tailscale debug prefs | grep -E '"CorpDNS"|"NoSNAT"'
```
✅ `"CorpDNS": false` y `"NoSNAT": true`. `/etc/resolv.conf`, sin cambios.

## 5. Verificación

- [x] `nslookup ad.doubleclick.net <IP_SERVIDOR>` da `0.0.0.0` y `debian.org` resuelve (desde el PC).
- [x] Servidores de subida en verde, con DoH y DNSSEC.
- [x] El contenedor corre con el UID de `adguard`.
- [x] El servidor sigue con los DNS del operador y el agente resuelve.
- [x] Tras el cambio del router, cuatro dispositivos de casa aparecen en el panel y se bloquean dominios de publicidad.
- [x] La red de casa no tiene IPv6 global (solo `fe80::`), así que nadie se salta AdGuard por un DNS IPv6 del router.
- [x] iPhone con datos y Tailscale: las consultas llegan a AdGuard con su IP `100.x` y se bloquean dominios.
- [x] Jellyfin sigue funcionando desde el iPhone por Tailscale tras `--snat-subnet-routes=false`.
- [ ] Tras un reinicio: las reglas vuelven y AdGuard arranca. En la sesión en persona.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| El contenedor se reinicia sin parar: `exec … operation not permitted` | El binario lleva `setcap cap_net_bind_service` y `cap_drop: ALL` quita esa capacidad | `cap_add: NET_BIND_SERVICE` |
| Primer arranque: avisa de que debe ejecutarse como administrador | Comprobación del primer arranque (root o capacidad ambiental) | Asistente como root y después volver a `adguard` (4.3) |
| "Probar servidores de subida" falla | La regla de `DOCKER-USER` del 53 cazaba la salida del contenedor | `--ctstate DNAT` en las reglas (4.2) |
| No bloquea nada: la lista de filtros sale vacía | Los filtros no se descargaron en el primer arranque; causa probable, el mismo bloqueo de la salida | `sudo docker restart adguard` |
| Un dispositivo no aparece en el panel | No ha renovado la IP | Apagar y encender su Wi-Fi |
| Algún anuncio se cuela | El dispositivo usó el DNS de respaldo, o el anuncio sale del mismo dominio que el contenido | Esperado con el respaldo (ADR 0010) |
| Safari en el iPhone no pasa por AdGuard | *Relay privado de iCloud* resuelve por su cuenta | Esperado; se puede desactivar en el iPhone |
| Las consultas de la tailnet salen como `172.19.0.1` | NAT de Tailscale al reenviar al contenedor | `tailscale set --snat-subnet-routes=false` (4.7) |

## 7. Cómo revertirlo

Por orden, de fuera hacia dentro:
1. **Tailscale:** desactiva **Override DNS servers** y borra el nameserver. Si quieres, quita `udp:53` y `tcp:53` de la política. Para el NAT: `sudo tailscale set --snat-subnet-routes=true`.
2. **Router:** en el pool de casa, los DNS que había (los del operador). Los dispositivos vuelven a ellos al renovar la IP.
3. **Servidor:**
   ```bash
   sudo nmcli con mod "Wired connection 1" ipv4.ignore-auto-dns no ipv4.dns "" ipv6.ignore-auto-dns no
   ```
   ```bash
   sudo nmcli device reapply eno1
   ```
4. **AdGuard:**
   ```bash
   sudo docker compose -f /srv/adguard/compose.yaml down
   ```
   ```bash
   sudo systemctl disable --now homelab-adguard-lan.service
   ```
   ```bash
   sudo rm /etc/systemd/system/homelab-adguard-lan.service
   ```
   `/srv/adguard` no se borra solo.

## 8. Seguridad

**Mitiga:**
- Anuncios, rastreo y dominios de malware conocidos en todos los dispositivos, sin instalar nada en ellos.
- El operador ya no ve los nombres que se consultan: AdGuard pregunta por DoH, y DNSSEC detecta respuestas alteradas.
- Contenedor comprometido: usuario sin privilegios, sin capacidades útiles y `no-new-privileges`. El panel solo es accesible desde la LAN; el DNS, desde la LAN y la tailnet.

**Queda:**
- El panel va por HTTP en la LAN: la contraseña viaja en claro hasta que haya HTTPS.
- Con el DNS de respaldo, parte de las consultas pueden saltarse el filtro. Lo eligió el dueño para no quedarse sin internet si el servidor falla ([ADR 0010](decisiones/0010-dns-de-casa-con-adguard.md)).
- Fuera de casa no hay respaldo: si el servidor se cae, un dispositivo con Tailscale conectado se queda sin DNS hasta desconectar Tailscale.
- El bloqueo por DNS no es un control de seguridad fuerte: un dispositivo puede usar su propio DNS (DoH en el navegador, Relay privado).
- La imagen está fijada: las actualizaciones son manuales.

## 9. Para la entrevista

**En 30 segundos:** "Monté AdGuard Home en Docker como DNS de toda la casa y de la tailnet. El contenedor corre con un usuario sin privilegios y escucha en puertos altos; Docker publica el 53. Dos cosas me obligaron a entender el detalle: la imagen tiene un binario con `setcap`, que no arranca con `cap_drop: ALL` sin devolverle esa capacidad, y la regla de firewall del puerto 53 en `DOCKER-USER` también bloqueaba la salida del contenedor hasta que la limité a las conexiones con DNAT. El servidor no usa su propio AdGuard: así no depende de un contenedor para arrancar y no rompe el filtro de salida del agente de IA."

**Preguntas posibles:**
- *¿Por qué `--ctstate DNAT`?* En `FORWARD` pasa tanto lo que entra al contenedor como lo que sale de él. El puerto original 53 coincide en los dos casos; `DNAT` distingue lo que entró por un puerto publicado.
- *¿Por qué no poner solo AdGuard como DNS?* Si el servidor se apaga, la casa se queda sin internet. El respaldo cambia algo de filtrado por disponibilidad.
- *¿Por qué no un público de respaldo también en Tailscale?* Tailscale consulta todos sus nameservers en paralelo: el público respondería a menudo y anularía el filtro.
- *¿Por qué veía AdGuard la IP de Docker en la tailnet?* Tailscale hace NAT a lo que reenvía desde `tailscale0`; el tráfico hacia un contenedor se reenvía. `--snat-subnet-routes=false` lo quita.

## Fuentes

- [AdGuard Home: Docker](https://github.com/AdguardTeam/AdGuardHome/wiki/Docker), [configuración](https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration) y [notas de las versiones](https://github.com/AdguardTeam/AdGuardHome/releases).
- `internal/home/home.go` de AdGuard Home (`checkNetworkPermissions`): la comprobación del primer arranque.
- [Tailscale: DNS](https://tailscale.com/kb/1054/dns) y [subnet routers](https://tailscale.com/kb/1019/subnets) (`--snat-subnet-routes`).
- [Docker: Packet filtering and firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/).
- [Quad9](https://quad9.net/) y [Cloudflare DNS](https://developers.cloudflare.com/1.1.1.1/).
