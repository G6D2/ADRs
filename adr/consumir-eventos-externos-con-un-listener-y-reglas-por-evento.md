# Consumir eventos externos con un listener y reglas por evento

- Estado: propuesta
- Fecha: 2026-10-16
- Decisores: @juanimoli (Producto), @Salterm27, @Matiamistadi y @luccaperazzo (Back), @KevinAlajarin (DevOps)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

[Usar Kafka como bus de eventos](usar-kafka-como-bus-de-eventos.md) dejó previsto que Emergencias consuma eventos de otros módulos, con deduplicación por `metadata.eventId` y una tabla de eventos procesados. En `main` ya está el esqueleto del consumidor (`src/listeners/`), apagado por defecto y sin reglas.

En el Hito 2 aparecen funcionalidades que reaccionan a eventos de otros módulos:

- Reclamos publica `com.citypass.reclamos.ReclamoCreado`. Un auto sobre una ciclovía genera una intervención en Emergencias.
- Movilidad publica `com.citypass.movilidad.EmergenciaBicicleta` cuando alguien pide ayuda durante un viaje, con las condiciones del triage (ver [Calcular los recursos requeridos con una tabla de reglas](calcular-los-recursos-requeridos-con-una-tabla-de-reglas.md)).
- Espacios publica `com.citypass.espacios.EventoComunitarioPublicado` y `com.citypass.espacios.CriticidadClimaticaConfirmada`, que elevan temporalmente la prioridad de una zona (ver [Elevar temporalmente la prioridad con contextos de zona](elevar-temporalmente-la-prioridad-con-contextos-de-zona.md)).

La lista de módulos de origen va a crecer, y cada uno publica sus eventos con su propio formato. Un mismo evento externo puede tener más de un efecto en Emergencias.

Fuerzas en juego:

- La entrega del bus es *at-least-once*: el mismo evento puede llegar más de una vez, y no siempre en orden.
- Si cada funcionalidad arma su propio consumer, duplica la conexión, la deduplicación y el manejo de errores.
- El formato de cada módulo de origen cambia por motivos que no controlamos y no debería filtrarse a nuestra lógica de negocio.
- El consumidor solo lee mientras el proceso está despierto, y Render free duerme el servicio tras unos 15 minutos sin tráfico.

## Decisión

- Vamos a tener un único paquete `src/listeners/` como **punto de entrada de todos los eventos externos**. Hay un `EventListener` concreto por tipo de evento, agrupado por caso de negocio. Ninguna otra parte del backend consume del bus.
- Vamos a resolver cada evento con un **Template Method**: `handle(event)` arma el contexto con `buildContext(event)`, que traduce el formato externo, evalúa las reglas del listener y devuelve los resultados `{ type, payload }` de todas las que aplican.
- Vamos a escribir **una regla por archivo** (`rules/*.rule.js`, subclase de `BusinessRule`), que se carga sola al arrancar. Las reglas no tienen efectos laterales ni acceden a tablas. Un archivo inválido hace fallar el arranque, no cada mensaje.
- Vamos a usar una **factory** que convierte cada resultado en una llamada a un servicio de aplicación. Los servicios escriben en la base y en el outbox, con su historial y validando las transiciones. Nada escribe SQL directo sobre `emergencies` desde el paquete de listeners.
- Vamos a **deduplicar una sola vez, en el listener**, con la tabla `eventos_procesados` indexada por `metadata.eventId`. El registro del evento procesado y los efectos de todos sus resultados se guardan en la misma transacción. Si el `eventId` ya existe, el evento se descarta sin efectos.
- Vamos a confirmar el *offset* recién después de que esa transacción haga commit.
- Vamos a propagar el `metadata.eventId` del evento externo como **`causationId`** a todo lo que genere. Esto incluye los eventos que publiquemos como consecuencia, para que la cadena entre módulos sea trazable.
- Vamos a **separar los errores transitorios de los permanentes**. Un error transitorio (la base caída) se reintenta. Un evento sin regla, inválido o que viola una restricción se registra y se descarta, sin reintentos infinitos: un evento con formato inesperado no puede frenar el consumo del tópico.
- Vamos a leer **directo de Kafka**, con SASL/OAUTHBEARER y el token del cliente `grupo5` del Login Federado. Los mensajes llegan en Avro con el formato de Confluent, y el `group.id` tiene que empezar con el namespace (`com.citypass.emergencias.`).
- Vamos a correr el listener **dentro del backend** (decisión de Producto del 2026-10-26). El hosting queda como está: no hay un proceso siempre prendido. Mientras el servicio duerme no lee, y al despertar retoma desde el último *offset* confirmado. `KAFKA_CONSUMER_ENABLED` lo apaga.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **Listener único dentro del backend + reglas por evento** | Un solo lugar para conexión, deduplicación y errores. Sumar un evento es sumar un listener y sus reglas. Las reglas son puras y fáciles de testear. | Un punto de entrada común: un error en el listener afecta a todas las integraciones. Solo lee con el servicio despierto. |
| Webhook del gateway | No necesita SASL, Avro ni un proceso leyendo: el dispatcher hace un `POST` con `metadata` y `data` en JSON. Es lo que usan Reclamos, Espacios y Residuos. | Si el destino está dormido, reintenta 3 veces cada 2 segundos y lo manda a la DLQ. A las 5 fallas seguidas silencia la suscripción 10 minutos. Con Render free, que tarda cerca de un minuto en despertar, se pierde el primer evento. Queda como plan B: `handle(event)` no depende del transporte. |
| Consumer en un servicio aparte que llama a un comando del backend | Siempre prendido, aguanta que Render duerma al backend: no confirma el *offset* hasta que el backend responde. | Segundo repo, segundo deploy e infraestructura paga. Hay que exponer y proteger una API interna. Producto lo descartó el 2026-10-26. |
| Mappers registrados a mano que emiten comandos (PR #62 del backend, cerrado) | El registro deja a la vista qué eventos se manejan. | Cada evento nuevo toca el registro. Los comandos escribían SQL directo, sin outbox ni historial. |
| Un consumer por funcionalidad | Cada funcionalidad es independiente de las otras. | Duplica deduplicación y conexión. Un evento con dos efectos se procesa dos veces con transacciones separadas, y puede quedar a medias. |
| Handlers con `if/switch` sobre el tipo de evento | Lo más rápido de escribir. | Cada origen nuevo modifica código existente. El formato externo se mezcla con la lógica de negocio. |

## Consecuencias

Se hace más fácil:

- Integrar un evento nuevo: se agrega el listener y sus reglas, sin tocar la clase base ni los servicios.
- Garantizar que un evento duplicado no tenga efectos, para todas las funcionalidades a la vez.
- Testear la integración sin Kafka: `handle(event)` recibe un evento y devuelve resultados.
- Seguir una cadena entre módulos con `correlationId` + `causationId`.

Se hace más difícil:

- **Leer el Kafka del Grupo 1 tiene su costo:** autenticación SASL con renovación del token, deserializar Avro contra el Schema Registry y respetar el prefijo del `group.id`. Lo integra Kevin.
- **Con el servicio dormido no se consume.** Los eventos esperan en el tópico hasta que despierta. El keep-warm cubre solo la ventana de los jueves de 17:00 a 23:30, que incluye las dos demos. Fuera de esa ventana, un evento puede tardar en procesarse.
- Hay que implementar la tabla `eventos_procesados` y su limpieza periódica.
- Dependemos del formato de cada módulo de origen: un cambio de su lado rompe su listener.

Falta implementar en `main` (al 2026-11-02):

- Deserializar Avro: `onMessage` hoy hace `JSON.parse`, y descartaría cada evento real como JSON inválido.
- SASL/OAUTHBEARER: `kafka.service.js` conecta sin autenticación.
- El prefijo del `group.id`: hoy es `emergencias-backend.<Clase>`.
- La deduplicación y la separación de errores: hoy un `handle()` que falla se reentrega sin fin.
- La factory: `onMessage` solo devuelve los resultados.

Pendiente de definir:

- Probar Kafka contra el bus real, para lo que hace falta el secreto de `grupo5`. Si no llega a tiempo, usamos el webhook.
- Si vuelve la carga manual de eventos desde el tablero, que estaba en el PR #62 cerrado.
- Retención de la tabla `eventos_procesados`.

## Historial

- 2026-10-16: creación del ADR (estado: propuesta)
- 2026-11-02: se aplica el review del 2026-10-18 y la decisión de Producto del 2026-10-26. El listener queda dentro del backend, se suman el webhook como alternativa y el costo de SASL y Avro, y se alinea con el paquete `src/listeners/` de `main` (reglas en vez de mappers). Cambia el título.
