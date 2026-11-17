# Fase 2: escritorio remoto (Sunshine y Moonlight)

> **Versiones usadas:** Debian 13.7, Sunshine 2026.914.233613 (`.deb` de LizardByte), driver NVIDIA 550.163.01 (sin tocar).

## 1. Qué vamos a hacer y por qué

**En sencillo:** el escritorio remoto ya funcionaba. Esta fase lo deja **actualizándose solo**, comprueba su configuración y quién tiene acceso, y explica cómo se usa con la tapa del portátil.

**Técnicamente:**
- Repositorio APT oficial de LizardByte (en Cloudsmith), con la clave comprobada, limitado al paquete `sunshine` y dentro de `unattended-upgrades`.
- Configuración comprobada: `capture = kms`, `encoder = vaapi`, `adapter_name = /dev/dri/renderD128`.
- Clientes emparejados revisados.
- Tapa **entreabierta**, por decisión del dueño.
- Seguimiento del aviso del driver NVIDIA.

## 2. Conceptos previos

- **Sunshine:** el servidor de escritorio remoto; **Moonlight:** el cliente.
- **Captura KMS:** Sunshine lee directamente la imagen que la gráfica envía a la pantalla. Necesita `cap_sys_admin` en el binario (lo pone el `postinst` del paquete) y una pantalla encendida.
- **Pin de APT:** prioridad que decide de qué repositorio puede salir cada paquete.

## 3. Requisitos y estado inicial

- Sunshine instalado a mano desde un `.deb` de GitHub: **sin actualizaciones**. En 2026 tuvo al menos siete avisos de seguridad (cinco altos y uno crítico); la versión instalada ya los corrige.
- Servicio de usuario `app-dev.lizardbyte.app.Sunshine`, que arranca con la sesión automática ([ADR 0006](decisiones/0006-reinicio-e-inicio-de-sesion-automaticos.md)).
- Puertos abiertos solo a `<RED_LAN>` en UFW ([paso 7](01-base-y-endurecimiento/07-firewall.md)).

## 4. Pasos

### 4.1 Repositorio oficial y actualizaciones automáticas

La documentación de Sunshine propone `curl … | sudo -E bash`. En lugar de ejecutar ese script, se leyó y se reprodujo a mano. La clave se comprobó así:
- Huella: `10BA 2B87 6CFD EE01 245A 2F2B 04C8 71E8 D308 324E`.
- El `InRelease` de `trixie` está firmado con ella.
- El `.deb` del repositorio y el de la publicación de GitHub de la misma versión tienen la **misma suma SHA-256**: el repositorio sirve lo que publica el proyecto.

```bash
sudo curl -fsSL -o /etc/apt/keyrings/lizardbyte-stable.asc https://dl.cloudsmith.io/public/lizardbyte/stable/gpg.04C871E8D308324E.key
```
```bash
gpg --show-keys --with-fingerprint /etc/apt/keyrings/lizardbyte-stable.asc
```
**Resultado:** La huella de arriba. Si no coincide, borra el archivo y no sigas.

`/etc/apt/sources.list.d/lizardbyte-stable.sources`:
```
Types: deb
URIs: https://dl.cloudsmith.io/public/lizardbyte/stable/deb/debian
Suites: trixie
Components: main
Architectures: amd64
Signed-By: /etc/apt/keyrings/lizardbyte-stable.asc
```

`/etc/apt/preferences.d/lizardbyte-stable`. De ese repositorio solo se acepta `sunshine`, aunque mañana publique otros paquetes:
```
Package: *
Pin: origin dl.cloudsmith.io
Pin-Priority: 1

Package: sunshine
Pin: origin dl.cloudsmith.io
Pin-Priority: 500
```

`/etc/apt/apt.conf.d/54homelab-unattended-upgrades-sunshine`:
```
Unattended-Upgrade::Origins-Pattern {
        "origin=cloudsmith/lizardbyte/stable,codename=trixie";
};
```
```bash
sudo apt update
```
```bash
apt-cache policy sunshine
```
**Resultado:** La versión instalada aparece también en `dl.cloudsmith.io` con prioridad 500.
```bash
sudo unattended-upgrade --dry-run --debug 2>&1 | grep "Allowed origins"
```
**Resultado:** Incluye `origin=cloudsmith/lizardbyte/stable,codename=trixie`.

Tras una actualización de Sunshine, el servicio sigue con el binario viejo hasta reiniciarse: `needrestart` lo avisa y el reinicio llega con la sesión siguiente.

### 4.2 Configuración

```bash
grep -vE '^\s*(#|$)' ~/.config/sunshine/sunshine.conf
```
**Resultado:** Solo `capture = kms`, `encoder = vaapi` y `adapter_name = /dev/dri/renderD128`. Las credenciales están en `sunshine_state.json`, que nunca se muestra.

### 4.3 Clientes emparejados

En el panel (`https://<IP_SERVIDOR>:47990`, desde la LAN), en la lista de clientes: solo el PC y el iPad. Si aparece uno desconocido o que ya no se usa, se desempareja.

### 4.4 Tapa entreabierta

Sunshine captura la pantalla interna por KMS: con la tapa cerrada se apaga y no hay imagen. Opciones valoradas:

| Opción | Por qué no |
|---|---|
| Pantalla virtual (EDID forzado) | Toca el arranque |
| Enchufe HDMI falso | El HDMI va por la NVIDIA, y la captura entre las dos gráficas ya falló ([lecciones](lecciones-aprendidas.md#pantalla-negra-en-moonlight-con-nvidia)) |
| **Tapa entreabierta** | Elegida por el dueño: sin cambios y, de paso, ventila mejor |

## 5. Verificación

- [x] Clave comprobada con la firma del `InRelease` y con el mismo `.deb` en GitHub.
- [x] `apt-cache policy` y `unattended-upgrade --dry-run` incluyen el repositorio.
- [x] Configuración `kms`, `vaapi` y `renderD128`.
- [x] Solo el PC y el iPad emparejados. El iPhone se emparejó después, en la [Fase 3](03-acceso-remoto.md).
- [x] Moonlight y una película de Jellyfin a la vez, con la tele por cable, sin cortes.
- [ ] Reiniciar Sunshine y comprobar que el panel ya no muestra el aviso de arranque.
- [ ] Tras un reinicio del servidor, Moonlight funciona sin tocar nada. En la sesión en persona.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| El panel avisa de `Unable to find display or encoder during startup`, pero Moonlight funciona | Sunshine arrancó con la sesión automática antes de que la pantalla estuviera lista; luego la encuentra al conectar | Es un aviso del arranque. `systemctl --user restart app-dev.lizardbyte.app.Sunshine` lo borra (corta la sesión de Moonlight) |
| `WARNING … nv_drm_revoke_modeset_permission` en el núcleo al conectar Moonlight | Al empezar una sesión, Sunshine abre todas las pantallas por KMS, también la de la NVIDIA, y el driver 550 se queja al soltarla | Sin efecto visible. No se tocan los drivers; se investiga solo si coincide con cortes o cuelgues |
| Pantalla negra en Moonlight | Tapa cerrada | Entreabrirla |
| Moonlight satura la red | Velocidad de vídeo al máximo en el cliente (se midieron 450 Mbit/s) | En Moonlight, 50-100 Mbit/s bastan para el escritorio |

## 7. Cómo revertirlo

```bash
sudo rm /etc/apt/sources.list.d/lizardbyte-stable.sources /etc/apt/preferences.d/lizardbyte-stable
```
```bash
sudo rm /etc/apt/apt.conf.d/54homelab-unattended-upgrades-sunshine /etc/apt/keyrings/lizardbyte-stable.asc
```
```bash
sudo apt update
```
Sunshine sigue instalado, pero vuelve a quedar sin actualizaciones.

## 8. Seguridad

**Mitiga:**
- Vulnerabilidades conocidas de Sunshine: se corrigen solas de madrugada.
- Un repositorio de terceros que publicara otros paquetes: el pin solo acepta `sunshine`.
- Clientes olvidados con acceso: revisados.

**Queda:**
- El binario tiene `cap_sys_admin`: un fallo en Sunshine es más grave que en otro programa de usuario. Por eso importa actualizarlo.
- Se confía en LizardByte y en Cloudsmith. La clave se comprobó con la firma del índice y comparando el paquete con GitHub, pero no hay una huella publicada por el proyecto.
- El panel (47990) está abierto a toda la LAN por decisión del dueño (paso 7).

## 9. Para la entrevista

**En 30 segundos:** "El escritorio remoto ya funcionaba, pero Sunshine estaba instalado desde un `.deb` suelto y no se actualizaba, con al menos siete avisos de seguridad en el año. Añadí su repositorio oficial sin ejecutar el script `curl | bash` que propone la documentación: comprobé la clave con la firma del índice y comparando el paquete con el de GitHub, limité con un pin el repositorio a ese único paquete y lo metí en las actualizaciones automáticas."

**Preguntas posibles:**
- *¿Por qué no `curl | sudo bash`?* Ejecuta como root lo que sirva el servidor en ese momento. Leído y reproducido a mano, sé exactamente qué cambia en el sistema.
- *¿Para qué el pin?* Un repositorio de terceros con prioridad normal podría sustituir cualquier paquete de Debian con uno suyo del mismo nombre.

## Fuentes

- [Sunshine: Getting Started](https://docs.lizardbyte.dev/projects/sunshine/latest/md_docs_2getting__started.html).
- [Publicaciones](https://github.com/LizardByte/Sunshine/releases) y [avisos de seguridad](https://github.com/LizardByte/Sunshine/security/advisories) de Sunshine.
- `man apt_preferences` y `man sources.list` de Debian 13.
