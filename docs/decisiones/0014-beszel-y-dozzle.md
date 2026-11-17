# ADR 0014: monitorización con Beszel y Dozzle, sin dar a nada el socket de Docker

- **Estado:** aceptada (2026-11-17).
- **Decide:** el dueño.

## Contexto

El panel ([ADR 0013](0013-panel-con-homepage.md)) muestra el estado del momento. Falta el historial (CPU, RAM, discos, temperatura y batería de un portátil que trabaja cerrado) y una forma cómoda de leer los registros de los contenedores. El plan de la Fase 5 nombraba Beszel y Dozzle. Las dos herramientas, tal como las documentan, montan el socket de Docker, que equivale a root.

## Opciones

**Historial y alertas:**

| Opción | A favor | En contra |
|---|---|---|
| **Beszel** | Ligero; temperatura, batería y GPU; alertas a Telegram | Joven; 4 avisos en 2026 |
| Grafana con Prometheus | Estándar y potente | Tres o cuatro servicios más que mantener |
| Netdata | Muy completo | Pesado; empuja a su nube |

**Cómo instala Beszel su agente:**

| Opción | A favor | En contra |
|---|---|---|
| Contenedor oficial | Sencillo | `network_mode: host` y el socket para ver contenedores |
| `.deb` oficial | Trae el servicio de systemd | Su `postinst` mete al usuario `beszel` en el grupo `docker` |
| **Binario con SHA-256 y servicio propio** | Sin Docker, sin root y sin puertos | Se actualiza a mano |

**Registros (Dozzle):**

| Opción | A favor | En contra |
|---|---|---|
| No instalarlo | Nada nuevo con acceso a Docker | `sudo docker logs` cada vez |
| Con el socket montado | Lo que propone su documentación | Equivale a root; varios avisos graves en 2026 |
| **Detrás de un proxy de solo lectura** | Sin escritura en Docker | Un contenedor más; el proxy sí tiene el socket |

## Decisión

- **Beszel 0.21.2:** el hub en Docker con el usuario `beszel-hub`. El agente es el binario oficial con su SHA-256, como servicio con el usuario `beszel`, sin el grupo `docker`, sin capacidades y sin puertos (`DISABLE_SSH`). No ve los contenedores, a propósito. Solo mide la GPU AMD.
- **Dozzle v11.4.3:** con usuario y contraseña, sin acciones ni consola, detrás de `docker-socket-proxy` con `POST=0` en una red interna. Lo pidió el dueño, sabiendo el riesgo.
- Las dos webs, desde la LAN y la tailnet (`DOCKER-USER` y `"tcp:8090"`, `"tcp:8080"`). Amplía el [ADR 0009](0009-tailscale.md).

## Consecuencias

- **Comprobado:** el hub corre sin root; el agente, como `beszel` y sin puertos; el proxy responde `403` a un `POST`; Dozzle ve los 11 contenedores.
- El proxy sí tiene el socket. Si alguien lo comprometiera, solo podría leer, pero leer incluye las variables de entorno de los contenedores (con `docker inspect`).
- Hay que vigilar los avisos de Dozzle y de Beszel, y actualizar los digest a mano.
- Las alertas de Beszel a Telegram se configuran en el paso 5.2.
