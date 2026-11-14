# Paso 4: el portátil como servidor (tapa, suspensión y batería)

> **Versiones usadas:** Debian 13.7, systemd 257.13-1~deb13u1, UPower 1.90.9-1.

## 1. Qué vamos a hacer y por qué

**En sencillo:** un portátil está pensado para dormirse en cuanto lo cierras o lo dejas quieto, y eso en un servidor es un apagón. Le quitamos esa costumbre: cerrar la tapa no hace nada, nada puede dormirlo, y si se va la luz y la batería se acaba, se **apaga ordenadamente** en lugar de quedarse a medias.

**Técnicamente:**
- `systemd-sleep`: suspensión, hibernación y sus variantes desactivadas con un archivo parcial en `sleep.conf.d/`.
- `systemd-logind`: la tapa no hace nada (`ignore`) con batería, con cargador y con pantalla externa.
- UPower: con la batería crítica, `PowerOff` al 10 % en vez de `HybridSleep` al 2 %.
- BIOS: buscar un límite de carga y el encendido al volver la corriente.

## 2. Conceptos previos

- **Suspender:** apaga casi todo y guarda el estado en la RAM. Si el equipo suspende, no responde a la red: SSH y Moonlight se cortan.
- **Hibernar:** guarda el estado en el disco (swap) y se apaga.
- **logind:** el servicio de systemd que decide qué hacer con la tapa y los botones.
- **UPower:** el servicio que vigila la batería y ejecuta la acción crítica.
- **Archivo parcial (*drop-in*):** un `.conf` en una carpeta `*.conf.d/` que cambia solo lo que dice, sin tocar el archivo principal. Se revierte borrándolo.

## 3. Requisitos y estado inicial

- SSH de emergencia funcionando ([paso 2](02-ssh-de-emergencia.md)) y escritorio remoto como respaldo.
- Estado inicial, comprobado sin `sudo`:
  - `logind.conf` y `sleep.conf` sin cambios ni carpetas `.conf.d`: la tapa **suspende**.
  - `upower -d` → `critical-action: HybridSleep`, al 2 %.
  - `/sys/class/power_supply/BAT0/`: 128 ciclos, capacidad igual a la de fábrica y **sin** `charge_control_end_threshold`. El núcleo no puede limitar la carga.

## 4. Pasos

En el servidor, con `<USUARIO_ADMIN>`. Pega los comandos **de uno en uno**.

### 4.1 Que nada pueda suspender ni hibernar

```bash
sudo mkdir -p /etc/systemd/sleep.conf.d
```
```bash
sudo nano /etc/systemd/sleep.conf.d/10-homelab.conf
```

Contenido:
```
# Homelab, Fase 1, paso 4: el portátil es un servidor.
# Nada lo suspende ni lo hiberna: ni la tapa, ni GNOME, ni la batería.
# Revertir: borrar este archivo.
[Sleep]
AllowSuspend=no
AllowHibernation=no
AllowSuspendThenHibernate=no
AllowHybridSleep=no
```

No hace falta reiniciar nada: systemd lee este archivo en cada intento de suspender.

**Comprobar sin suspender** (sirve cualquier usuario):
```bash
busctl call org.freedesktop.login1 /org/freedesktop/login1 org.freedesktop.login1.Manager CanSuspend
```
✅ `s "no"`. Lo mismo con `CanHibernate`, `CanHybridSleep` y `CanSuspendThenHibernate`. **No** se prueba con `systemctl suspend`: si la configuración fallase, el equipo se dormiría y se perdería el acceso remoto.

### 4.2 Que la tapa no haga nada

```bash
sudo mkdir -p /etc/systemd/logind.conf.d
```
```bash
sudo nano /etc/systemd/logind.conf.d/10-homelab.conf
```

Contenido:
```
# Homelab, Fase 1, paso 4: cerrar la tapa no hace nada,
# con cargador, con batería o con pantalla externa.
# Revertir: borrar este archivo y recargar systemd-logind.
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

Las tres líneas hacen falta: logind usa `HandleLidSwitchDocked` si hay más de una pantalla conectada y `HandleLidSwitchExternalPower` con el cargador puesto.

```bash
sudo systemctl reload systemd-logind
```
`reload` relee la configuración sin cerrar sesiones. **No** uses `restart`.

**Comprobar:**
```bash
busctl get-property org.freedesktop.login1 /org/freedesktop/login1 org.freedesktop.login1.Manager HandleLidSwitch
```
✅ `s "ignore"`. Igual con `HandleLidSwitchExternalPower` y `HandleLidSwitchDocked`.

### 4.3 Apagado limpio con la batería al 10 %

UPower 1.90.9 **solo lee `/etc/UPower/UPower.conf`**: no admite archivos parciales (comprobado en su código fuente, `src/up-config.c`). Se edita el archivo con copia previa. Es un archivo de configuración del paquete: si una actualización trae otra versión, apt preguntará si conservar la tuya.

Dos detalles del código fuente (`src/up-daemon.c` y `src/linux/up-backend.c`):
- Los umbrales deben ir **de mayor a menor** (aviso > crítico > acción). Si no, UPower usa sus valores por defecto **sin avisar**.
- Si la acción configurada no está disponible, UPower prueba la siguiente: `HybridSleep` → `Hibernate` → `PowerOff`. Tras el paso 4.1 ya acababa en `PowerOff`, pero se deja escrito para no depender de ello.

```bash
sudo cp -a /etc/UPower/UPower.conf /etc/UPower/UPower.conf.bak-AAAAMMDD
```
```bash
sudo sed -i 's/^PercentageLow=20.0$/PercentageLow=25.0/' /etc/UPower/UPower.conf
```
```bash
sudo sed -i 's/^PercentageCritical=5.0$/PercentageCritical=15.0/' /etc/UPower/UPower.conf
```
```bash
sudo sed -i 's/^PercentageAction=2.0$/PercentageAction=10.0/' /etc/UPower/UPower.conf
```
```bash
sudo sed -i 's/^CriticalPowerAction=HybridSleep$/CriticalPowerAction=PowerOff/' /etc/UPower/UPower.conf
```
```bash
diff /etc/UPower/UPower.conf.bak-AAAAMMDD /etc/UPower/UPower.conf
```
✅ Exactamente cuatro cambios, con los valores nuevos.
```bash
sudo systemctl restart upower
```

**Comprobar:**
```bash
upower -d | grep critical-action
```
✅ `critical-action: PowerOff`.

### 4.4 BIOS: límite de carga y encendido tras un corte (pendiente)

En persona: reiniciar y pulsar `F10` en el logotipo de HP. **Solo mirar**, y anotar si hay:
- Un límite de carga ("Adaptive Battery Optimizer", "Battery Care" o similar). Con el cargador siempre puesto, la batería al 100 % se desgasta antes.
- Encender al conectar el cargador, o Wake-on-LAN con el equipo apagado.

Cualquier cambio en la BIOS necesita confirmación expresa.

## 5. Verificación

- [x] logind responde `no` a suspender, hibernar, suspensión híbrida y suspender-y-hibernar.
- [x] logind ignora la tapa en los tres casos, sin haberse reiniciado.
- [ ] UPower: `critical-action: PowerOff` con los umbrales 25/15/10.
- [ ] Tapa cerrada 30 minutos con cargador: SSH vivo, `uptime` sigue, el registro de logind dice "Lid closed" sin suspender, y temperaturas anotadas antes y después.
- [ ] Cargador desenchufado 10 minutos: sigue encendido y `upower -d` muestra `on-battery: yes`.
- [ ] Reinicio con la tapa cerrada: vuelve solo, con su IP reservada, y SSH con clave + TOTP.
- [ ] BIOS revisada (4.4).
- [ ] El apagado al 10 % se comprueba solo por configuración, salvo que se haga una descarga completa.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| El archivo pegado tiene espacios al principio de cada línea | nano conserva la sangría al pegar | systemd la ignora; compruébalo con `busctl` (4.1 y 4.2) |
| Tras `restart` de logind se cierra la sesión gráfica | `restart` en vez de `reload` | Usa siempre `reload` |
| `CanSuspend` sigue en `yes` | Archivo fuera de `sleep.conf.d/`, sin extensión `.conf` o sin la línea `[Sleep]` | Revisa la ruta y el contenido |
| UPower ignora los porcentajes | Umbrales sin orden de mayor a menor | Aviso > crítico > acción |

## 7. Cómo revertirlo

```bash
sudo rm /etc/systemd/sleep.conf.d/10-homelab.conf
```
```bash
sudo rm /etc/systemd/logind.conf.d/10-homelab.conf
```
```bash
sudo systemctl reload systemd-logind
```
```bash
sudo cp -a /etc/UPower/UPower.conf.bak-AAAAMMDD /etc/UPower/UPower.conf
```
```bash
sudo systemctl restart upower
```

## 8. Seguridad

**Mitiga:**
- Pérdida de disponibilidad por una suspensión: cerrar la tapa, la inactividad de GNOME o la batería crítica ya no duermen el equipo.
- Corrupción de datos en un corte largo: apagado ordenado con margen, en vez de quedarse sin batería a medias.

**Queda:**
- Si la batería se agota del todo, el equipo **no vuelve solo** al volver la luz, salvo que la BIOS lo permita (4.4). Alguien tiene que pulsar el botón.
- Durante un corte, el router también se apaga: el servidor sigue encendido pero sin red.
- Con el cargador siempre puesto, la batería se desgasta antes si no hay límite de carga.
- Con la tapa cerrada se ventila peor: hay que vigilar las temperaturas (monitorización en la Fase 5).
- Moonlight con la tapa cerrada no está resuelto: la pantalla interna se apaga y Sunshine la captura por KMS. Se trata en la Fase 2.

## 9. Para la entrevista

**En 30 segundos:** "Para usar el portátil como servidor, desactivé la suspensión en `sleep.conf.d` y la acción de la tapa en `logind.conf.d`, cubriendo los tres casos de logind: batería, cargador y pantalla externa. Lo verifiqué preguntando a logind por D-Bus, sin suspender de verdad. En UPower leí el código fuente para saber que no admite archivos parciales y que ignora en silencio umbrales mal ordenados, y dejé un apagado limpio al 10 %."

**Preguntas posibles:**
- *¿Por qué desactivar la suspensión si ya ignoras la tapa?* Porque la tapa no es lo único que suspende: GNOME tras un rato de inactividad y UPower con batería crítica también. Cortarlo en la raíz cubre todos los casos.
- *¿Por qué no probaste con `systemctl suspend`?* Porque si fallaba, el equipo se dormía y perdía el acceso remoto. `CanSuspend` por D-Bus da la misma información sin riesgo.
- *¿Por qué `reload` y no `restart` de logind?* `reload` relee la configuración; reiniciar logind puede afectar a las sesiones abiertas.
- *¿Y si se va la luz?* La batería actúa como un SAI pequeño. Si el corte dura más, se apaga ordenadamente al 10 %.

## Fuentes

- `man logind.conf` y `man systemd-sleep.conf` de systemd 257.
- Código fuente de UPower 1.90.9 (`src/up-config.c`, `src/up-daemon.c`, `src/linux/up-backend.c`), etiqueta `v1.90.9` de [su repositorio oficial](https://gitlab.freedesktop.org/upower/upower).
