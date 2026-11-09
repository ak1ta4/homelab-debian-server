# ADR 0005: gitleaks con configuración fuera del repositorio y workflow propio

- **Estado:** aceptada (2026-11-09).

## Contexto

El repositorio debe poder hacerse público algún día sin exponer nada. Por eso no puede contener secretos **ni datos identificativos** (IP, MAC, usuarios, dominio). Un agente de IA escribe buena parte del contenido. El encargo pide gitleaks como hook de pre-commit y en GitHub Actions.

## Opciones

| Opción | A favor | En contra |
|---|---|---|
| Framework `pre-commit` con el hook oficial | Estándar | Una dependencia más; la configuración vive en el repositorio |
| Hooks propios con `.gitleaks.toml` en el repositorio | Sencillo | Un cambio puede debilitar las reglas o añadir excepciones |
| **Hooks propios con configuración de root en `/etc/homelab`** | Ni el agente ni un parche pueden debilitarla; puede contener los valores reales de la casa | Las reglas de datos de la casa no viajan a GitHub |
| `gitleaks-action` en GitHub | Oficial | En la primera subida revisa desde un commit inexistente y no revisa nada; código bajo licencia comercial (gratis para cuentas personales) |
| **Workflow propio** con el binario oficial fijado por checksum | Misma versión que en local; revisa todo el historial; falla si no revisó nada | Hay que actualizar la versión a mano |
| GitGuardian (app de GitHub) | Avisos automáticos | Un tercero lee el repositorio privado |

## Decisión

- **En el servidor:** tres hooks en `/opt/homelab` (`pre-commit`, `pre-applypatch` y `pre-push`) con `--config /etc/homelab/gitleaks.toml`, `--gitleaks-ignore-path /etc/homelab` e `--ignore-gitleaks-allow`. La configuración amplía las reglas por defecto con la red de casa, MAC y los valores reales del inventario.
- **En GitHub:** workflow propio que instala gitleaks 8.30.1 comprobando su SHA-256 y revisa todo el historial. Exige `commits revisados >= git rev-list --count HEAD`. Acciones fijadas por commit.
- **Sin GitGuardian:** el dueño lo retiró de todos sus repositorios.

## Consecuencias

- Tres barreras locales y una remota. La remota solo conoce las reglas genéricas de secretos.
- gitleaks se actualiza a mano, en el servidor y en el workflow: va en el calendario. El proyecto está en modo de solo arreglos de seguridad; se vigilará su sucesor.
- Ninguna herramienta lo detecta todo: la revisión humana de cada parche sigue siendo obligatoria.
