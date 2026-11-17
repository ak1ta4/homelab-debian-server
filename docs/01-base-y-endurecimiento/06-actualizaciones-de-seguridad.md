# Paso 6: actualizaciones de seguridad automáticas

> **Versiones usadas:** Debian 13.7, unattended-upgrades 2.12, GDM 48.0, GNOME Shell 48.7.

## 1. Qué vamos a hacer y por qué

**En sencillo:** cada día, el servidor mira si Debian ha publicado arreglos de seguridad y los instala solo. Si alguno necesita reiniciar, se reinicia sobre las 6 de la mañana. Al volver, entra solo en la sesión, para que el escritorio remoto funcione, y bloquea la pantalla al instante.

**Técnicamente:**
- `unattended-upgrades` activado con `20auto-upgrades`, con los orígenes por defecto de Debian.
- Reinicio automático justo después de actualizar, solo si existe `/run/reboot-required`. El temporizador `apt-daily-upgrade` lo lanza entre las 6:00 y las 7:00.
- GNOME Software deja de descargar e instalar por su cuenta.
- Inicio de sesión automático en GDM, más un programa de inicio que bloquea la pantalla.
- Decisión y riesgos: [ADR 0006](../decisiones/0006-reinicio-e-inicio-de-sesion-automaticos.md).

## 2. Conceptos previos

- **`unattended-upgrades`:** programa de Debian que instala actualizaciones sin preguntar, solo de los orígenes permitidos.
- **`/run/reboot-required`:** archivo que crea, por ejemplo, la instalación de un núcleo nuevo (`/etc/kernel/postinst.d/unattended-upgrades`). Indica que hace falta reiniciar.
- **Temporizadores de apt:** `apt-daily.timer` actualiza la lista de paquetes; `apt-daily-upgrade.timer` instala, a las 6:00 con hasta 60 minutos de retraso aleatorio.
- **GDM:** la pantalla de inicio de sesión de GNOME.

## 3. Requisitos y estado inicial

- Repositorio de seguridad configurado (`trixie-security` en `/etc/apt/sources.list`).
- Estado inicial: sin `unattended-upgrades`. Las actualizaciones llegaban por GNOME Software (PackageKit), que instala el núcleo en un arranque especial: por eso un reinicio con actualización da dos arranques seguidos.
- Sunshine como servicio de usuario habilitado: `systemctl --user is-enabled app-dev.lizardbyte.app.Sunshine` → `enabled`.

## 4. Pasos

En el servidor, con `<USUARIO_ADMIN>`. Pega los comandos **de uno en uno**.

### 4.1 Instalar

```bash
sudo apt install unattended-upgrades
```

Configuración por defecto del paquete en `/etc/apt/apt.conf.d/50unattended-upgrades`, que no se toca. Orígenes activos:
- `origin=Debian,codename=trixie,label=Debian`: las versiones de la estable (13.x).
- `origin=Debian,codename=trixie,label=Debian-Security` y `origin=Debian,codename=trixie-security,label=Debian-Security`: seguridad.

Los repositorios externos, como el de Claude Code, **no** se actualizan solos: el agente no cambia sin revisión. Por defecto también elimina los núcleos viejos y las dependencias que quedan sin uso tras actualizar.

### 4.2 Activar y configurar el reinicio

Dos archivos (`sudo nano`):

`/etc/apt/apt.conf.d/20auto-upgrades`, lo mismo que escribe `dpkg-reconfigure unattended-upgrades`:
```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

`/etc/apt/apt.conf.d/52homelab-unattended-upgrades`, que se lee después del archivo del paquete y manda sobre él:
```
// Homelab, Fase 1, paso 6. El archivo del paquete (50unattended-upgrades)
// no se toca: lo de aquí se lee después y manda.
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
Unattended-Upgrade::Automatic-Reboot-Time "now";
```
- `WithUsers "true"`: hace falta porque, con el inicio automático, siempre hay una sesión abierta.
- `Time "now"`: reinicia al terminar, es decir, entre las 6:00 y las 7:00. Una hora fija, como `"06:00"`, puede pasar al día siguiente si la actualización termina después.

**Prueba en seco:**
```bash
sudo unattended-upgrade --dry-run --debug
```
**Resultado:** `Allowed origins are:` con los tres orígenes de Debian, y nada instalado.

### 4.3 GNOME Software, sin descargas automáticas

Sin `sudo`, porque es un ajuste de tu usuario:
```bash
gsettings set org.gnome.software download-updates false
```
```bash
gsettings get org.gnome.software download-updates
```
**Resultado:** `false`. GNOME Software sigue sirviendo para instalar a mano.

### 4.4 Bloqueo al entrar

Primero, comprueba que el bloqueo funciona (bloquea la pantalla de verdad):
```bash
loginctl lock-session
```

Sin `sudo`:
```bash
mkdir -p ~/.config/autostart
```
```bash
nano ~/.config/autostart/bloquear-al-entrar.desktop
```
```
[Desktop Entry]
Type=Application
Name=Bloquear al entrar
Comment=Homelab: bloquea la pantalla tras el inicio automático
Exec=sh -c "sleep 3; loginctl lock-session"
NoDisplay=true
X-GNOME-Autostart-enabled=true
```

### 4.5 Inicio de sesión automático

```bash
sudo cp -a /etc/gdm3/daemon.conf /etc/gdm3/daemon.conf.bak-AAAAMMDD
```
```bash
sudo nano /etc/gdm3/daemon.conf
```
En la sección `[daemon]`, quita el `#` de las dos líneas de ejemplo y pon tu usuario:
```
AutomaticLoginEnable = true
AutomaticLogin = <USUARIO_ADMIN>
```
```bash
diff /etc/gdm3/daemon.conf.bak-AAAAMMDD /etc/gdm3/daemon.conf
```
**Resultado:** Solo esas dos líneas. **No reinicies GDM**, porque cerraría la sesión gráfica: el cambio se aplica en el próximo arranque.

## 5. Verificación

- [x] `apt-config dump` muestra `APT::Periodic::Unattended-Upgrade "1"` y las tres opciones de reinicio.
- [x] La prueba en seco acepta solo los orígenes de Debian.
- [x] `apt-daily-upgrade.timer` tiene programada la próxima ejecución.
- [x] `gsettings` → `false`.
- [x] `loginctl lock-session` bloquea la sesión.
- [x] `daemon.conf`: solo las dos líneas cambiadas, de `root` y con permisos 644.
- [ ] La primera ejecución real: `/var/log/unattended-upgrades/unattended-upgrades.log` al día siguiente.
- [ ] Al reiniciar: entra solo, bloquea la pantalla y Moonlight muestra la pantalla de bloqueo y deja entrar. En la sesión en persona.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| No actualiza nada nunca | Falta `20auto-upgrades` | `apt-config dump \| grep Periodic` |
| Se reinicia a media mañana | El equipo estaba apagado a las 6 y el temporizador es `Persistent=true`: recupera la ejecución perdida al arrancar | Es lo esperado. Para evitarlo, `Automatic-Reboot-Time` a una hora fija |
| Tras reiniciar, Moonlight no conecta | La sesión no ha entrado sola, o Sunshine no ha arrancado | Por SSH: `loginctl list-sessions` y, en la sesión, `systemctl --user status app-dev.lizardbyte.app.Sunshine` |
| El escritorio queda sin bloquear tras el arranque | El programa de inicio no se ejecutó | Comprueba el archivo `.desktop` y prueba `loginctl lock-session` a mano |
| Una aplicación pide la contraseña del "llavero" | Con el inicio automático, el llavero no se abre solo | Es lo esperado: escríbela una vez |

## 7. Cómo revertirlo

```bash
sudo rm /etc/apt/apt.conf.d/20auto-upgrades
```
```bash
sudo rm /etc/apt/apt.conf.d/52homelab-unattended-upgrades
```
```bash
sudo cp -a /etc/gdm3/daemon.conf.bak-AAAAMMDD /etc/gdm3/daemon.conf
```
```bash
rm ~/.config/autostart/bloquear-al-entrar.desktop
```
```bash
gsettings reset org.gnome.software download-updates
```

## 8. Seguridad

**Mitiga:**
- Vulnerabilidades conocidas sin parchear: los arreglos de seguridad se aplican en menos de un día.
- Núcleo parcheado pero sin usar: reinicio automático cuando hace falta.
- Dos sistemas de actualización a la vez: GNOME Software ya no descarga por su cuenta.

**Queda:**
- Unos segundos de escritorio sin bloquear en cada arranque, con el portátil delante ([ADR 0006](../decisiones/0006-reinicio-e-inicio-de-sesion-automaticos.md)).
- Una actualización puede romper algo y nadie se entera hasta mirar el registro: los avisos por correo llegan en la Fase 5.
- Cada reinicio cierra la sesión de tmux del agente.
- Los repositorios externos (Claude Code) y el firmware (`fwupd`) se actualizan a mano.

## 9. Para la entrevista

**En 30 segundos:** "Activé `unattended-upgrades` con los orígenes por defecto de Debian, sin tocar el archivo del paquete: mis cambios van en un archivo propio que se lee después. Reinicia solo cuando existe `/run/reboot-required`, y como el temporizador corre entre las 6 y las 7, uso `Automatic-Reboot-Time "now"` en lugar de una hora fija. Para que el escritorio remoto vuelva tras el reinicio, activé el inicio de sesión automático con un bloqueo inmediato de la pantalla, y documenté el riesgo en una ADR."

**Preguntas posibles:**
- *¿Por qué no actualizar también los repositorios externos?* Porque cambian sin el control de calidad de Debian. En este caso es el propio agente de IA: sus cambios se revisan.
- *¿No es peligroso reiniciar solo?* Lo es si nadie vuelve a montar lo necesario. Por eso se cubre lo que depende de una sesión (inicio automático) y lo que depende de los discos (`nofail`). La alternativa, núcleos parcheados sin usar durante semanas, es peor.
- *¿Por qué no `"06:00"` como hora de reinicio?* Porque la actualización empieza a una hora aleatoria entre las 6 y las 7, y una hora ya pasada puede programar el reinicio para el día siguiente.
- *¿El inicio automático no anula la contraseña?* No del todo: la pantalla se bloquea en segundos. El riesgo físico restante es pequeño frente a un disco sin cifrar, que ya está aceptado.

## Fuentes

- `/usr/share/doc/unattended-upgrades/README.md.gz` y `/etc/apt/apt.conf.d/50unattended-upgrades` de unattended-upgrades 2.12.
- [Debian Wiki: UnattendedUpgrades](https://wiki.debian.org/UnattendedUpgrades).
- `systemctl cat apt-daily-upgrade.timer` de apt en Debian 13.
- `/etc/gdm3/daemon.conf` de GDM 48 y `man loginctl` de systemd 257.
- [How To Secure A Linux Server: Automatic Security Updates and Alerts](https://github.com/imthenachoman/How-To-Secure-A-Linux-Server) (CC BY-SA 4.0), adaptada: ver [`../guia-securizacion.md`](../guia-securizacion.md).
