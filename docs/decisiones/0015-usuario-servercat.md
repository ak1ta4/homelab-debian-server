# ADR 0015: usuario `servercat` sin TOTP para el widget del iPhone

- **Estado:** aceptada (2026-11-18).
- **Decide:** el dueño.

## Contexto

El dueño quiere ver el estado del servidor en un widget del iPhone con ServerCat. La app lee `/proc` por SSH, sin instalar nada y sin root, y se conecta cada pocos segundos mientras está abierta. El SSH exige clave y TOTP a todos ([ADR 0003](0003-ssh-clave-totp-fail2ban.md)), y la cuenta que hay es la de administrador. Se le propuso usar Beszel en su lugar ([ADR 0014](0014-beszel-y-dozzle.md)), pero quiere el widget.

## Opciones

| Opción | A favor | En contra |
|---|---|---|
| No usar ServerCat (Beszel en el navegador) | Nada cambia en el SSH | Sin widget |
| ServerCat con la cuenta de administrador | Nada nuevo que crear | Pediría el TOTP en cada conexión, y una app de terceros tendría la clave de una cuenta con `sudo` |
| **Usuario propio, solo clave, solo desde casa y la tailnet** | Funciona el widget; si se pierde la clave, solo se lee | Una excepción al TOTP |

## Decisión

- Usuario `servercat`, sin contraseña, sin `sudo` ni `docker`, en `sshusers`.
- Su clave, generada en la app, con `restrict` y `from="<RED_LAN>,100.64.0.0/10"`.
- `Match User servercat Address <RED_LAN>,100.64.0.0/10` con `AuthenticationMethods publickey` y `PermitTTY no`. Desde cualquier otra dirección, el TOTP sigue siendo obligatorio, y `servercat` no lo tiene.
- Comprobado con `sshd -T` que la configuración de `<USUARIO_ADMIN>` no cambia.

## Consecuencias

- **Comprobado:** ServerCat muestra los datos en casa; `<USUARIO_ADMIN>` sigue con clave y código.
- Quien tenga el iPhone desbloqueado y esté en casa o en la tailnet puede leer lo que lee cualquier usuario normal: procesos, uso de recursos y archivos legibles para todos.
- Fail2ban, en modo *aggressive*, bloquea el iPhone si la app se conecta mal: cuenta como fallo cada conexión que se cierra sin autenticarse, y la app lo hace cada 5 segundos.
- La regla del CLAUDE.md "clave + TOTP" tiene esta única excepción.
- Fuera de casa, por la tailnet: falta probarlo.
