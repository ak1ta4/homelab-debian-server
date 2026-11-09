# Preguntas de entrevista

Preguntas generales sobre el diseño. Cada guía tiene además su propio apartado "Para la entrevista".

## El proyecto en 30 segundos

"Monté un homelab en un portátil con Debian: archivos, notas, fotos, DNS con filtrado para toda la casa y acceso remoto, todo con Docker y **sin exponer nada a internet**. Lo documenté para poder reconstruirlo a mano. Empecé con una auditoría de solo lectura, que encontró el SSH del router abierto a internet. Puse un acceso de emergencia por SSH con clave, TOTP y Fail2ban antes de tocar nada delicado. Y lo administro con un agente de IA que trabaja en una cuenta sin privilegios y cuyos cambios reviso yo."

## Preguntas

**¿Por qué nada expuesto a internet?**
Porque es la medida que más superficie de ataque elimina. El acceso desde fuera irá por Tailscale (WireGuard), con ACL de mínimo privilegio. La única regla de reenvío del router sirve para **tapar** un servicio del propio router.

**¿Cuál es tu modelo de amenazas?**
Robo del portátil, dispositivo de casa comprometido, clave o token filtrados, contenedor vulnerable, ransomware y error del agente de IA. Cada una tiene un control o un riesgo aceptado por escrito. Por ejemplo, el robo físico es un **riesgo aceptado**: los discos no van cifrados ([ADR 0001](decisiones/0001-sin-cifrado-de-disco.md)).

**¿Cómo limitas lo que puede hacer el agente de IA?**
Con el sistema operativo, no con instrucciones:
- Cuenta propia sin `sudo` ni `docker` y sin acceso a mi carpeta.
- Configuración de Claude Code propiedad de root que prohíbe el modo sin permisos.
- Sin credenciales de GitHub.

Hice pruebas negativas para comprobarlo. Lo que queda abierto, la red de salida, está documentado y se tratará con el firewall ([ADR 0002](decisiones/0002-permisos-del-agente-de-ia.md)).

**¿Cómo evitas no quedarte sin acceso al cambiar algo delicado?**
- Un segundo acceso probado antes del cambio.
- Copia del archivo y forma de revertirlo escrita.
- Rollback automático con temporizador al activar el firewall.
- Para SSH: una sesión abierta de respaldo, `sshd -t` antes de `reload` y la prueba en otra ventana.

**¿Qué haces si se filtra un secreto en el repositorio?**
Rotarlo primero, en el servicio correspondiente. Limpiar el historial no basta, porque lo publicado puede estar copiado. Para que no pase, gitleaks revisa en local en tres puntos y otra vez en GitHub.

**¿Qué error te enseñó más?**
Un control que "pasaba" sin revisar nada. La acción de gitleaks en GitHub no revisaba nada en la primera subida, y la herramienta podía terminar con éxito tras un fallo de git. Desde entonces compruebo que cada control falle cerrado y que haya hecho su trabajo ([lecciones](lecciones-aprendidas.md#gitleaks-action-y-la-primera-subida)).
