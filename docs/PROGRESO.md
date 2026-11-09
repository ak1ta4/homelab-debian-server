# Progreso

Registro para retomar el trabajo en una sesión nueva. Fechas en formato AAAA-MM-DD.

## Fase 0: auditoría (terminada)

- Auditoría de solo lectura del sistema, discos, gráficos, red y exposición.
- Decisiones: sin cifrado de disco (riesgo físico aceptado por el dueño), sin sandbox de Claude Code y usuario limitado para el agente desde antes de la Fase 1.
- Revisión del router (2026-11-02 y 2026-11-08):
  - Sin CGNAT. DMZ, reenvíos y UPnP desactivados.
  - El SSH propio del router respondía desde internet y su panel no permite cerrarlo. Se tapó con una regla de reenvío del TCP 22 hacia `<IP_SUMIDERO>`. Es una excepción documentada a "sin reenvíos". Comprobado desde datos móviles.

## Fase 1: base y endurecimiento (en curso)

| Paso | Estado |
|---|---|
| 1. Reservas DHCP del servidor y del PC | Hecho (2026-11-08). Se verifica en el próximo reinicio |
| 2. SSH de emergencia | Hecho (2026-11-08) |
| 3. Repositorio y documentación | En curso |
| 4. Portátil como servidor | Pendiente |
| 5. Disco de datos | Pendiente |
| 6. Actualizaciones de seguridad | Pendiente |
| 7. Firewall | Pendiente |
| 8. Docker | Pendiente |
| 9. Resto de la base | Pendiente |

### Paso 2: SSH de emergencia

- OpenSSH 10.0 de Debian 13. Configuración propia en `/etc/ssh/sshd_config.d/10-homelab.conf`: solo el grupo `sshusers`, sin root, `AuthenticationMethods publickey,keyboard-interactive` y sin reenvíos.
- TOTP con `libpam-google-authenticator`. `/etc/pam.d/sshd` no incluye `common-auth`, así que nunca pide la contraseña de Unix.
- Fail2ban 1.1.0, solo para `sshd`: 5 fallos en 10 min → 1 h, excluye `<IP_PC>`.
- Comprobado:
  - Entra con clave + código.
  - Se rechazan la contraseña, root, el código incorrecto y la conexión sin clave.
  - Un código de emergencia sirve.
  - Fail2ban bloquea y desbloquea, y no bloquea el PC.

### Paso 3: repositorio

- `git` de Debian y gitleaks 8.30.1 (binario oficial verificado con su checksum).
- Hooks `pre-commit`, `pre-applypatch` y `pre-push` en el servidor, con una configuración de gitleaks propiedad de root, fuera del repositorio. Ignoran `gitleaks:allow`.
- Comprobado: un token falso bloquea el commit.
- El agente trabaja en su clon y entrega parches; `<USUARIO_ADMIN>` los revisa, los aplica con `git am` y los sube con una clave de despliegue protegida con frase de paso.

## Próximo

Primera subida, guías de la Fase 0 y del SSH, y paso 4.
