# Paso 7: firewall de entrada (UFW) y filtro de salida del agente

> **Versiones usadas:** Debian 13.7, ufw 0.36.2-9, iptables 1.8.11 (backend nft), nftables 1.1.3.

## 1. Qué vamos a hacer y por qué

**En sencillo:** ponemos un portero en la puerta del servidor. Solo deja pasar, y solo a gente de casa, lo que usamos: la puerta de emergencia (SSH) y el escritorio remoto (Sunshine). Además, al ayudante de IA le ponemos su propio portero de salida: puede llamar a internet por la vía segura (HTTPS), pero no puede tocar nada de la red de casa.

**Técnicamente:**
- **Parte A:** UFW con entrada denegada por defecto. Se permiten SSH, Sunshine y mDNS desde `<RED_LAN>`. Activación con desactivación automática a los 5 minutos.
- **Parte B:** tabla nftables propia que filtra solo los paquetes de `homelab-agent`: sin LAN ni servicios locales, y a internet solo DNS y HTTPS ([ADR 0007](../decisiones/0007-filtro-de-salida-del-agente.md)).

## 2. Conceptos previos

- **Firewall de entrada:** decide qué conexiones nuevas puede recibir el equipo.
- **Filtro de salida:** decide a dónde puede conectarse un programa o un usuario.
- **UFW:** una interfaz sencilla sobre el firewall del núcleo. En Debian 13 usa `iptables-nft`, que escribe en nftables.
- **nftables:** el firewall del núcleo. Varias tablas conviven: un paquete rechazado en cualquiera de ellas no pasa.
- **`meta skuid`:** en nftables, el usuario dueño del socket que envía el paquete.

## 3. Requisitos y estado inicial

- SSH de emergencia funcionando ([paso 2](02-ssh-de-emergencia.md)) y Moonlight como segundo acceso.
- Estado inicial (`ss -tulnH`):
  - Abiertos a todas las redes: SSH (22), Sunshine TCP 47984, 47989, 47990 (panel) y 48010, Sunshine UDP 47998 a 48000, y mDNS (5353).
  - Solo en el propio equipo: CUPS (631).
  - Sin firewall: `ufw` sin instalar y `nftables.service` inactivo. Fail2ban tiene su propia tabla.

## 4. Pasos

En el servidor, con `<USUARIO_ADMIN>`. Pega los comandos **de uno en uno**.

### Parte A: firewall de entrada

#### A.1 Instalar (no activa nada)

```bash
sudo apt install ufw
```
Queda con `ENABLED=no` en `/etc/ufw/ufw.conf`. Por defecto bloquea la entrada y permite la salida (`/etc/default/ufw`).

#### A.2 Reglas, con UFW apagado

Con UFW apagado, las reglas solo se guardan.
```bash
sudo ufw default deny incoming
```
```bash
sudo ufw default allow outgoing
```
```bash
sudo ufw allow proto tcp from <RED_LAN> to any port 22 comment 'SSH desde casa'
```
```bash
sudo ufw allow proto tcp from <RED_LAN> to any port 47984,47989,47990,48010 comment 'Sunshine TCP'
```
```bash
sudo ufw allow proto udp from <RED_LAN> to any port 47998:48000 comment 'Sunshine UDP'
```
```bash
sudo ufw allow proto udp from <RED_LAN> to any port 5353 comment 'mDNS'
```
```bash
sudo ufw show added
```
- Las reglas no se limitan a una interfaz: así, si un día el servidor va por Wi-Fi, SSH sigue entrando desde casa.
- El panel de Sunshine (47990) queda abierto a toda la LAN, por decisión del dueño. Sigue pidiendo su usuario y contraseña.

#### A.3 Activar con desactivación automática

```bash
sudo systemd-run --on-active=5m --unit=ufw-rescate /usr/sbin/ufw disable
```
```bash
sudo ufw enable
```
**En los 5 minutos siguientes**, desde el PC: una conexión SSH **nueva** y una conexión por Moonlight. **Solo si las dos funcionan:**
```bash
sudo systemctl stop ufw-rescate.timer
```
Si algo falla, no toques nada: a los 5 minutos UFW se apaga solo.

```bash
sudo ufw status verbose
```
**Resultado:** `Status: active`, `deny (incoming)` y las cuatro reglas.

### Parte B: filtro de salida del agente

#### B.1 Reglas

`/etc/homelab/agente-salida.nft`, de root con permisos 644:
```
#!/usr/sbin/nft -f
# Homelab, Fase 1, paso 7: filtro de salida del agente de IA (homelab-agent).
# Tabla propia: no toca las reglas de UFW ni las de Fail2ban.
# Solo afecta a los paquetes de procesos de homelab-agent; el resto pasa sin cambios.
# Permite: DNS a servidores públicos y HTTPS (443) a internet.
# Bloquea: la red de casa, el propio equipo (lo), rangos privados y cualquier otro puerto.

# Crear y borrar primero: así se puede cargar varias veces sin duplicar reglas.
table inet agente_salida
delete table inet agente_salida

table inet agente_salida {
	set privadas4 {
		type ipv4_addr
		flags interval
		elements = { 10.0.0.0/8, 100.64.0.0/10, 169.254.0.0/16,
			     172.16.0.0/12, <RANGO_PRIVADO_16>, 224.0.0.0/4 }
	}

	set privadas6 {
		type ipv6_addr
		flags interval
		elements = { fc00::/7, fe80::/10, ff00::/8 }
	}

	chain salida {
		type filter hook output priority filter; policy accept;
		# Solo los paquetes con dueño homelab-agent entran en el filtro.
		meta skuid "homelab-agent" jump agente
	}

	chain agente {
		ct state established,related accept
		# Nada hacia el propio equipo: ni el panel de Sunshine ni otros servicios locales.
		oif "lo" reject
		# Nada hacia la red de casa ni otros rangos privados.
		ip daddr @privadas4 reject
		ip6 daddr @privadas6 reject
		# DNS (a servidores públicos: los privados ya se han rechazado arriba).
		meta l4proto { tcp, udp } th dport 53 accept
		# HTTPS a internet.
		tcp dport 443 accept
		# Todo lo demás, rechazado.
		reject
	}
}
```
- `<RANGO_PRIVADO_16>` es el tercer rango privado de la RFC 1918, el `/16` (ver el [glosario](../glosario.md)). En el archivo real va escrito tal cual.
- La cadena principal **solo salta** a la del agente si el paquete es suyo. Los paquetes del núcleo sin dueño, como las respuestas a un `ping`, no se tocan.
- **No uses `/etc/nftables.conf`**: empieza con `flush ruleset` y borraría las reglas de UFW y Fail2ban.

```bash
sudo nft -c -f /etc/homelab/agente-salida.nft
```
**Resultado:** Sin salida: la sintaxis es correcta.

#### B.2 Cargarlo en cada arranque

`/etc/systemd/system/homelab-agente-salida.service`, de root con permisos 644:
```
[Unit]
Description=Homelab: filtro de salida del agente de IA
Documentation=file:/etc/homelab/agente-salida.nft
# Cargar antes de que haya red, para que el agente nunca salga sin filtro.
Before=network-pre.target
Wants=network-pre.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/nft -f /etc/homelab/agente-salida.nft
ExecStop=/usr/sbin/nft delete table inet agente_salida

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
```

#### B.3 Activar con desactivación automática

```bash
sudo systemd-run --on-active=5m --unit=agente-rescate /usr/sbin/nft delete table inet agente_salida
```
```bash
sudo systemctl enable --now homelab-agente-salida
```
Pruebas, **como `homelab-agent`**, en los 5 minutos siguientes (sección 5). Si todo está bien:
```bash
sudo systemctl stop agente-rescate.timer
```

## 5. Verificación

- [x] `ufw status verbose`: activo, entrada denegada por defecto y las cuatro reglas.
- [x] Con UFW activo: SSH nuevo desde el PC y Moonlight funcionan.
- [x] UFW habilitado en el arranque (`ENABLED=yes`).
- [x] Filtro del agente, probado como `homelab-agent`:
  - `getent hosts deb.debian.org` resuelve.
  - `curl https://deb.debian.org` y `curl https://api.anthropic.com` responden.
  - Rechazados al instante: `curl http://deb.debian.org` (80), el router (443 y 22), el panel de Sunshine en `127.0.0.1:47990` y por la IP de la LAN, y el SSH local.
- [x] `homelab-agente-salida` activo y habilitado; sus archivos son de root.
- [ ] Tras un reinicio: UFW y el filtro activos. En la sesión en persona.
- [ ] Desde fuera de casa no se llega a ningún puerto: escaneo externo en la Fase 8.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `ufw enable` no pregunta "may disrupt existing ssh connections" | Solo pregunta si lo ejecutas desde una sesión SSH | Normal desde el escritorio |
| Se pegan todos los comandos y se cancela la protección antes de probar | Copiar un bloque entero | Primero las pruebas, y solo después el `stop` del temporizador |
| UFW se apaga solo a los 5 minutos | El temporizador de rescate no se canceló a tiempo | Es la protección funcionando: repite A.3 |
| `nft -c` da `Operation not permitted` | Hace falta root incluso para validar | Con `sudo` |
| El agente falla con "connection refused" hacia un servicio | El filtro lo rechaza, como debe | Decidir si se permite y documentarlo en la ADR 0007 |

## 7. Cómo revertirlo

```bash
sudo ufw disable
```
```bash
sudo systemctl disable --now homelab-agente-salida
```
Para borrar también las reglas de UFW (con UFW apagado): `sudo ufw reset`.

## 8. Seguridad

**Mitiga:**
- Servicios expuestos a toda la red: solo se puede entrar a lo necesario y desde la LAN. Lo que se instale en el futuro queda cerrado hasta abrirlo a propósito.
- El agente ya no puede sondear la red de casa, ni usar los paneles del router o de Sunshine, ni servicios locales. Tampoco puede salir por puertos poco vigilados.

**Queda:**
- Por HTTPS, el agente puede enviar a cualquier web lo que pueda leer ([ADR 0007](../decisiones/0007-filtro-de-salida-del-agente.md)).
- Cualquier aparato de casa llega a SSH y a Sunshine; lo protegen la clave + TOTP y el usuario de Sunshine.
- Docker publica puertos saltándose las reglas de UFW: se trata en el paso 8.
- La salida del resto del sistema está abierta.
- En la Fase 4, un DNS local (AdGuard) no será alcanzable para el agente sin cambiar el filtro.

## 9. Para la entrevista

**En 30 segundos:** "Activé UFW con entrada denegada por defecto, permitiendo SSH y Sunshine solo desde la LAN, y lo encendí con un `systemd-run` que lo apagaba a los cinco minutos si no lo cancelaba tras probar SSH y Moonlight. Para el agente de IA hice un filtro de salida con una tabla nftables propia que solo mira los paquetes de su usuario con `meta skuid`: sin LAN ni servicios locales, y a internet solo DNS y HTTPS. Evité `/etc/nftables.conf` porque su `flush ruleset` borraría UFW y Fail2ban."

**Preguntas posibles:**
- *¿Por qué una tabla nftables y no reglas de UFW?* UFW no filtra por usuario en su sintaxis. Una tabla propia se carga y se quita sin tocar UFW, y la mantiene su propio servicio.
- *¿Por qué `jump` en vez de filtrar todo en una cadena?* Los paquetes que genera el núcleo sin dueño, como un RST o la respuesta a un `ping`, no tienen `skuid`. Con `jump` solo se filtra lo del agente; con un `reject` final para todos se rechazarían esos paquetes.
- *¿Por qué rechazar y no descartar en silencio?* Para el agente, un "connection refused" inmediato es más claro que esperar a un tiempo agotado, y en local no hay que esconder nada.
- *¿El filtro impide que el agente saque datos?* No del todo: HTTPS sigue abierto. Lo reduce a un canal y deja fuera la red de casa. Para cerrarlo haría falta una lista de dominios.

## Fuentes

- `man ufw` y `/etc/default/ufw` de ufw 0.36.2.
- `man nft` de nftables 1.1.3 (`meta skuid`, conjuntos con `flags interval`).
- `man systemd-run` y `man systemd.special` (`network-pre.target`) de systemd 257.
- [How To Secure A Linux Server: Firewall With UFW](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server) (CC BY-SA 4.0), adaptada: ver [`../guia-securizacion.md`](../guia-securizacion.md).
