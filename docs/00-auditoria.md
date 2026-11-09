# Fase 0: auditoría inicial (solo lectura)

> **Específico de este equipo:** portátil HP Victus 16 con Ryzen 5 5600H (gráfica AMD integrada) y RTX 3050, Debian 13 con GNOME, conectado por cable al router de la operadora. Los nombres de menú del router son los de su portal avanzado; en otro router cambiarán.

## 1. Qué vamos a hacer y por qué

**En sencillo:** antes de reformar una casa se mira cómo está: qué paredes son maestras, por dónde van los cables, qué puertas dan a la calle. La Fase 0 es esa inspección. **No se cambia nada**; solo se mira y se apunta.

**Técnicamente:** inventario del sistema, discos, gráficas, red y accesos, y comprobación de qué está expuesto a la red de casa y qué a internet. El resultado es un informe con riesgos y preguntas, y el plan de la Fase 1.

## 2. Conceptos previos

IP privada y pública, router, NAT, reenvío de puertos, DMZ, UPnP, CGNAT, puerto, y la diferencia entre "abierto en la LAN" y "accesible desde internet": ver [`00-conceptos.md`](00-conceptos.md).

## 3. Requisitos y estado inicial

- Acceso al servidor con un usuario con `sudo` (al principio solo había escritorio remoto con Moonlight).
- Acceso de administrador al router, desde un navegador de casa.
- Un móvil con datos móviles para la prueba "desde fuera".
- Nada se instala: lo que haga falta (como `smartctl`) se usa sin instalar.

## 4. Pasos

Todos se ejecutan **en el servidor**, salvo los del router y el móvil. Los que llevan `sudo` piden la contraseña del administrador.

> **Nota de fidelidad:** la auditoría original la hizo una sesión de Claude Code que no conservó la lista literal de comandos. Estos son los comandos que reproducen cada dato del informe. Los que no llevan `sudo` se han ejecutado en este equipo (Debian 13.7); los de `sudo`, solo se han comprobado sus opciones.

### 4.1 Sistema

```bash
cat /etc/debian_version
```
Versión de Debian. Aquí: `13.x`.

```bash
uname -r
```
Versión del núcleo (kernel).

```bash
echo $XDG_SESSION_TYPE
```
Tipo de sesión gráfica, ejecutado desde la propia sesión: `wayland` o `x11`.

```bash
mokutil --sb-state
```
Estado de Secure Boot (`SecureBoot disabled` o `enabled`). Si no existe `mokutil`, mirar `bootctl status`.

```bash
ls -l /dev/tpm*
```
Si existe `/dev/tpm0`, hay un chip TPM visible. Aquí al principio no aparecía; tras activarlo en la BIOS, sí.

```bash
lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINTS,TYPE
```
Discos y particiones. Si alguna partición fuera `crypto_LUKS`, el disco estaría cifrado. Aquí ninguna lo está.

```bash
free -h
```
```bash
df -h /
```
Memoria y espacio libre.

```bash
systemctl --failed
```
Servicios que han fallado. Lo correcto: `0 loaded units listed`.

```bash
timedatectl
```
La línea `System clock synchronized: yes` importa: el TOTP y los certificados dependen de la hora.

### 4.2 Discos y SMART

Para no instalar `smartmontools`, se descarga el paquete y se usa el programa sin instalarlo:

```bash
cd "$(mktemp -d)"
```
```bash
apt-get download smartmontools
```
```bash
dpkg-deb -x smartmontools_*.deb x
```
```bash
sudo ./x/usr/sbin/smartctl -a /dev/nvme0n1
```
Salud del NVMe: desgaste (`Percentage Used`), errores de datos y temperatura.

```bash
sudo ./x/usr/sbin/smartctl -a -d sat /dev/sda
```
El SSD externo va por USB: `-d sat` le dice a `smartctl` cómo hablar con él a través del adaptador. Algunos datos (como el estado global) no llegan por USB; los atributos sí.

**No asumas que el disco de datos está vacío:** monta en solo lectura o mira su contenido antes de decidir nada. Aquí tenía cuatro carpetas vacías y metadatos de NTFS.

### 4.3 Gráficas

```bash
lspci -k | grep -A3 -E 'VGA|3D|Display'
```
Las dos GPU y el driver que usa cada una (`amdgpu` y `nvidia`).

```bash
ls -l /dev/dri/by-path/
```
Qué `card` y qué `renderD…` corresponde a cada dirección PCI. Cruzándolo con la salida de `lspci` (cada línea empieza por esa dirección, como `01:00.0`), se sabe qué GPU es cada una. Aquí: AMD = `card0` y `renderD128`; NVIDIA = `card1` y `renderD129`.

```bash
for c in /sys/class/drm/card*-*; do echo "$c $(cat $c/status)"; done
```
Qué salidas de vídeo hay conectadas y a qué tarjeta: la pantalla interna (`eDP`) va en la AMD y el HDMI en la NVIDIA.

Además: cómo está instalado Sunshine (`.deb`, Flatpak o AppImage; un Flatpak puede no admitir captura KMS) y buscar las notas o copias del intento anterior con NVIDIA. Ver [`lecciones-aprendidas.md`](lecciones-aprendidas.md#pantalla-negra-en-moonlight-con-nvidia).

### 4.4 Acceso y puertos

```bash
getent passwd | awk -F: '$7 !~ /(nologin|false|sync)$/ {print $1, $7}'
```
Usuarios con shell de inicio de sesión. Se excluye `sync`, una cuenta del sistema cuyo "shell" es `/bin/sync` y no permite entrar.

```bash
getent group sudo
```
Quién puede usar `sudo`.

```bash
sudo ss -tulpen
```
Qué escucha en cada puerto y qué programa lo abre. Fíjate en la dirección: `0.0.0.0` o `[::]` significa "en todas las redes"; `127.0.0.1` significa "solo en el propio equipo".

```bash
sudo nft list ruleset
```
Reglas del firewall del núcleo. Si no sale nada, no hay firewall.

Aquí, al principio: sin SSH, sin firewall, y Sunshine escuchando en todas las interfaces (TCP 47984, 47989, 47990 y 48010; UDP 47998 a 48000).

### 4.5 Red

```bash
ip -br address
```
```bash
ip route
```
IP de cada interfaz y puerta de enlace (`<IP_ROUTER>`).

```bash
cat /etc/resolv.conf
```
Servidores DNS en uso (líneas `nameserver`). En este Debian no hay `resolvectl`; si lo hubiera, `resolvectl status` da más detalle.

```bash
sudo ss -tulpn 'sport = :53'
```
Si algo ocupa ya el puerto 53 (importará para AdGuard en la Fase 4).

**IP pública:** abre en el navegador del propio servidor una web que la muestre, o compárala más tarde con la del router.

### 4.6 Portátil

```bash
cat /sys/class/power_supply/BAT*/cycle_count
```
```bash
ls /sys/class/power_supply/BAT*/ | grep -i charge_control
```
Ciclos de la batería y si el núcleo permite limitar la carga (`charge_control_end_threshold`). Aquí **no existe**: el límite habrá que buscarlo en la BIOS.

```bash
grep -E '^#?HandleLidSwitch' /etc/systemd/logind.conf
```
Qué hace la tapa. Las líneas empiezan por `#`, así que rige el valor por defecto que muestran: `suspend`. Cerrar la tapa **suspende el equipo y corta Moonlight**.

```bash
cat /sys/class/thermal/thermal_zone*/temp
```
Temperaturas en milésimas de grado (`55000` = 55 °C). `lm-sensors` no está instalado; se instalará en la Fase 5 para la monitorización.

### 4.7 Router: exposición a internet

En un navegador de casa, entra en el portal del router (`https://<IP_ROUTER>`), y en él, en la **configuración avanzada**. **No cambies nada en esta revisión.**

1. **WAN Setting → WAN Interface:** apunta la IP de la conexión a internet. Si coincide con la IP pública y no está entre `100.64.0.0` y `100.127.255.255`, **no hay CGNAT**.
2. **Network Setting → NAT → Port Mapping:** no debe haber reglas hacia el servidor ni hacia los puertos 22 o 47984-48010.
3. **Network Setting → NAT → DMZ:** desactivada.
4. **UPnP → UPnP Setting:** desactivado.
5. **El punto de acceso Wi-Fi:** en su panel o app, el modo debe ser *Access Point*, no router (evita una doble NAT).
6. **Si algo apunta al servidor, bórralo de inmediato.**

### 4.8 Prueba desde fuera

Con el móvil **con el Wi-Fi apagado** (datos móviles) y sin VPN:

- En el navegador: `https://<IP_PUBLICA>:47990`, `http://<IP_PUBLICA>:47989/serverinfo` y `https://<IP_PUBLICA>:47984/serverinfo` (los puertos TCP de Sunshine).
- **Correcto:** no cargan, error o tiempo agotado. **Mal:** sale una página, un XML o un aviso de certificado.

Esta prueba no cubre los puertos UDP ni sustituye al escaneo externo completo de la Fase 8.

### 4.9 Hallazgo: el router tenía su propio SSH abierto a internet

Un análisis de red desde el PC avisó de un puerto 22 abierto **en el router**. Así se comprobó y se cerró:

**Comprobar desde casa** (en el servidor):

```bash
timeout 3 bash -c '</dev/tcp/<IP_ROUTER>/22'; echo "codigo=$?"
```
`codigo=0`: el router acepta conexiones en el 22.

```bash
ssh-keyscan -t rsa <IP_ROUTER> | ssh-keygen -lf -
```
```bash
ssh-keyscan -t rsa <IP_PUBLICA> | ssh-keygen -lf -
```
Si las dos huellas coinciden, quien responde en la IP pública es **el propio router**, no un equipo de casa.

**Comprobar desde fuera** (móvil con datos móviles, app de SSH como Termius): conexión a `<IP_PUBLICA>`, puerto 22, **con el usuario ya escrito en la conexión** y **sin escribir nunca una contraseña**.
- Error o *timeout*: cerrado.
- Pide contraseña o enseña una huella: **expuesto**. Aquí pedía contraseña.

**Corrección.** El portal no tiene opción para apagar ese SSH. Sin llamar a la operadora, se tapó con una regla de **Port Mapping** que manda el puerto 22 de internet a una IP de casa **que no usa nadie** (`<IP_SUMIDERO>`), elegida fuera de todos los rangos DHCP del router y comprobada sin respuesta:

| Campo | Valor |
|---|---|
| Adm.State | Enable |
| Interface | la de internet (aquí `ip2`), **no** la de la LAN (`ip1`) |
| Protocol | TCP |
| Remote IP | `0.0.0.0` (cualquiera) |
| External port | 22 to 22 |
| Internal IP | `<IP_SUMIDERO>` |
| Internal port | 22 |

Después, la misma prueba desde fuera dio *"connection could not be established"*. Decisión y riesgos en [`decisiones/0004-tapar-ssh-del-router.md`](decisiones/0004-tapar-ssh-del-router.md).

## 5. Verificación

- Informe con cada apartado anterior y sus riesgos señalados.
- Router: reenvíos y DMZ vacíos, UPnP desactivado, sin CGNAT.
- Prueba exterior: los puertos TCP de Sunshine y el 22 no responden desde datos móviles.
- Nada cambiado en el servidor (comprobable: no se instaló ningún paquete en esas fechas).

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| Un antivirus avisa de "puerto 22 abierto en el router" | Analiza desde dentro de casa | No significa que esté abierto a internet: hay que probar desde fuera (4.8 y 4.9) |
| La app de SSH pide un usuario y no sabes si ha llegado al router | La app lo pide *antes* de conectar si la conexión no tiene usuario | Escribe el usuario en la configuración de la conexión y repite: solo cuenta si pide **contraseña** o enseña una **huella** |
| "Desde fuera" carga la página | El móvil seguía en el Wi-Fi de casa | Apaga el Wi-Fi y comprueba que se ve 4G/5G |
| El escáner de la IP pública desde casa da puertos abiertos | El router responde a su propia IP pública desde dentro | Las pruebas de exposición se hacen siempre desde fuera |

## 7. Cómo revertirlo

La auditoría no cambia nada. La única modificación posterior, la regla que tapa el SSH del router, se quita en **Port Mapping**: marcar la regla y pulsar **Del**.

## 8. Seguridad

- **Mitiga:** cambiar cosas a ciegas. Detectó tres riesgos reales antes de montar nada: el SSH del router expuesto a internet, la ausencia de firewall y la tapa que suspende el equipo.
- **Queda:**
  - El SSH del router sigue abierto **dentro** de casa; lo protege su contraseña.
  - La regla del sumidero depende de que nadie use `<IP_SUMIDERO>` y de que la operadora no la borre con una actualización remota. Se revisa en el calendario periódico.
  - Los puertos UDP no se probaron desde fuera.
  - El disco no está cifrado: riesgo físico aceptado por el dueño ([ADR 0001](decisiones/0001-sin-cifrado-de-disco.md)).

## 9. Para la entrevista

**En 30 segundos:** "Antes de tocar nada hice una auditoría de solo lectura: sistema, discos con SMART, gráficas, red y exposición. Distinguí lo abierto en la LAN de lo accesible desde internet probando desde datos móviles, y así encontré que el router de la operadora tenía su SSH abierto a internet. Como el panel no permitía cerrarlo, lo tapé con un reenvío a una IP sin uso y lo verifiqué desde fuera."

**Preguntas posibles:**
- *¿Por qué no basta con mirar los puertos del servidor?* Porque lo que decide si algo llega desde internet es el router. Un puerto abierto en el servidor puede no estar expuesto, y el router puede exponer cosas suyas, como pasó aquí.
- *¿Cómo sabes que quien respondía era el router y no otro equipo?* Comparando la huella SSH por la IP interna del router y por la pública: era la misma.
- *¿Qué riesgo tiene la regla del sumidero?* Si algún día un equipo recibe esa IP, su puerto 22 queda expuesto. Por eso está fuera de los rangos DHCP, documentada como "no asignar nunca" y se revisa periódicamente.
- *¿Por qué usar `smartctl` sin instalarlo?* Para cumplir que la Fase 0 sea de solo lectura: no cambia el sistema.
