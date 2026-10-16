# Generar respuestas predeterminadas con reglas y templates por recurso

- Estado: propuesta
- Fecha: 2026-10-16
- Decisores: @juanimoli (Producto), @Matiamistadi y @luccaperazzo (Back)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

[Publicar las respuestas requeridas como un evento por tipo de recurso](publicar-las-respuestas-requeridas-como-un-evento-por-tipo-de-recurso.md) define qué publicamos: un evento por recurso (Bomberos, Policía o Ambulancia). Falta decidir cómo lo generamos.

Una misma respuesta se puede originar en disparadores distintos:

- **Internos:** un usuario reporta un incendio en nuestra plataforma y se crea la emergencia por HTTP.
- **Externos:** otro módulo informa un incendio por el bus, por ejemplo Tránsito y Movilidad. Entra por el listener de [Consumir eventos externos con un listener y mappers por origen](consumir-eventos-externos-con-un-listener-y-mappers-por-origen.md).

Van a aparecer nuevos disparadores que pidan un recurso existente (otro caso que requiere Bomberos), y eventualmente nuevos recursos. Queremos sumar cualquiera de los dos sin modificar el código que ya funciona.

Además, si se crea una emergencia y la respuesta no sale, el problema es grave. La respuesta tiene que ser atómica con la creación.

## Decisión

- Vamos a separar **disparadores** de **respuestas** con un **catálogo de reglas** en código que mapea cada tipo de emergencia a los recursos que requiere. Ejemplos ilustrativos, a definir con Producto: `INCENDIO → [BOMBEROS]`, `MEDICA → [AMBULANCIA]`. Un disparador nuevo que pide un recurso existente se resuelve con una regla, o con un mapper que lo traduzca a un tipo existente.
- Vamos a modelar cada recurso como un **Template Method**. La clase base `ResponseTemplate` fija el esqueleto común: identificadores, `correlationId`, `causationId`, origen, barrio y fecha. Cada subclase (`BomberosResponse`, `PoliciaResponse`, `AmbulanciaResponse`) completa solo su tipo de evento y su payload específico. Un recurso nuevo es un template nuevo.
- Vamos a usar un **factory** (`ResponseEventFactory`) que, dado un recurso y el contexto de la emergencia, elige el template y devuelve la fila de outbox lista. El `OutboxPublisher` existente la publica sin cambios.
- Vamos a que el **flujo interno no pase por el bus**. Al crear la emergencia por HTTP, el servicio aplica las reglas y guarda las respuestas en el outbox **en la misma transacción** que la emergencia.
- Vamos a que el **flujo externo** llegue como un comando emitido por un mapper del listener. El comando invoca el mismo servicio, así que no hay una segunda implementación.
- Vamos a que reglas, templates y factory sean **módulos puros**: no conocen Postgres ni Kafka, reciben datos y devuelven filas de outbox.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **Reglas + Template por recurso + factory al outbox** | Disparadores y recursos crecen de forma independiente. Una sola construcción de cada evento. Reutiliza el outbox. | Más piezas que una solución directa. Hay que cuidar que la lógica de disparador no se filtre en los templates. |
| Un handler por caso de uso que arma sus eventos | Cada caso queda autocontenido. | Un segundo caso que pide Bomberos duplica la construcción del evento. Cambiar un payload obliga a tocar todos los casos. |
| Todo entra por el listener, incluido nuestro `EmergenciaCreada` | Un único punto de entrada. | La respuesta interna depende de que el bus esté disponible. Pierde atomicidad con la creación y no funciona con `KAFKA_ENABLED=false`. |
| Reglas en base de datos editables desde el tablero | Producto cambia reglas sin deploy. | Una pantalla y validaciones más. Reglas sin versionado ni tests. No se justifica con tres recursos. |

## Consecuencias

Se hace más fácil:

- Sumar un disparador o un recurso sin tocar lo que ya funciona.
- Cambiar el payload de un recurso en un único lugar cuando se renegocie el contrato.
- Testear sin infraestructura: reglas, templates y factory son puros.
- Garantizar que toda emergencia creada en la plataforma tenga sus respuestas encoladas.

Se hace más difícil:

- Cambiar una regla requiere deploy.
- Hay tres conceptos nuevos (regla, template, factory) que el equipo tiene que entender antes de sumar un caso.

Pendiente de definir:

- El catálogo inicial de reglas, con Producto.
- Si un disparador externo crea una emergencia propia o solo genera la respuesta. Si la crea, reutiliza el flujo interno completo.

## Historial

- 2026-10-16: creación del ADR (estado: propuesta)
