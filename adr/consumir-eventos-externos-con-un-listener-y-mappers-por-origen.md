# Consumir eventos externos con un listener y mappers por origen

- Estado: propuesta
- Fecha: 2026-10-16
- Decisores: @juanimoli (Producto), @Matiamistadi y @luccaperazzo (Back), @KevinAlajarin (DevOps)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

[Usar Kafka como bus de eventos](usar-kafka-como-bus-de-eventos.md) dejó previsto que Emergencias consuma eventos de otros módulos, con deduplicación por `metadata.eventId` y una tabla de eventos procesados. Hasta hoy el backend solo publica: no existe ningún consumer.

En el Hito 2 aparecen funcionalidades que reaccionan a eventos de otros módulos:

- Un incendio informado por Tránsito y Movilidad requiere una respuesta de Bomberos (ver [Generar respuestas predeterminadas con reglas y templates por recurso](generar-respuestas-predeterminadas-con-reglas-y-templates-por-recurso.md)).
- Una reserva de espacio público o una alerta climática elevan temporalmente la prioridad de una zona (ver [Elevar temporalmente la prioridad con contextos de zona](elevar-temporalmente-la-prioridad-con-contextos-de-zona.md)).

La lista de módulos de origen va a crecer, y cada uno publica sus eventos con su propio formato. Un mismo evento externo puede tener más de un efecto en Emergencias.

Fuerzas en juego:

- La entrega del bus es *at-least-once*: el mismo evento puede llegar más de una vez.
- Si cada funcionalidad arma su propio consumer, duplica la conexión, la deduplicación y el manejo de errores.
- El formato de cada módulo de origen cambia por motivos que no controlamos y no debería filtrarse a nuestra lógica de negocio.
- Un consumer necesita un proceso *always-on*, que el hosting actual (Render free) no garantiza. Ver `docs/hosting-consumer-strategy.md` del backend.

## Decisión

- Vamos a tener un único paquete `integracion/` como **punto de entrada de todos los eventos externos**. El listener consume los tópicos suscritos a través de la abstracción `EventBus` del ADR de Kafka. Ninguna otra parte del backend consume del bus.
- Vamos a escribir **un mapper por cada par (módulo de origen, tipo de evento)**, registrado en un registro de mappers. El mapper traduce el evento externo a uno o más **comandos** de servicios de aplicación (por ejemplo, "solicitar respuestas para un incendio" o "dar de alta un contexto de zona"). Los mappers no contienen reglas de negocio ni acceden a tablas.
- Vamos a **deduplicar una sola vez, en el listener**, con la tabla `eventos_procesados` indexada por `metadata.eventId`. El registro del evento procesado y los efectos de todos sus comandos se guardan en la misma transacción. Si el `eventId` ya existe, el evento se descarta sin efectos.
- Vamos a confirmar el *offset* recién después de que esa transacción haga commit.
- Vamos a propagar el `metadata.eventId` del evento externo como **`causationId`** a todo lo que genere. Esto incluye los eventos que publiquemos como consecuencia, para que la cadena entre módulos sea trazable, ya que el bus no correlaciona eventos.
- Vamos a **registrar y descartar** los eventos sin mapper y los eventos inválidos, sin reintentos infinitos. Un evento con formato inesperado no puede frenar el consumo del tópico.
- Vamos a correr el listener en un proceso *always-on*. Dónde se aloja se decide aparte. Mientras no exista, `KAFKA_ENABLED=false` lo desactiva y los mismos comandos se pueden invocar a mano desde el tablero.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **Listener único + mapper por origen que emite comandos** | Un solo lugar para conexión, deduplicación y errores. Sumar un origen es sumar un mapper. Los mappers son puros y fáciles de testear. | Un punto de entrada común: un error en el listener afecta a todas las integraciones. |
| Un consumer por funcionalidad | Cada funcionalidad es independiente de las otras. | Duplica deduplicación y conexión. Un evento con dos efectos se procesa dos veces con transacciones separadas, y puede quedar a medias. |
| Consumer en un servicio aparte que llama a una API interna | Aísla la integración del backend. | Segundo deploy y segundo servicio que se duerme. Hay que exponer y proteger una API interna. Rompe la transacción entre deduplicación y efecto. |
| Handlers con `if/switch` sobre el tipo de evento, sin mappers | Lo más rápido de escribir. | Cada origen nuevo modifica código existente. El formato externo se mezcla con la lógica de negocio. |

## Consecuencias

Se hace más fácil:

- Integrar un módulo nuevo: se escribe su mapper y se registra, sin tocar el listener ni los servicios.
- Garantizar que un evento duplicado no tenga efectos, para todas las funcionalidades a la vez.
- Testear la integración sin Kafka: los mappers reciben un evento y devuelven comandos.
- Seguir una cadena entre módulos con `correlationId` + `causationId`.

Se hace más difícil:

- **Suma infraestructura.** El listener necesita un proceso *always-on* y un broker accesible. Mientras eso no se resuelva, las entradas automáticas no funcionan en producción y quedan solo las altas manuales desde el tablero.
- Hay que implementar la tabla `eventos_procesados` y su limpieza periódica.
- Dependemos del formato de cada módulo de origen: un cambio de su lado rompe su mapper.

Pendiente de definir:

- Hosting del proceso consumer (decisión aparte, a partir de `docs/hosting-consumer-strategy.md`).
- Formato de los eventos de Tránsito y Movilidad y de Espacios.
- Retención de la tabla `eventos_procesados`.

## Historial

- 2026-10-16: creación del ADR (estado: propuesta)
