# Paso 1: reservas DHCP del servidor y del PC

> **Específico de esta red:** router de operadora con portal avanzado. Los nombres de menú cambian en otros routers, pero casi todos tienen algo llamado "reserva DHCP", "DHCP estático" o *static lease*.

## 1. Qué vamos a hacer y por qué

**En sencillo:** hasta ahora el router prestaba a cada equipo una dirección que podía cambiar, como un buzón distinto cada semana. Le pedimos que el servidor y el PC tengan **siempre la misma**. Pasó de verdad: el PC cambió de dirección entre la auditoría y este paso.

**Técnicamente:** se crean dos *static leases* que asocian la MAC de cada equipo a su IP. El SSH, Fail2ban (que nunca debe bloquear el PC de rescate), el firewall, AdGuard (Fase 4) y las copias (Fase 6) dependen de esas IP.

## 2. Conceptos previos

DHCP, reserva DHCP y MAC: ver [`../00-conceptos.md`](../00-conceptos.md#dhcp-y-reserva-dhcp).

## 3. Requisitos y estado inicial

- Acceso de administrador al portal avanzado del router.
- Saber qué IP tiene ahora cada equipo y su MAC.
- Antes de nada, mirar los rangos DHCP del router: aquí hay **tres** (uno para los equipos de casa, otro para los decodificadores de TV de la operadora y otro para sus aparatos IoT). **No se toca ninguno.**

## 4. Pasos

### 4.1 MAC del servidor (en el servidor, cualquier usuario)

```bash
cat /sys/class/net/eno1/address
```
MAC de la tarjeta por cable (`eno1`). Si tu interfaz se llama distinto, mírala con `ip -br link`.

### 4.2 MAC y IP del PC (en el PC con Windows, Símbolo del sistema)

```
ipconfig /all
```
Busca el adaptador **Ethernet** que corresponde a la tarjeta física (aquí, Intel I225-V) y apunta su **Dirección física** y su **Dirección IPv4**. Ignora los adaptadores virtuales (VPN, VirtualBox, Bluetooth).

> **Cuidado:** al principio se tomó la MAC que el servidor veía en la IP que el informe atribuía al PC, y resultó ser de **otro aparato** con MAC aleatoria. El PC había cambiado de IP. Saca siempre la MAC del propio equipo.

### 4.3 Crear las reservas (navegador, portal del router)

1. **LAN Setting → LAN DHCP**, pulsa el número del rango de los equipos de casa.
2. Pestaña **Static Lease → Add**:
   - **Adm.State:** Enable.
   - **MAC Address:** la del equipo, un par de caracteres en cada casilla.
   - **IP Address:** la IP **que ya tiene**, para no cortarle la conexión.
3. **Apply**. Repite para el otro equipo.
4. Comprueba que las dos reservas aparecen en la lista de **Static Lease**.

No uses la pestaña **Reserved IP**: sirve para que el router *no reparta* ciertas IP, no para fijarlas.

## 5. Verificación

- Las dos reservas aparecen en **Static Lease**.
- **Prueba real:** la reserva solo se nota cuando el equipo renueva la IP. Se comprueba en el próximo reinicio de cada uno: la IP debe ser la reservada (`ip -br address` en el servidor, `ipconfig` en el PC). **Pendiente** hasta ese reinicio.

## 6. Errores frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| La reserva no tiene efecto | MAC mal copiada, o MAC aleatoria (habitual en móviles y en el Wi-Fi de algunos portátiles) | Copia la MAC del propio equipo; en móviles, desactiva la dirección privada para esa red si necesitas reserva |
| El equipo pierde la conexión al guardar | La IP reservada es distinta de la actual y el equipo renovó | Reserva la IP que ya tiene, o reinicia la conexión del equipo |
| Tras cambiar el PC, Fail2ban lo bloquea | La IP de rescate ya no es la suya | Actualiza la reserva y `ignoreip` en Fail2ban ([guía del SSH](02-ssh-de-emergencia.md)) |

## 7. Cómo revertirlo

En **Static Lease**, marca la reserva y pulsa **Del**. El equipo seguirá con su IP hasta que la renueve.

## 8. Seguridad

- **Mitiga:** reglas de firewall y exclusiones de Fail2ban que apunten a una IP que ya no es la del equipo correcto.
- **Queda:** cualquiera en la red de casa puede cambiar su MAC por la del PC y recibir su IP, y con ello la exclusión de Fail2ban. Por eso esa exclusión solo ahorra bloqueos: entrar sigue exigiendo clave y código.

## 9. Para la entrevista

**En 30 segundos:** "Fijé las IP del servidor y del PC con reservas DHCP en el router, no con IP estáticas en los equipos, para que el router siga siendo la única fuente de verdad y no haya conflictos. Lo hice antes que el SSH porque Fail2ban y el firewall dependen de esas IP."

**Preguntas posibles:**
- *¿Reserva DHCP o IP estática?* Reserva: se gestiona en un solo sitio y el router no puede dar esa IP a otro. Una estática en el equipo funciona igual, pero si el router no la conoce puede repartirla y causar un conflicto.
- *¿Una reserva es una medida de seguridad?* No. Es estabilidad. La MAC se puede falsificar, así que nada de seguridad debe depender solo de la IP.
