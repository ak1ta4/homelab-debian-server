# Conceptos previos

Este documento explica, sin dar nada por sabido, las ideas que aparecen en las guías. Se amplía en cada fase: DNS y HTTPS llegarán con la Fase 4, Docker con la Fase 1 (paso 8) y las VPN con la Fase 3.

## Red de casa

### IP privada e IP pública
Cada equipo de casa tiene una **IP privada** (`<IP_SERVIDOR>`, `<IP_PC>`…) que solo tiene sentido dentro de casa, como el número de una habitación dentro de un edificio. Hacia internet, toda la casa sale con **una sola IP pública** (`<IP_PUBLICA>`), que es la dirección del edificio.

### Router
El aparato que une la red de casa con internet. Hace de portero: reparte IP dentro de casa (DHCP) y traduce entre las privadas y la pública (NAT).

### DHCP y reserva DHCP
**DHCP** es el servicio del router que da una IP a cada equipo cuando se conecta. Por defecto la "presta" durante un tiempo y puede cambiar. Una **reserva DHCP** le dice al router: "a este equipo, identificado por su MAC, dale siempre esta IP". Así el servidor y el PC tienen siempre la misma dirección.

### Dirección MAC
Identificador de la tarjeta de red de cada equipo: seis pares de caracteres hexadecimales separados por dos puntos o guiones (`xx:xx:xx:xx:xx:xx`). El router la usa para reconocer a quién aplicar una reserva. Algunos móviles usan una MAC **aleatoria** por red para proteger la privacidad; en esos equipos una reserva puede no funcionar.

### NAT, reenvío de puertos, DMZ y UPnP
- **NAT:** el router traduce las conexiones que salen de casa para que usen la IP pública. Las conexiones que *llegan* desde internet no entran solas: el router no sabe a qué equipo mandarlas y las descarta.
- **Reenvío de puertos** (*port mapping*): regla que abre esa puerta a propósito: "lo que llegue al puerto X, mándalo al equipo Y". Es la forma habitual de exponer un servicio a internet. En este homelab **no se usa**, con una excepción documentada que sirve justo para lo contrario (tapar el SSH del propio router).
- **DMZ:** reenvía *todos* los puertos a un equipo. Nunca debe estar activa.
- **UPnP:** permite que los programas de casa abran puertos en el router sin preguntar. Se deja desactivado.

### CGNAT
Algunas operadoras no dan una IP pública propia a cada casa, sino que comparten una entre muchas (*Carrier-Grade NAT*). Se detecta si la IP de la WAN del router está en `100.64.0.0/10` o no coincide con la IP pública que ven las webs. Afecta a cómo se conecta Tailscale (Fase 3).

### Puerto
Número que identifica un servicio dentro de un equipo (22 para SSH, 443 para HTTPS…). Si la IP es la dirección del edificio, el puerto es la puerta. "Puerto abierto" significa que algo escucha detrás de esa puerta.

### Abierto en la LAN frente a accesible desde internet
Que un puerto esté abierto en el servidor solo significa que los equipos de casa pueden llegar a él. Para que llegue alguien de internet, además tiene que dejarlo pasar el router. Por eso la auditoría distingue los dos casos, y la prueba "desde fuera" se hace con el móvil **con el Wi-Fi apagado**, usando datos móviles, como lo haría un desconocido.

## Acceso remoto

### SSH
Programa para manejar otro equipo con una terminal, con todo el tráfico cifrado. Hay un **servidor** (`sshd`, en el homelab) y un **cliente** (`ssh`, en el PC).

### Clave pública y clave privada
En vez de una contraseña, SSH puede usar un par de claves:
- La **privada** se queda en el PC y nunca se comparte. Es la llave.
- La **pública** se copia al servidor, en `~/.ssh/authorized_keys`. Es el candado.

Solo quien tenga la llave abre ese candado. La privada se protege además con una **frase de paso**: si alguien copia el archivo, sin la frase no le sirve.

### Huella del servidor
La primera vez que te conectas, SSH enseña la **huella** del servidor (`SHA256:…`) y pregunta si te fías. Compararla con la real evita conectarte a un equipo que se haga pasar por el tuyo. Después la recuerda en `~/.ssh/known_hosts` y avisa si cambia.

### Segundo factor: TOTP
Un código de 6 cifras que cambia cada 30 segundos y genera una app del móvil a partir de un secreto compartido con el servidor. Para entrar hacen falta **algo que tienes** (la clave del PC) y **otro algo que tienes** (el móvil). Si te roban uno, el otro sigue protegiendo.

### PAM
Sistema de Linux que decide *cómo* se autentica cada servicio: con contraseña, con un código, con ambos… Cada servicio tiene su archivo en `/etc/pam.d/`. Aquí se usa para que SSH pida el código TOTP y nunca la contraseña de Unix.

### Fail2ban
Programa que lee los registros del sistema y, si una IP falla al entrar varias veces seguidas, la bloquea un tiempo en el firewall. No impide un ataque con la clave correcta; reduce el ruido y frena los intentos a ciegas.

### Firewall
Filtro de tráfico del propio equipo: decide qué conexiones entrantes acepta. Hoy el servidor no tiene uno activo; llegará en la Fase 1 (paso 7). Fail2ban ya usa el filtro del núcleo (nftables) para sus bloqueos.

## Git y el repositorio

### Repositorio, commit y rama
Un **repositorio** guarda los archivos y todo su historial. Cada **commit** es una foto de un cambio, con autor, fecha y mensaje. Una **rama** es una línea de trabajo separada que luego se une a la principal (`main`).

### Parche
Un archivo de texto con las diferencias de un commit. Aquí el agente de IA entrega su trabajo como parches; la persona los revisa y los aplica con `git am`.

### Remoto y clave de despliegue
El **remoto** (`origin`) es la copia del repositorio en GitHub. La **clave de despliegue** es una clave SSH que solo da acceso a *ese* repositorio, no a la cuenta entera.

### Hook
Script que Git ejecuta solo en ciertos momentos: antes de un commit (`pre-commit`), antes de aplicar un parche (`pre-applypatch`) o antes de subir (`pre-push`). Aquí lanzan gitleaks.

### Secreto y dato identificativo
- **Secreto:** algo que da acceso (contraseña, token, clave privada, secreto TOTP).
- **Dato identificativo:** algo que no da acceso pero ayuda a localizarte o atacarte (IP pública, MAC, nombres de usuario, dominio).

Ninguno de los dos entra en el repositorio. Si un secreto llegara a subirse, borrarlo del historial no basta: hay que **cambiarlo** (rotarlo).
