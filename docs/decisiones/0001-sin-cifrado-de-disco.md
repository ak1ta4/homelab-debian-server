# ADR 0001: no cifrar los discos ni reinstalar Debian

- **Estado:** aceptada (2026-11-02).
- **Decide:** el dueño, que acepta expresamente el riesgo físico.

## Contexto

La auditoría encontró el disco del sistema (NVMe) **sin cifrar**. En él vivirán los secretos de los servicios (`/etc/homelab/secrets/`), las bases de datos y el estado de las aplicaciones. Cifrarlo con LUKS exige reinstalar Debian. El disco de datos externo (SSD por USB) estaba vacío y podía cifrarse sin reinstalar. El portátil tiene TPM 2.0 (se activó en la BIOS), lo que permitiría desbloquear el cifrado al arrancar sin teclear nada.

## Opciones

| Opción | A favor | En contra |
|---|---|---|
| **A.** Reinstalar con LUKS en el NVMe y en el SSD, con TPM | Protege todo frente a robo o venta | Reinstalar desde cero; riesgo para el acceso actual; complejidad del desbloqueo sin pantalla |
| **B.** Cifrar solo el SSD de datos, con TPM | Protege fotos y archivos sin reinstalar | Los secretos y bases de datos del NVMe siguen legibles: protección parcial |
| **C.** Sin cifrado de disco | Sencillo, sin reinstalar, arranque sin intervención | Quien tenga los discos lo lee todo |

## Decisión

**Opción C.** No se cifra ni el NVMe ni el SSD, y no se reinstala Debian. El portátil no sale de casa.

## Consecuencias

- **Riesgo aceptado:** con los discos en la mano (robo, reparación, venta) se puede leer todo: secretos de servicios, bases de datos, fotos, el secreto TOTP del SSH y la clave de trabajo de las copias.
- **Compensaciones:**
  - Cifrado en red en todos los servicios (SSH, HTTPS, WireGuard).
  - Ningún secreto en el disco de datos: los *snapshots* de btrfs conservarían copias.
  - Las copias de restic van cifradas.
  - Antes de vender o reparar: borrado seguro de los discos y rotación de credenciales (en el runbook).
- El modelo de amenazas lo recoge como riesgo aceptado, no como riesgo cubierto.
- Contraseña de BIOS y Secure Boot pasan a ser opcionales: sin cifrado, no protegen los datos.
