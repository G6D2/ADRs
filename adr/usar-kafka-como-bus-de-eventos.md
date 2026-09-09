# Usar Kafka como bus de eventos

- Estado: aceptada
- Fecha: 2026-09-07
- Decisores: @juanimoli (Producto), @Matiamistadi y @luccaperazzo (Back), @KevinAlajarin (DevOps), Grupo 1 (EDA)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

CityPass+ se construye con arquitectura orientada a eventos: cada módulo
publica lo que ocurre en su dominio y reacciona a lo que publican los demás.
Emergencias publica eventos de su ciclo de vida (`EmergenciaCreada`,
`EmergenciaPriorizada`, `EmergenciaEstadoActualizado`, `EmergenciaDespachada`,
`EmergenciaRefuerzoSolicitado`, `EmergenciaCerrada`) y consume eventos de
otros módulos que pueden originar una emergencia (por ejemplo un incendio
detectado por Residuos o un accidente de Movilidad). El módulo de Analítica
ya diseñó sus tableros sobre nuestros eventos.

El Grupo 1 (EDA) es dueño de la infraestructura de mensajería y provee un
**event gateway sobre Apache Kafka** con estas reglas:

- Autenticación OAuth 2.0 *client credentials* contra el proveedor de
  identidad del Grupo 2; el token lleva un claim `namespace` que a la vez es
  el prefijo del tópico (`com.citypass.<namespace>.<EventType>`).
- Registro previo de cada tipo de evento (`POST /api/v1/event-types`), nombre
  en PascalCase y tiempo pasado, sin sufijo `Event` ni versión en el nombre.
- El productor envía solo `data`; el gateway sella la `metadata` (`eventId`,
  `occurredAt`, `source`, `version`).
- Entrega **at-least-once**: pueden llegar duplicados. El bus no correlaciona
  eventos entre sí.
- Publicar o consumir con un namespace ajeno devuelve 403 o es denegado por
  el broker.

El Grupo 1 eligió Kafka por su cuenta el 15/08 y pidió a cada grupo que deje
la adopción registrada como ADR propio. Nuestro backend ya integra `kafkajs`
para desarrollo local (broker en `docker-compose`) y hoy publica la entidad
cruda sin envelope, con nombres de tópico en minúsculas
(`emergencias.boton-panico`, `emergencias.estado-actualizado`), lo que no
cumple el contrato del gateway.

Fuerzas en juego: el bus no lo elegimos nosotros, así que la decisión real es
**cómo** nos integramos a él; el equipo es junior y no puede operar un broker;
en el Hito 1 los eventos pueden publicarse sin que nadie los consuma; la
integración real se valida entre los Sprints 3 y 6; y hay un conflicto de
numeración de namespaces con otro grupo que sigue sin cerrarse.

## Decisión

- Vamos a usar **Kafka, a través del event gateway del Grupo 1**, como único
  canal de integración asíncrona con los demás módulos. No vamos a operar un
  broker propio fuera del entorno local de desarrollo.
- Vamos a encapsular toda la comunicación con el bus en una abstracción
  `EventBus` del backend (publicar, consumir, obtener y refrescar el token por
  *client credentials*, reintentos). Los controllers no hablan con Kafka.
- Vamos a publicar los seis eventos del dominio con nombre
  `com.citypass.<namespace>.<EventType>` en PascalCase y tiempo pasado,
  enviando solo `data` con los cuatro ejes de la emergencia (origen, tipo,
  severidad, estado), la ubicación y un `correlationId` propio para trazar la
  cadena de una misma emergencia.
- Vamos a **deduplicar por `metadata.eventId`**, nunca por hash del payload,
  y a mantener una tabla de eventos procesados.
- Vamos a publicar de forma confiable con un **outbox**: el cambio de estado y
  el evento pendiente se guardan en la misma transacción, y un job los envía y
  reintenta.
- Vamos a obtener el `namespace` y las credenciales del bus desde la
  configuración, nunca del código, y a no escribir ningún cliente contra el
  gateway hasta que el Grupo 1 confirme por escrito el namespace de
  Emergencias.
- Mientras el gateway no esté disponible, vamos a mantener el broker local de
  `docker-compose` y la bandera `KAFKA_ENABLED=false` en Render, para que el
  backend arranque sin bus.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **Kafka vía gateway del Grupo 1** | Es el estándar impuesto por la plataforma: sin él no hay integración con ningún módulo. Persistencia y replay de eventos. Operación a cargo del Grupo 1. | Dependencia fuerte de otro equipo (namespace, registro de tipos, disponibilidad). At-least-once obliga a idempotencia. Curva de aprendizaje de Kafka para el equipo. |
| RabbitMQ propio | Más simple de operar y de entender para un equipo junior. Colas y ruteo flexibles. | Nadie más lo usaría: quedaríamos aislados del resto de CityPass+. Habría que hostearlo y mantenerlo. |
| Webhooks HTTP entre módulos | Trivial de implementar con Express. Sin infraestructura nueva. | Acoplamiento punto a punto con cada grupo, sin persistencia ni replay, sin desacoplamiento temporal. Contradice la consigna de EDA. |
| Polling REST de los otros módulos | Cero infraestructura y control total del ritmo. | Cada módulo debería exponer endpoints para nosotros; latencia y carga innecesarias; no escala a 8 grupos. |

## Consecuencias

Se hace más fácil:

- Integrarnos con cualquier módulo de CityPass+ sin acuerdos bilaterales de
  transporte: solo hay que acordar el contrato del evento.
- Que Analítica y otros consumidores usen nuestros eventos sin tocar nuestro
  backend.
- Justificar la dimensión de arquitectura orientada a eventos de la rúbrica
  con un componente real.

Se hace más difícil:

- El backend depende del Grupo 1 para publicar en producción: sin namespace
  confirmado, sin registro de tipos y sin gateway disponible, los eventos
  quedan en el outbox.
- Hay que implementar idempotencia, outbox y refresco de token antes de la
  integración real (Sprints 3 a 6), y testear sin bus real.
- Hay que renombrar los tópicos y armar el envelope que hoy no cumplen el
  contrato, y volver a coordinar el payload con Analítica.
- La operación del broker queda fuera de nuestro control: un corte del Grupo 1
  se vuelve un corte nuestro.

## Historial

- 2026-09-07: creación del ADR (estado: propuesta)
- 2026-09-07: cambio de estado: propuesta → aceptada

