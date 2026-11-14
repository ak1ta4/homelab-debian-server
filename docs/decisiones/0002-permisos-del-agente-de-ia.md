# ADR 0002: el agente de IA trabaja en una cuenta sin privilegios

- **Estado:** aceptada (2026-11-02). Se revisa en la Fase 1 (filtro de salida) y en la Fase 7 (acceso remoto del agente).
- **Decide:** el dueño.

## Contexto

Claude Code administra el homelab junto al dueño. Empezó ejecutándose como **root** durante la auditoría, y después se planeó como el usuario administrador con `sudo`. En cualquiera de los dos casos, un error del agente o una instrucción maliciosa leída en una web o un log podía tocar todo el sistema y leer los datos personales del dueño. Además se comprobó que la sesión gráfica del administrador permitía, vía polkit y **sin contraseña**, apagar el equipo, cambiar la red y montar discos.

## Opciones

| Opción | A favor | En contra |
|---|---|---|
| **A.** Agente como root o como el administrador | Cómodo: hace todo solo | Sin límite real; lee la carpeta personal, tokens y sesiones |
| **B.** Administrador + *sandbox* de Claude Code | Limita red por dominios y escritura | Por defecto deja leer casi todo el equipo; solo cubre Bash, no el resto de herramientas; falta una dependencia, y sin ella ejecuta sin sandbox |
| **C.** Cuenta propia sin privilegios (`homelab-agent`) | El núcleo aplica el límite a **todos** sus procesos y herramientas | Más trabajo para el dueño: ejecuta él los comandos con `sudo` y pega las salidas |

## Decisión

**Opción C**, adelantada a antes de la Fase 1:
- `homelab-agent` sin contraseña de inicio de sesión, **sin `sudo`**, sin grupo `docker` y sin acceso a la carpeta del administrador. Comprobado con pruebas negativas: `sudo -n`, lectura de carpetas ajenas, polkit, pantallas gráficas y entorno heredado.
- **Configuración gestionada** de Claude Code (`/etc/claude-code/managed-settings.json`, propiedad de root): prohíbe el modo sin permisos y el modo automático, desactiva los conectores y la sincronización de plugins y skills, y no permite servidores MCP.
- Sin credenciales de GitHub: el agente entrega parches que el dueño revisa y publica ([ADR 0005](0005-gitleaks-local-y-en-github.md)).
- **Sandbox: no por ahora.** Se revisa al configurar el firewall.

## Consecuencias

- **Queda limitado por el sistema:** la carpeta del dueño, `sudo`, Docker y las sesiones del dueño.
- **Sigue siendo posible:**
  - **Red de salida abierta:** el agente puede conectarse a internet y a la LAN, incluidos el panel del router y el de Sunshine, aunque sin sus credenciales, y enviar fuera lo que pueda leer. Riesgo aceptado hasta decidir el filtro de salida (Fase 1, paso 7). **Actualización (2026-11-14):** limitado por la [ADR 0007](0007-filtro-de-salida-del-agente.md): sin LAN ni servicios locales, y a internet solo DNS y HTTPS.
  - **Lectura** de todo lo que sea legible para cualquier cuenta.
  - **Cuenta de Anthropic compartida** con el dueño, con Remote Control activo. Se revisa en la Fase 7.
  - **Fallos del núcleo:** una vulnerabilidad local permitiría escalar privilegios.
- La revisión humana de cada comando con `sudo` sigue siendo un control real.
- **Lecciones de la puesta en marcha:**
  - `adduser` añadía la cuenta al grupo `users`.
  - `sudo -iu` conservaba `DISPLAY` y `XAUTHORITY`.
  - `--dangerously-skip-permissions` se ignoraba en vez de rechazarse.

  Ver [lecciones aprendidas](../lecciones-aprendidas.md#puesta-en-marcha-de-la-cuenta-del-agente).
