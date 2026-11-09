# Paso 3: repositorio, gitleaks y GitHub

> **Versiones usadas:** git 2.47.3 (Debian), gitleaks 8.30.1 (binario oficial), cliente OpenSSH 10.0. Las acciones de GitHub están fijadas por commit en el workflow.

## 1. Qué vamos a hacer y por qué

**En sencillo:** la documentación vive en un cuaderno con historial (Git) y una copia en GitHub. Para que nunca se cuele una contraseña o la dirección de casa, un revisor automático (gitleaks) lee cada página antes de guardarla y antes de subirla. El agente de IA escribe en **su propio cuaderno**; tú decides qué pasa al cuaderno bueno.

**Técnicamente:**
- `/opt/homelab` es el repositorio de `<USUARIO_ADMIN>`. El agente solo puede leerlo.
- Hooks `pre-commit`, `pre-applypatch` y `pre-push` lanzan gitleaks con una configuración **propiedad de root, fuera del repositorio**, que añade reglas para datos de la casa.
- El agente trabaja en su clon y entrega parches; `<USUARIO_ADMIN>` los revisa, los aplica con `git am` y los sube con una **clave de despliegue** con frase de paso.
- En GitHub, un workflow repite la búsqueda sobre todo el historial.

## 2. Conceptos previos

Repositorio, commit, rama, parche, remoto, clave de despliegue, hook, secreto y dato identificativo: ver [`../00-conceptos.md`](../00-conceptos.md#git-y-el-repositorio).

## 3. Requisitos y estado inicial

- SSH de emergencia terminado ([paso 2](02-ssh-de-emergencia.md)).
- Cuenta de GitHub con 2FA. El repositorio lo crea la persona, no el agente.
- `git` sin instalar (en Debian 13 con GNOME no viene por defecto).

## 4. Pasos

### 4.1 Instalar git y gitleaks (servidor, `<USUARIO_ADMIN>`)

```bash
sudo apt install git
```

Debian trae gitleaks 8.16, de 2023, con menos reglas. Se usa el binario oficial de la última versión, verificando su suma:

```bash
cd "$(mktemp -d)"
```
```bash
V=8.30.1
```
```bash
U=https://github.com/gitleaks/gitleaks/releases/download/v$V
```
```bash
curl -fLO $U/gitleaks_${V}_linux_x64.tar.gz
```
```bash
curl -fLO $U/gitleaks_${V}_checksums.txt
```
```bash
sha256sum --ignore-missing -c gitleaks_${V}_checksums.txt
```
Esperado: `gitleaks_8.30.1_linux_x64.tar.gz: OK`. Si no, para.

```bash
tar -xzf gitleaks_${V}_linux_x64.tar.gz gitleaks
```
```bash
sudo install -m 755 gitleaks /usr/local/bin/
```
```bash
gitleaks version
```
Esperado: `8.30.1`.

> **Límite:** la suma viene de la misma página que el programa. Protege de una descarga corrupta, no de una publicación comprometida. Y este binario **no se actualiza con apt**: va en el calendario de mantenimiento. El proyecto anunció que solo publicará arreglos de seguridad y que su autor se centra en un sucesor (Betterleaks); se revisará en el mantenimiento.

### 4.2 Configuración de gitleaks fuera del repositorio (servidor, `<USUARIO_ADMIN>`)

```bash
sudo install -d -m 755 /etc/homelab
```
```bash
sudo nano /etc/homelab/gitleaks.toml
```
```toml
# Configuración local de gitleaks. Propiedad de root: el agente no puede cambiarla.
# Amplía las reglas por defecto con datos de la casa, que en el repositorio
# deben ir como marcadores (<IP_SERVIDOR>, <USUARIO_ADMIN>...).
title = "homelab"

[extend]
useDefault = true

[[rules]]
id = "homelab-ip-lan"
description = "IP de la red de casa: usa <IP_SERVIDOR>, <IP_PC>..."
regex = '''\b<RED_LAN_REGEX>\b'''

[[rules]]
id = "homelab-mac"
description = "Dirección MAC: usa <MAC_...>"
regex = '''\b[0-9A-Fa-f]{2}([:-][0-9A-Fa-f]{2}){5}\b'''

[[rules]]
id = "homelab-inventario"
description = "Valor real del inventario local (IP pública, usuarios, equipo)"
regex = '''(?i)\b(<IP_PUBLICA_REGEX>|<USUARIO_ADMIN>|<USUARIO_GITHUB>|<OTROS_NOMBRES>)\b'''
```
- `<RED_LAN_REGEX>`: la red de casa como expresión regular, con los puntos escapados y `\d{1,3}` en los números variables.
- `<IP_PUBLICA_REGEX>`: la IP pública con los puntos escapados.
- Los nombres van separados por `|`.

**Por qué fuera del repositorio:** si estuviera dentro, un cambio podría debilitarla (gitleaks lee `.gitleaks.toml` del repositorio que revisa). Además contiene los valores reales que precisamente no deben subirse.

### 4.3 Repositorio y hooks (servidor, `<USUARIO_ADMIN>`)

```bash
sudo install -d -o <USUARIO_ADMIN> -g <USUARIO_ADMIN> -m 755 /opt/homelab
```
Tuyo; el resto de usuarios, incluido el agente, solo puede leerlo.

```bash
git -C /opt/homelab init -b main
```
```bash
git -C /opt/homelab config user.name homelab
```
```bash
git -C /opt/homelab config user.email homelab@invalid
```
Autor sin datos personales. `.invalid` es un dominio reservado que nunca existirá.

Los tres hooks van en `/opt/homelab/.git/hooks/`, con permisos 755:

`pre-commit`:
```sh
#!/bin/sh
# Homelab: bloquea el commit si hay secretos o datos identificativos en lo preparado.
# Usa la configuración de /etc/homelab (no la del repositorio) e ignora los
# comentarios gitleaks:allow, para que un cambio no pueda desactivar la revisión.
exec gitleaks git --pre-commit --staged --config /etc/homelab/gitleaks.toml \
  --gitleaks-ignore-path /etc/homelab --ignore-gitleaks-allow \
  --redact --no-banner -v
```

`pre-applypatch`: idéntico, cambiando el comentario ("bloquea el parche de git am…"). `git am` lo ejecuta tras aplicar cada parche y antes de crear el commit.

`pre-push`:
```sh
#!/bin/sh
# Homelab: antes de subir a GitHub, revisa todo el historial con la misma configuración.
exec gitleaks git --config /etc/homelab/gitleaks.toml \
  --gitleaks-ignore-path /etc/homelab --ignore-gitleaks-allow \
  --redact --no-banner
```

- `--redact`: oculta el secreto en la salida (`REDACTED`).
- `--gitleaks-ignore-path /etc/homelab`: el archivo de excepciones (`.gitleaksignore`) se busca ahí, no en el repositorio.
- `--ignore-gitleaks-allow`: no hace caso de los comentarios que dicen "ignora esta línea".

**Prueba obligatoria con un secreto falso** (en `/opt/homelab`):
```bash
cd /opt/homelab
```
```bash
T=$(tr -dc A-Za-z0-9 </dev/urandom | head -c 36)
```
```bash
echo "token = ghp_$T" > prueba.txt
```
```bash
git add prueba.txt
```
```bash
git commit -m prueba
```
Esperado: `RuleID: github-pat`, `leaks found: 1`, y **el commit no se hace**. Limpieza:
```bash
git rm -q --cached prueba.txt
```
```bash
rm prueba.txt
```

### 4.4 Clon del agente (servidor, usuario del agente)

```bash
git config --global --add safe.directory /opt/homelab/.git
```
Git se niega a trabajar con repositorios de otro usuario (*dubious ownership*). Se autoriza **solo** este, cuyo dueño es el administrador.

```bash
git clone /opt/homelab ~/homelab
```
```bash
git -C ~/homelab config user.name homelab-agent
```
```bash
git -C ~/homelab config user.email homelab-agent@invalid
```
El agente copia también el `pre-commit` a su clon. Ahí es una ayuda, no un control: el agente podría quitarlo. **El control real son los hooks de `/opt/homelab`.**

### 4.5 Repositorio y clave de despliegue en GitHub

**En el navegador (persona):**
1. **+ → New repository**, nombre `homelab-debian-server`, **Private**, sin README, sin `.gitignore` y sin licencia.
2. **Settings → Applications → Installed GitHub Apps:** revisa qué apps tienen acceso a **All repositories**. Aquí la app de Claude solo tiene acceso a otros repositorios, y GitGuardian se retiró de todos.

**En el servidor (`<USUARIO_ADMIN>`):**
```bash
ssh-keygen -t ed25519 -f ~/.ssh/github-homelab -C deploy-homelab
```
Frase de paso **nueva**, distinta de la del PC.

```bash
ssh-keygen -y -f ~/.ssh/github-homelab
```
Pide la frase y muestra la parte pública: comprueba que la frase guardada es la buena.

```bash
cat ~/.ssh/github-homelab.pub
```
**En GitHub:** el repositorio → **Settings → Deploy keys → Add deploy key**. Título `servidor-homelab`, pega la línea, marca **Allow write access** y pulsa **Add key**.

**Huella de GitHub:** añádela a `~/.ssh/known_hosts` comparándola con [la publicada por GitHub](https://docs.github.com/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints). Así no hay que fiarse de la primera conexión.

**Alias en `~/.ssh/config`:**
```
# Clave de despliegue solo para el repositorio homelab-debian-server.
Host github-homelab
    HostName github.com
    User git
    IdentityFile ~/.ssh/github-homelab
    IdentitiesOnly yes
    # Sin agente: pide la frase de paso en cada subida.
    IdentityAgent none
```
```bash
chmod 600 ~/.ssh/config
```
```bash
ssh -G github-homelab | grep -i identityagent
```
Esperado: `identityagent none`.

```bash
ssh -T github-homelab
```
Esperado: pide la frase **en la terminal** y dice `Hi <USUARIO_GITHUB>/homelab-debian-server! You've successfully authenticated…`.

```bash
cd /opt/homelab
```
```bash
git remote add origin git@github-homelab:<USUARIO_GITHUB>/homelab-debian-server.git
```

### 4.6 Ciclo de trabajo

1. **Agente**, en su clon: trabaja en una rama, hace commits y exporta los parches a `/tmp/homelab-parches/` con `git format-patch`. Da la suma SHA-256 del conjunto.
2. **Administrador:**
   ```bash
   cd /tmp/homelab-parches
   ```
   ```bash
   sha256sum *.patch | sha256sum
   ```
   Compara con la del agente.
   ```bash
   less *.patch
   ```
   Lee lo que vas a publicar (`:n` siguiente archivo, `q` salir).
   ```bash
   cd /opt/homelab
   ```
   ```bash
   git am /tmp/homelab-parches/*.patch
   ```
   Cada parche pasa por gitleaks (`pre-applypatch`).
   ```bash
   git push
   ```
   Frase de paso y gitleaks sobre todo el historial (`pre-push`).
3. **Agente:** sincroniza su clon con `/opt/homelab` y borra los temporales.

En cada commit queda como **autor** `homelab-agent` y como **committer** `homelab`: se ve qué propuso el agente y qué aprobó la persona.

### 4.7 gitleaks en GitHub Actions

`.github/workflows/gitleaks.yml` instala gitleaks 8.30.1 comprobando su SHA-256, revisa **todo el historial** y exige haber revisado al menos todos los commits de la rama. Las acciones de terceros están fijadas por commit, con la versión en un comentario. Ver el archivo en el repositorio.

## 5. Verificación

- [x] Un token falso bloquea el commit en `/opt/homelab`.
- [x] Un parche con una IP de casa se bloquea en `git am` (probado en repositorios de prueba).
- [x] El agente no puede escribir en `/opt/homelab` ni en sus hooks.
- [x] `git push` pide la frase de paso y gitleaks revisa antes de subir.
- [x] El workflow de GitHub termina en verde. En pruebas locales falla con un token falso y también si git no revisa nada.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `fatal: detected dubious ownership` al clonar | El repositorio es de otro usuario | `safe.directory` solo para esa ruta (4.4) |
| `ssh -T` no pide la frase de paso | El agente de claves del llavero de GNOME la cargó la primera vez | `ssh-add -d ~/.ssh/github-homelab` y `IdentityAgent none` en el alias |
| `keyword identitiesonly extra arguments at end of line` | Dos directivas quedaron en la misma línea de `~/.ssh/config` | Una directiva por línea |
| El workflow de GitHub falla en la primera subida | `gitleaks-action` revisa desde `primer_commit^`, que no existe en el primer commit; git falla y no revisa nada | Workflow propio que revisa todo el historial y comprueba cuántos commits revisó ([lecciones](../lecciones-aprendidas.md#gitleaks-action-y-la-primera-subida)) |
| `Unknown SCM platform` en el `pre-push` | gitleaks no reconoce el alias `github-homelab` para generar enlaces | Solo informativo |

## 7. Cómo revertirlo

- **Quitar los hooks:** borrar `pre-commit`, `pre-applypatch` y `pre-push` de `/opt/homelab/.git/hooks/`.
- **Quitar la clave de despliegue:** en GitHub, **Settings → Deploy keys → Delete**. En el servidor, borrar `~/.ssh/github-homelab*` y su bloque de `~/.ssh/config`.
- **Quitar gitleaks:** `sudo rm /usr/local/bin/gitleaks /etc/homelab/gitleaks.toml`.
- Si llegara a subirse un secreto: **rotarlo** primero. Borrarlo del historial no basta.

## 8. Seguridad

**Mitiga:**
- Secretos y datos de la casa en el repositorio, con tres barreras locales y una en GitHub.
- Que el agente publique sin revisión: no tiene credenciales de GitHub ni escritura en `/opt/homelab`.
- Que alguien suba con la clave del servidor sin saber la frase.

**Queda:**
- gitleaks no detecta todo: un secreto con formato raro o un dato identificativo no previsto pasaría. La revisión humana del parche sigue siendo necesaria.
- La clave de despliegue con escritura puede borrar ramas e historial de **ese** repositorio. No caduca: rotarla en el calendario.
- Root en el servidor podría modificar los hooks o la configuración.
- El binario de gitleaks se actualiza a mano.

## 9. Para la entrevista

**En 30 segundos:** "La documentación va en Git con gitleaks en tres puntos: commit, aplicación de parches y subida, con una configuración propiedad de root y fuera del repositorio que añade reglas para datos de mi red, así que ni el agente ni un cambio pueden desactivarla. El agente de IA no tiene credenciales: entrega parches que reviso y subo con una clave de despliegue con frase de paso. En GitHub repito la búsqueda sobre todo el historial con una versión fijada por checksum."

**Preguntas posibles:**
- *¿Por qué no darle un token de GitHub al agente?* Crear repositorios exige un permiso que también sirve para borrar o modificar **todos** los repositorios de la cuenta. Con parches y una clave de un solo repositorio, el peor caso queda acotado y siempre hay revisión humana.
- *¿Por qué la configuración de gitleaks fuera del repositorio?* Porque gitleaks lee `.gitleaksignore` y `.gitleaks.toml` del repositorio: un cambio podría silenciarlo. Fuera, y propiedad de root, no.
- *¿Por qué no usaste `gitleaks-action`?* En la primera subida revisaba desde un commit inexistente, git fallaba y no se revisaba nada. Lo reproduje en local, lo sustituí por un paso que revisa todo el historial y comprueba el número de commits revisados, y probé que falla con un secreto falso.
- *¿Qué haces si se sube un secreto?* Rotarlo en el servicio correspondiente, y después limpiar el historial. El orden importa: lo publicado ya puede estar copiado.

## Fuentes

- [gitleaks](https://github.com/gitleaks/gitleaks) y [gitleaks-action](https://github.com/gitleaks/gitleaks-action) (código revisado de la v3.0.0).
- [GitHub: claves de despliegue](https://docs.github.com/authentication/connecting-to-github-with-ssh/managing-deploy-keys).
- [GitHub: huellas de sus claves SSH](https://docs.github.com/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints).
