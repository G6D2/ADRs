# Programar cobertura preventiva ante eventos y reservas grandes

- Estado: aceptada
- Fecha: 2026-11-17
- Decisores: Producto (@juanimoli). Lo implementaron en el backend @KevinAlajarin (PR #72 y #119) y @Matiamistadi (PR #107).
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Un evento masivo cambia el riesgo de una zona. El 26/10/2026 Producto eligió **"las dos"** para los eventos masivos del caso de reserva de espacio:

- **Contexto de zona:** mientras dura el evento, las emergencias que se crean cerca suben un nivel de prioridad. Está en [Elevar la prioridad en un radio alrededor del evento](elevar-la-prioridad-en-un-radio-alrededor-del-evento.md).
- **Cobertura preventiva** para los eventos grandes: Emergencias programa presencia antes de que pase algo.

La segunda no tiene ADR. Entró con el PR #72 del backend (05/11) y creció con el #107 (`ActividadCreada.v3`, 16/11) y el #119 (`ReservaCreada`, 16/11). Lo único que dice el registro es, en el ADR de radio, que una preventiva "no sube".

Este ADR registra lo que corre en `main` al 17/11/2026. Entró sin pasar por el registro; Producto lo confirmó el 17/11 (ver las preguntas al final).

Fuerzas en juego:

- **Espacios publica el mismo evento varias veces.** Republica `EventoComunitarioPublicado` cada vez que cambian los inscriptos, y además publica `ActividadCreada.v3` con el mismo id (`actividadId` = `eventoId`). Y la entrega del bus es *at-least-once*.
- **Un listener nuevo lee su tópico desde el principio** (desde `EDA_CONSUMER_SINCE`). En su primer arranque procesa todo lo publicado, también lo viejo.
- **`ActividadCreada` no es solo de eventos.** Espacios también la publica cuando un vecino reserva un espacio (`organizadorTipo: CIUDADANO`). La baja de esas reservas viaja en `ActividadModificada`, que no consumimos.
- **La única baja que llega es `EventoCancelado`**, y es de eventos comunitarios. Las reservas de espacio no tienen cancelación en Espacios.
- **El backend duerme** (Render free). Al despertar lee de una vez lo pendiente de todos los tópicos.

## Decisión

Vamos a crear sola una emergencia `PREVENTIVA` cuando Espacios anuncia un evento, una actividad o una reserva de espacio de 500 personas o más. El dispatcher la trata como cualquier emergencia pendiente.

### Disparadores

| Evento de Espacios | Listener | Cubre si | No cubre | Causa (`causaTipo` / `causaId`) |
|---|---|---|---|---|
| `EventoComunitarioPublicado` (tópico en `EDA_TOPIC_EVENTO_PUBLICADO`) | `EventoComunitarioPublicadoListener` (#72) | `capacidad` (o `cupoMaximo`) ≥ `UMBRAL_CAPACIDAD_COBERTURA_PREVENTIVA` | Eventos con estado cancelado. | `EVENTO_PUBLICO` / `eventoId` |
| `ActividadCreada.v3` (tópico por defecto; `EDA_TOPIC_ACTIVIDAD_CREADA` lo cambia) | `ActividadCreadaListener` (#107) | `capacidad` ≥ `UMBRAL_CAPACIDAD_COBERTURA_PREVENTIVA` | Actividades terminadas (`fechaHoraFinalizacion` pasada), organizadas por un ciudadano (`organizadorTipo: CIUDADANO`) o canceladas. | `EVENTO_PUBLICO` / `actividadId` |
| `ReservaCreada` (opt-in: sin `EDA_TOPIC_RESERVA_CREADA` no se consume) | `ReservaCreadaListener` (#119) | `tipoReserva: ESPACIO` y `cantidadPersonas` ≥ `UMBRAL_PERSONAS_COBERTURA_RESERVA` | Inscripciones a eventos (`tipoReserva: EVENTO`) y reservas terminadas (`fechaFin` pasada). | `RESERVA` / `reservaId` |

- Los dos umbrales valen 500 por defecto. El primero lo comparten eventos y actividades, así las dos entradas usan un solo criterio. Las reservas tienen el suyo.
- Sin fecha de fin legible, o sin `organizadorTipo`, se cubre.
- Una inscripción a un evento no se cubre porque la cobertura del evento ya la programan los otros dos disparadores.
- Las tres reglas emiten el mismo resultado (`ProgramarCobertura`) y comparten el handler.

### Qué se crea

Una emergencia de origen externo (`source = EXTERNAL`), sin ciudadano, con:

- tipo `PREVENTIVA`, estado `PENDING` y severidad `low`, la que corresponde al tipo;
- descripción `Cobertura preventiva — <título> (<N> asistentes esperados)`; en una reserva, el título es `Reserva de espacio (<categoría>)`;
- la ubicación del evento, cifrada como la de cualquier emergencia, y el barrio que sale de ella. Sin coordenadas queda `requiresManualValidation = true` para que el dispatcher la revise;
- `causaTipo` y `causaId` del disparador;
- el `correlationId` de Espacios si es un UUID; si no, el id de la emergencia.

El alta pasa por el repositorio, en la transacción del listener: la emergencia y su `EmergenciaCreada` en el outbox se guardan juntas.

### Qué ve el dispatcher

En el tablero aparece como una emergencia más:

- estado **PENDIENTE** y prioridad **baja**;
- como título, la descripción. Por ejemplo: "Cobertura preventiva — Festival de Folklore (5000 asistentes esperados)";
- se valida, se asigna y se cierra con el flujo de siempre.

Además:

- No sube de prioridad por contexto de zona: la cobertura del propio evento no es un llamado.
- No entra en la detección de incidentes.
- Un ciudadano no puede reportar una `PREVENTIVA`: la crea solo el sistema.

### Una sola cobertura por causa

- Antes de crear, el handler busca una `PREVENTIVA` con la misma causa (`causaTipo` + `causaId`), con la fila bloqueada. Si existe, no crea nada.
- En la base lo garantiza el índice único parcial `uq_emergencies_preventiva_causa`: (`causa_tipo`, `causa_id`) donde `emergency_type = 'PREVENTIVA'` (migración V23).
- Por eso `EventoComunitarioPublicado` y `ActividadCreada.v3` del mismo evento crean una sola cobertura, en cualquier orden. Una `RESERVA` no choca con un `EVENTO_PUBLICO` del mismo id.
- La búsqueda no mira el estado: una cobertura cancelada o expirada también frena una nueva para la misma causa.
- Antes de todo eso, la deduplicación por `metadata.eventId` del listener descarta las reentregas del mismo mensaje.

### Baja

`EventoCancelado` (tópico en `EDA_TOPIC_EVENTO_CANCELADO`) cancela las coberturas `EVENTO_PUBLICO` de ese id que siguen en `PENDING` o `VALIDATED`. Alcanza también a las de `ActividadCreada.v3`, que usan el mismo id.

- Deja historial con actor `eda:espacios` y motivo "Evento comunitario cancelado". No copia el motivo del organizador, que es texto libre.
- Publica `EmergenciaActualizada` con `CANCELADA`.
- El mismo evento da de baja el contexto de zona.
- Si la cobertura ya se despachó, no la toca: queda en el log y la resuelve el dispatcher.

No hay baja para las reservas de espacio, porque Espacios no tiene ese flujo, ni para las actividades suspendidas o modificadas, porque `ActividadModificada` no se consume.

### Qué se publica

Sale un `EmergenciaCreada`, como en cualquier alta, con:

- `categoria: PREVENTIVA`, `prioridad: BAJA` y `estado: PENDIENTE`;
- `causaTipo` (`EVENTO_PUBLICO` o `RESERVA`) y `causaId`, con la convención de Analítica para las causas externas. Según el PR #119, Analítica mide `RESERVA` como la métrica c6 de su tablero de Emergencias;
- `afectaCirculacion: true`, `ciudadanoId: null` y las coordenadas aproximadas y cifradas, como cualquier emergencia.

Los cambios de estado posteriores salen como `EmergenciaActualizada`.

### Configuración

| Variable | Default | En producción (17/11) |
|---|---|---|
| `EDA_TOPIC_EVENTO_PUBLICADO`, `EDA_TOPIC_EVENTO_CANCELADO` | Sin default. | Cargadas en `render.yaml`. |
| `EDA_TOPIC_ACTIVIDAD_CREADA` | `com.citypass.espacios.ActividadCreada.v3` | Usa el default. |
| `EDA_TOPIC_RESERVA_CREADA` | Sin default (opt-in). | Cargada en el dashboard: `/health` muestra el listener corriendo. No figura en `render.yaml`. |
| `UMBRAL_CAPACIDAD_COBERTURA_PREVENTIVA` | 500 | No está en `render.yaml`. |
| `UMBRAL_PERSONAS_COBERTURA_RESERVA` | 500 | No está en `render.yaml`. |

## Alternativas consideradas

| Alternativa | A favor | En contra |
|---|---|---|
| Solo contexto de zona (lo decidido el 08/10) | No suma trabajo al dispatcher. Ya existe. | Nada asegura presencia en un evento de miles de personas: la suba solo ordena emergencias que ya se reportaron. |
| Solo cobertura preventiva | Presencia antes de que pase algo. | Las emergencias reales cerca del evento no suben. |
| **Las dos (elegida por Producto el 26/10)** | Presencia en los eventos grandes y mejor orden cerca de todos. | Dos mecanismos para el mismo evento. El tablero suma emergencias que no son llamados. |
| Umbral de 1.000 personas (primera versión del #107) | Menos coberturas. Es el número que usan otros módulos del caso para clasificar la causa. | Los eventos de 500 a 999 quedan sin cobertura. En la review del #107, Producto marcó que la variable es compartida y subirla los dejaba afuera; quedó en 500. |

## Consecuencias

### Esperadas

Se hace más fácil:

- Tener presencia en los eventos grandes sin que nadie mire la agenda de Espacios.
- Que un evento genere una sola cobertura, aunque Espacios lo republique o lo mande por dos tipos.
- Que Analítica cuente coberturas por causa.

Se hace más difícil o cuesta:

- **El tablero suma emergencias que no son llamados.** Llegan con prioridad baja, pero ocupan la cola de pendientes.
- **Ráfagas.** Cuando un listener nuevo lee su tópico por primera vez, crea de una vez la cobertura de cada evento grande que encuentra. El 16/11 a las 19:31, al entrar el #107 en producción, se crearon 11 coberturas `PENDING` en nueve segundos. El 11/11 ya había pasado con 8, que hubo que descartar a mano. Los filtros de actividades y reservas terminadas sacan lo viejo, pero no los eventos futuros.
- **Dos mecanismos para el mismo evento** (contexto y cobertura), con reglas que hay que mantener alineadas.

Neutras:

- Las otras emergencias no cambian: la cobertura no sube ni baja la prioridad de nadie.

### No esperadas (riesgos a vigilar)

- **Cancelación antes que la publicación** (issue #88 del backend, abierto). Si el `EventoCancelado` de un evento se procesa antes que su publicación, no encuentra cobertura, y la publicación tardía crea una `PENDING` de un evento cancelado. Con un solo consumer sigue pudiendo pasar: los tópicos se procesan en paralelo, sin orden entre ellos. El contexto de zona sí lo resuelve (la baja deja una marca); la cobertura no.
- **El mismo evento con distinto id.** La deduplicación es por id. Entre las 11 del 16/11 hay tres pares con el mismo título y distinta capacidad.
- **Una cobertura despachada de un evento cancelado** sigue abierta hasta que el dispatcher la cierre.
- **Expiración.** Con [Expirar las emergencias abiertas sin cambios de estado](expirar-las-emergencias-abiertas-sin-cambios-de-estado.md), una cobertura `EVENTO_PUBLICO` con contexto de zona vigente no expira antes de que termine su evento. Las que no tienen contexto (todas las de `RESERVA`, las de un evento sin coordenadas y las de una actividad sin `EventoComunitarioPublicado`, porque `ActividadCreada.v3` no registra contexto) expiran a las 72 h de creadas, aunque el evento sea después. Y por la deduplicación no se vuelven a crear.
- **Un tópico roto frena a todos.** Con un solo consumer, si `ActividadCreada.v3` no se puede leer no lee ningún listener, y su tópico tiene default en el código. Ver [Consumir todos los eventos externos con un solo consumer](consumir-todos-los-eventos-externos-con-un-solo-consumer.md).

### Preguntas abiertas para Producto

- [x] ¿500 personas es el umbral correcto para los tres disparadores? **Sí, 500 para los tres** (Producto, 17/11).
- [x] ¿La reserva de un vecino de 500 personas o más lleva cobertura? **No, sigue sin cobertura** (Producto, 17/11). Hoy no. Si la lleva, hay que consumir `ActividadModificada` para la baja.
- [x] ¿Aceptamos que una actividad suspendida no baje su cobertura? **Sí, como limitación conocida: si hace falta, la descarta el dispatcher** (Producto, 17/11).
- [x] ¿Una cobertura sin contexto de zona tiene que contar el plazo de expiración desde el fin del evento, como las otras? **No por ahora: cuenta desde la creación, como corre hoy** (Producto, 17/11). Si nadie las toca y no tienen contexto, las 11 del 16/11 expiran el jueves 19/11 a partir de las 19:31, en horario de clase.
- [x] ¿Se arregla el #88 antes de la demo del 26/11 o queda como riesgo aceptado? **Se arregla antes del 26/11** (Producto, 17/11).

## Relacionados

- [Elevar la prioridad en un radio alrededor del evento](elevar-la-prioridad-en-un-radio-alrededor-del-evento.md)
- [Consumir eventos externos con un listener y reglas por evento](consumir-eventos-externos-con-un-listener-y-reglas-por-evento.md)
- [Consumir todos los eventos externos con un solo consumer](consumir-todos-los-eventos-externos-con-un-solo-consumer.md) (PR #42)
- [Expirar las emergencias abiertas sin cambios de estado](expirar-las-emergencias-abiertas-sin-cambios-de-estado.md) (PR #41)
- [Publicar coordenadas aproximadas en claro en los eventos del bus](publicar-coordenadas-aproximadas-en-claro-en-los-eventos-del-bus.md)

## Historial

- 2026-11-17: creación del ADR (estado: propuesta). Documenta la cobertura preventiva tal como corre en producción (PR #72, #107 y #119 del backend), que entró sin ADR.
- 2026-11-17: cambio de estado: propuesta → aceptada
