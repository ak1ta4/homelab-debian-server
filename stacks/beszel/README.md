# Beszel: gráficas e historial del servidor

> **Versión usada:** Beszel `0.21.2` (hub y agente). Docker Engine 29.9.3, Compose 5.7.1.

La guía completa, con el porqué, está en la [Fase 5, paso 1](../../docs/05-servicios/01-monitorizacion.md). Aquí, solo los archivos.

## Archivos

- [`compose.yaml`](compose.yaml): el **hub** (la web), en `/srv/beszel/`. Usuario `beszel-hub`, sin capacidades, con `no-new-privileges`, imagen fijada por digest y sin socket de Docker. Va en las redes `default` y `panel`.
- [`.env.example`](.env.example): cópialo como `.env` con los UID y GID de `beszel-hub`.
- `datos/` (de `beszel-hub`, `700`): la base de datos del hub. **No va al repositorio.**
- [`beszel-agent.service`](beszel-agent.service): el **agente**, como servicio del sistema en `/etc/systemd/system/`. Es el del `.deb` oficial con `User=beszel` y más endurecimiento.
- [`beszel-agent.conf.example`](beszel-agent.conf.example): cópialo como `/etc/beszel-agent.conf` (de `beszel`, `600`) con la KEY y el TOKEN del hub.

El agente es el binario oficial (`beszel-agent_linux_amd64.tar.gz`), comprobado con su SHA-256 y copiado en `/usr/local/bin/`. **No se usa el `.deb`:** su `postinst` mete al usuario `beszel` en el grupo `docker`, que equivale a root.

## Puertos

| Fuera (servidor) | Dentro | Para qué | Quién llega |
|---|---|---|---|
| 8090 TCP | 8090 | La web | `<RED_LAN>` y la tailnet por `tailscale0` |

El filtro está en `DOCKER-USER`, con `homelab-beszel-lan.service`. El agente no escucha ningún puerto (`DISABLE_SSH=true`): se conecta él al hub en `127.0.0.1:8090`.

## Actualizar

Lee las [notas de la versión](https://github.com/henrygd/beszel/releases) y los [avisos de seguridad](https://github.com/henrygd/beszel/security/advisories). Cambia el digest del hub en `compose.yaml` y:
```bash
sudo docker compose -f /srv/beszel/compose.yaml pull
```
```bash
sudo docker compose -f /srv/beszel/compose.yaml up -d
```
Para el agente, descarga el `tar.gz` nuevo, comprueba su SHA-256 con el `checksums.txt` de la versión, sustituye `/usr/local/bin/beszel-agent` y `sudo systemctl restart beszel-agent`. No uses `beszel-agent update`: no puede escribir en `/usr/local/bin` (`ProtectSystem=strict`), y es mejor así. Cambia también las insignias del `README.md`.
