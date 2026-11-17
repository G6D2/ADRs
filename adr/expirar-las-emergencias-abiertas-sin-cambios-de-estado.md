# Expirar las emergencias abiertas sin cambios de estado

- Estado: propuesta
- Fecha: 2026-11-17
- Decisores: Producto (@juanimoli), que lo pidió el 12/11 y lo implementó en el PR #97 del backend.
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Hasta el 15/11/2026, una emergencia que nadie cerraba quedaba abierta para siempre:

- seguía en el tablero del dispatcher;
- seguía activa para los otros módulos, que nunca recibían su cierre;
- conservaba la ubicación sin plazo, porque la retención de [Cifrar ubicación y PII sensible en reposo](cifrar-ubicacion-y-pii-sensible-en-reposo.md) cuenta desde el cierre (`closed_at`).

En producción se acumulaban emergencias de pruebas viejas. La migración V26 del backend dejaba escrito que "las PENDING viejas no se tocan (decisión de Producto)".

El PR #97 del backend agregó un job que las termina solas. Se mergeó el 15/11 a las 22:06 (16/11 en UTC) y revierte lo de V26, a pedido de Producto. El 13/11 se había propuesto dejarlo para después de la demo del 26/11; entró antes, con una excepción para las coberturas preventivas.

Cambia lo que ven los otros módulos y no pasó por este registro. Este ADR documenta lo que hace el código y queda en `propuesta` hasta que Producto lo confirme.

Fuerzas en juego:

- **Retención.** La ubicación es un dato sensible. Guardarla sin plazo va contra la minimización del ADR de cifrado.
- **Contrato del bus.** Los otros módulos ya tratan `CANCELADA` y `CERRADA` como estados finales. Un estado nuevo los obligaría a cambiar.
- **Render free.** El backend se duerme sin tráfico, y un job solo corre mientras está despierto.
- **Coberturas preventivas.** Se crean días antes del evento y nadie las toca hasta entonces (ver [Programar cobertura preventiva ante eventos y reservas grandes](programar-cobertura-preventiva-ante-eventos-y-reservas-grandes.md)).

## Decisión

Vamos a terminar sola toda emergencia abierta que pase el plazo sin cambios de estado, y a borrarle la ubicación en el mismo paso. Lo hace un job del backend: `src/services/emergency-expiration-job.js`, que llama a `EmergencyRepository.expireStaleEmergencies`.

### Cuándo expira

- La emergencia está en un estado no terminal: `PENDING`, `VALIDATED`, `DISPATCHED`, `EN_ROUTE`, `ON_SCENE` o `RESOLVED`.
- Su **último cambio de estado** tiene más que el plazo. Es la última fila de `emergency_status_history`, o la creación si nunca cambió. No se mide contra `updated_at`, que también lo mueven el job de incidentes y el backfill de barrios.
- **Excepción:** una cobertura preventiva con causa `EVENTO_PUBLICO` no expira mientras exista el contexto de zona de su evento (mismo id, no dado de baja) y ese evento no haya terminado hace más que el plazo. Su reloj corre desde el fin del evento (`contexto_zona.vigente_hasta`). Las preventivas sin contexto no tienen la excepción: las de `RESERVA`, las de un evento sin coordenadas y las que creó solo `ActividadCreada.v3`.

### A qué estado pasa

| Estado | Pasa a |
|---|---|
| `RESOLVED` | `CLOSED` |
| `PENDING`, `VALIDATED`, `DISPATCHED`, `EN_ROUTE`, `ON_SCENE` | `CANCELLED` |

No hay estado nuevo. Esta tabla (`EXPIRATION_TARGETS`, en `src/domain/emergency-status.js`) solo la usa el job. Por API, `DISPATCHED → CANCELLED` sigue dando 409.

### Qué hace

En una transacción por emergencia:

1. **Anonimiza la ubicación**, igual que la retención: borra la coordenada cifrada y la legacy y marca `location_anonymized_at`. El barrio se conserva; si faltaba, se deriva antes de borrar.
2. **Cambia el estado** y marca `closed_at`.
3. **Libera al oficial asignado:** si estaba `ocupado`, vuelve a `disponible`. La emergencia conserva `assigned_officer_id`.
4. **Deja historial** con actor `sistema:expiracion` y motivo `Expirada: sin cambios de estado en 72 h` (el número es el plazo vigente).
5. **Encola un `EmergenciaActualizada`** en el outbox.

Después del lote, borra los vínculos de esas emergencias con contextos de zona (y sus eventos de vínculo que sigan en el outbox) y pone en `null` las coordenadas de sus eventos guardados en el outbox.

Cada emergencia va en su propia transacción, con `FOR UPDATE SKIP LOCKED` y la condición repetida adentro. Una que falla no frena al resto, y dos instancias no expiran dos veces la misma.

### Qué se publica

Un `EmergenciaActualizada` (`com.citypass.emergencias.EmergenciaActualizada`, por el gateway) con:

- `estadoAnterior` (por ejemplo `EN_CAMINO`) y `estadoNuevo` `CANCELADA` o `CERRADA`;
- `usuarioId: "sistema:expiracion"` y `resolucion: "Expirada: sin cambios de estado en 72 h"`;
- `latitud`, `longitud`, `ubicacion.latitud` y `ubicacion.longitud` en `null`, porque el evento se arma después de anonimizar. El `barrio` se mantiene;
- `asignadoA: null`, como en todo cambio que no es a `DESPACHADA`.

No hay un tipo de evento propio. Un consumidor reconoce la expiración por `usuarioId` o por `resolucion`.

### Plazo y configuración

| Variable | Para qué | Default |
|---|---|---|
| `EMERGENCY_EXPIRATION_HOURS` | Plazo sin cambios de estado. | `LOCATION_RETENTION_HOURS`; si tampoco está, 72. |
| `EMERGENCY_EXPIRATION_ENABLED` | Con `false` el job no corre. | Prendido. |
| `EMERGENCY_EXPIRATION_INTERVAL_MS` | Cada cuánto corre. | 1 h. |

- `render.yaml` no define ninguna de las tres. El plazo de producción sale de `LOCATION_RETENTION_HOURS=72`, que sí está: la ubicación de una emergencia abandonada dura lo mismo que la de una cerrada.
- La primera pasada es a los 15 s del arranque. Cada pasada toma hasta 100 emergencias, las más viejas primero, y no se superpone con la anterior.

## Alternativas consideradas

| Alternativa | A favor | En contra |
|---|---|---|
| No expirar (lo que decía V26) | No se pierde nada. | Emergencias abiertas para siempre, en el tablero y para los otros módulos, con la ubicación sin plazo. |
| Estado nuevo `EXPIRADA` | Se distingue a simple vista. | Cambia el contrato del bus, el front y la base. Los consumidores no lo conocen. |
| **`CANCELLED` y `CLOSED`, que ya existen (elegida)** | No cambia el contrato, ni el front, ni la base. | La expiración solo se distingue por `usuarioId` y `resolucion`. |
| Solo anonimizar, sin terminarla | Cumple la retención. | Queda abierta una emergencia sin ubicación, que nadie puede atender y que sigue activa para los demás. |
| Medir desde `updated_at` | Más simple. | Lo mueven jobs que no cambian el estado, y eso posterga la expiración de una emergencia que nadie atendió. |

## Consecuencias

### Esperadas

Se hace más fácil:

- Mantener el tablero sin emergencias abandonadas.
- Tener una sola regla de retención: la ubicación dura 72 h desde el último cambio de estado, esté la emergencia cerrada o no.
- Que los otros módulos den de baja incidentes que nadie cerró.
- Recuperar oficiales que quedaron `ocupado` en emergencias olvidadas.

Se hace más difícil o cuesta:

- **No se puede deshacer.** La ubicación borrada no vuelve, y un estado terminal no tiene salida.
- **Se cancela también lo despachado.** Los consumidores reciben `DESPACHADA`, `EN_CAMINO` o `EN_LUGAR` → `CANCELADA`, una transición que por API no existe.
- **Llega en tandas.** El job solo corre con el servicio despierto. Con Render free, lo que venció mientras dormía expira junto al despertar (por ejemplo, al empezar la clase del jueves): hasta 100 por pasada, con un `EmergenciaActualizada` por cada una. Al desplegar el #97 expiró de una vez todo lo que ya estaba vencido.
- **El plazo está atado a la retención.** Cambiar `LOCATION_RETENTION_HOURS` cambia también la expiración, salvo que se cargue `EMERGENCY_EXPIRATION_HOURS`.

Neutras:

- La tabla de transiciones de los endpoints no cambia.
- La retención de las emergencias que cierra una persona sigue igual.

### Para los otros módulos

- Pueden recibir cancelaciones y cierres que no hizo una persona, sin coordenadas. Se reconocen por `usuarioId = sistema:expiracion`.
- Un consumidor que cuente cancelaciones o cierres (Analítica, por ejemplo) los mezcla con los del sistema si no filtra por `usuarioId`.
- Un consumidor que siga nuestros estados tiene que aceptar `CANCELADA` después de `DESPACHADA`, `EN_CAMINO` y `EN_LUGAR`.

### No esperadas (riesgos a vigilar)

- **Oficial en camino.** El oficial vuelve a `disponible` y la emergencia queda cancelada sin aviso, ni para él ni para el dispatcher.
- **Sin aviso previo.** Nadie ve que una emergencia está por expirar.
- **Coberturas preventivas sin contexto.** Expiran a las 72 h de creadas, aunque el evento sea después, y la deduplicación por causa impide que se vuelvan a crear. Las 11 que se crearon el 16/11 a las 19:31 expiran el jueves 19/11 a partir de las 19:31 si nadie las toca y no tienen contexto.
- **Botón de pánico.** Un pánico que nadie atendió en 72 h termina como `CANCELADA`, y así lo ve el ciudadano.
- **El ADR de cifrado** describe la retención "después de cerradas". Es un ADR aceptado y no se edita: este deja registro de que una emergencia abierta también pierde la ubicación cuando expira.

### Preguntas abiertas para Producto

- [ ] ¿Expiramos una emergencia con oficial asignado (`DISPATCHED`, `EN_ROUTE`, `ON_SCENE`)? Las opciones son dejarlas afuera, avisarle al dispatcher antes o seguir como está.
- [ ] ¿Está bien que una `RESOLVED` se cierre sola, o el cierre lo tiene que hacer el dispatcher?
- [ ] ¿Avisamos antes de expirar, por ejemplo marcándola en el tablero a las 48 h?
- [ ] ¿El plazo sigue atado a `LOCATION_RETENTION_HOURS` o pasa a tener su propia variable en `render.yaml`?
- [ ] ¿Las coberturas preventivas sin contexto de zona tienen que contar el plazo desde el fin del evento?
- [ ] ¿Les avisamos a los otros módulos que pueden llegar cancelaciones del sistema desde `DESPACHADA` en adelante?

## Relacionados

- [Cifrar ubicación y PII sensible en reposo](cifrar-ubicacion-y-pii-sensible-en-reposo.md)
- [Publicar coordenadas aproximadas en claro en los eventos del bus](publicar-coordenadas-aproximadas-en-claro-en-los-eventos-del-bus.md)
- [Programar cobertura preventiva ante eventos y reservas grandes](programar-cobertura-preventiva-ante-eventos-y-reservas-grandes.md) (propuesta en otro PR)
- [Usar Kafka como bus de eventos](usar-kafka-como-bus-de-eventos.md)

## Historial

- 2026-11-17: creación del ADR (estado: propuesta). Documenta el job de expiración que está en producción desde el PR #97 del backend, que entró sin ADR.
