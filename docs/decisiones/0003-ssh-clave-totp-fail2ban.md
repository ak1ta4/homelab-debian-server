# ADR 0003: SSH de emergencia con clave + TOTP y Fail2ban

- **Estado:** aceptada (2026-11-08).

## Contexto

El único acceso remoto era el escritorio remoto (Moonlight), frágil ante fallos de gráficos, red o firewall. Hacía falta un acceso de rescate por SSH que no fuera un punto débil. El servidor no está expuesto a internet; el acceso remoto desde fuera llegará con Tailscale (Fase 3). El dueño no tenía experiencia previa con claves SSH.

## Opciones

| Opción | A favor | En contra |
|---|---|---|
| Contraseña | Sencillo | Adivinable; la contraseña viaja al servidor |
| Solo clave | Muy robusto frente a adivinar | Una clave robada basta |
| **Clave + TOTP** | Hace falta el PC **y** el móvil | Más pasos; si se pierde el móvil, hace falta un plan de rescate |
| Cambiar el puerto 22 | Menos ruido en servidores expuestos | Aquí no está expuesto; complica el rescate |
| Tailscale SSH | Identidad de la cuenta de Tailscale | Depende de un servicio externo; no sirve de rescate si falla Tailscale |

## Decisión

- **Clave Ed25519 con frase de paso + TOTP**, exigidos los dos con `AuthenticationMethods publickey,keyboard-interactive`.
- **PAM sin `common-auth`:** nunca pide la contraseña de Unix. **Sin `nullok`.**
- Solo el grupo `sshusers`, sin root, sin reenvíos.
- **Puerto 22 sin cambiar.**
- **Fail2ban solo para `sshd`**, leyendo el journal: 5 fallos en 10 minutos → 1 hora de bloqueo, excluido el PC de rescate.
- Configuración instalada **antes** que el paquete `openssh-server`.

## Consecuencias

- Rescate escrito y probado: códigos de emergencia, escritorio remoto y acceso físico.
- Con `mode = aggressive`, un solo intento con un usuario inexistente ya bloquea. Se acepta: contra desconocidos es deseable, y el PC está excluido.
- Hasta el paso 7 (firewall), el 22 es alcanzable desde toda la LAN.
- Algoritmos y `moduli` sin restringir: se harán en el paso 9, tras comprobar la compatibilidad con los clientes (Windows, iPad, iPhone).
- El usuario del agente no entra en `sshusers`. Si necesita SSH en la Fase 7, será con un bloque `Match` explícito y solo por la tailnet.
- **Actualización (2026-11-18):** primera excepción al TOTP, el usuario `servercat` para el widget del iPhone: solo clave y solo desde casa y la tailnet ([ADR 0015](0015-usuario-servercat.md)).
