# ADR 0004: tapar el SSH del router con un reenvío a una IP sin uso

- **Estado:** aceptada (2026-11-08). Es una **excepción documentada** a la regla "sin reenvío de puertos".
- **Decide:** el dueño.

## Contexto

La revisión del router (reenvíos, DMZ y UPnP desactivados) y una prueba exterior de los puertos de Sunshine se dieron por buenas. Después, un antivirus del PC avisó de un puerto 22 abierto en el router. Se comprobó:
- Desde casa, el router acepta TCP 22 (servidor SSH Dropbear).
- La huella SSH era la misma por la IP interna y por la pública: respondía **el propio router**.
- Desde datos móviles, la conexión llegaba hasta pedir contraseña: **expuesto a internet**.

El portal del router no tiene ninguna opción para desactivar ese servicio. El panel web del router no respondía desde fuera.

## Opciones

| Opción | A favor | En contra |
|---|---|---|
| Pedir a la operadora que lo cierre | Se cierra en origen | Sin garantía; puede volver con una actualización; el dueño prefiere gestionarlo él |
| **Reenviar el TCP 22 de internet a una IP de la LAN sin uso** | Inmediato, lo gestiona el dueño, reversible con un clic | Depende de que esa IP nunca se use; la operadora podría borrar la regla |
| Router propio con el de la operadora en modo puente | Control total; el router de la operadora deja de tener la IP pública | Cambio grande; reconfigurar TV y telefonía de la operadora |
| Entrar por SSH al router y apagar el servicio | — | Sin soporte, puede volver al reiniciar o romper el router |
| Solo contraseña fuerte | Fácil | El puerto sigue expuesto |

## Decisión

Regla de **Port Mapping** en la interfaz de internet: TCP 22, cualquier origen, hacia `<IP_SUMIDERO>:22`, una IP fuera de todos los rangos DHCP y comprobada sin respuesta. Verificado desde datos móviles: ya no se establece la conexión.

Además: cambiar la contraseña de administración del router, que el dueño hace por su cuenta. El router propio se valorará más adelante.

## Consecuencias

- **No asignar nunca `<IP_SUMIDERO>`** a ningún equipo: su puerto 22 quedaría expuesto a internet. Anotado en el inventario local y en el glosario.
- El SSH del router sigue abierto **dentro** de casa; lo protege su contraseña.
- Repetir la prueba desde fuera en el calendario periódico y en el escaneo externo de la Fase 8: la operadora podría borrar la regla.
- Un honeypot en esa IP se descartó: sería el primer servicio de casa expuesto a internet. Si interesa como proyecto, mejor en un VPS fuera de casa.
- **Lección:** la revisión inicial de exposición solo cubrió los puertos del servidor. Hay que probar también **los servicios del propio router**.
