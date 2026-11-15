# ADR 0007: filtro de salida del agente de IA

- **Estado:** aceptada (2026-11-14). Revisa el riesgo "red de salida abierta" de la [ADR 0002](0002-permisos-del-agente-de-ia.md).
- **Decide:** el dueño.

## Contexto

El agente (`homelab-agent`) no tiene privilegios, pero podía conectarse a cualquier destino: el panel del router, el de Sunshine, otros aparatos de casa y cualquier puerto de internet. Una instrucción maliciosa leída en una web o un log podía usarlo para sondear la red de casa o sacar datos por un canal poco vigilado. Claude Code necesita DNS y HTTPS hacia Anthropic, y el agente consulta documentación oficial en la web.

## Opciones

| Opción | A favor | En contra |
|---|---|---|
| Sin filtro | Nada que mantener | El riesgo de la ADR 0002 sigue abierto |
| **Sin red de casa ni equipo local; a internet solo DNS y HTTPS** | Sencillo; corta el acceso a la LAN y a los servicios locales; no se rompe al cambiar las IP de los servicios web | Por HTTPS puede llegar a cualquier web, y por tanto sacar datos legibles |
| Lista cerrada de dominios (proxy con lista permitida) | Corta también la salida de datos a webs arbitrarias | Proxy que mantener; cada fuente nueva de documentación exige pedir permiso |
| Sandbox de Claude Code | Integrado | Descartado por ahora (ADR 0002); se revisa en la Fase 7 |

## Decisión

Tabla nftables propia, `inet agente_salida`, en `/etc/homelab/agente-salida.nft` (propiedad de root), cargada por `homelab-agente-salida.service` antes de que haya red (`Before=network-pre.target`). Solo afecta a los paquetes cuyo dueño es `homelab-agent` (`meta skuid`):
- Conexiones ya establecidas: permitidas.
- Hacia el propio equipo (`lo`): rechazadas.
- Hacia rangos privados (LAN, `100.64.0.0/10` de Tailscale, enlace local y multidifusión, IPv4 e IPv6): rechazadas.
- DNS (53) y HTTPS (443) hacia internet: permitidos.
- Todo lo demás: rechazado.

No se usa `/etc/nftables.conf`, porque empieza con `flush ruleset` y borraría las reglas de UFW y Fail2ban.

## Consecuencias

- **Comprobado:** DNS y HTTPS a Anthropic y Debian funcionan. El router (443 y 22), el panel de Sunshine (local y por la LAN), el SSH local y el HTTP (80) se rechazan.
- **Riesgo que queda:** por HTTPS, el agente puede enviar a cualquier web lo que pueda leer. Se valorará la lista cerrada de dominios en la Fase 7.
- **Tailscale (2026-11-15):** su DNS (`100.100.100.100`) está en `100.64.0.0/10`, bloqueado para el agente. Con él activo, el agente se queda sin DNS: el servidor usa `--accept-dns=false` ([ADR 0009](0009-tailscale.md)).
- En la **Fase 4**, si el DNS pasa a AdGuard en el propio servidor, el agente no podrá usarlo (`lo` y LAN rechazados): habrá que permitir el 53 local o seguir con DNS públicos para el agente.
- Si Claude Code necesitara otro puerto o un servicio local, fallará de forma visible ("connection refused") y se decidirá aquí, no con una excepción silenciosa.
- Revertir: `sudo systemctl disable --now homelab-agente-salida`.
