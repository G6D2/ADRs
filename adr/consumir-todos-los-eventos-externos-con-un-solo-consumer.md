# Consumir todos los eventos externos con un solo consumer

- Estado: propuesta
- Fecha: 2026-11-16
- Decisores: (a completar: Back, DevOps y Producto)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

[Consumir eventos externos con un listener y reglas por evento](consumir-eventos-externos-con-un-listener-y-reglas-por-evento.md)
(aceptada el 05/11/2026) fija un único punto de entrada para los eventos
externos y descarta "un consumer por funcionalidad" porque "duplica
deduplicación y conexión". Pero su decisión habla de "un `EventListener`
concreto por tipo de evento" y no distingue entre:

- **listener:** la clase que interpreta un tipo de evento (reglas,
  contexto, outcomes);
- **consumer:** la conexión a Kafka con su `group.id`.

El código resolvió esa ambigüedad al revés: desde el 29/10 cada
`EventListener` crea su propio `kafka.consumer({ groupId })`, con un grupo
por clase (`com.citypass.emergencias.<Listener>`). Cada listener nuevo
sumaba un consumer, y el ADR presentaba sumar un listener como gratis, así
que ningún PR lo volvió a medir:

| Fecha | Cambio | Consumers | Conexiones al broker |
|---|---|---|---|
| 05/11 | Cobertura preventiva | 2 | 4 |
| 12/11 | Reclamo por ciclovía | 4 | 8 |
| 13/11 | `EmergenciaSolicitada` de Movilidad | 5 | 10 |
| 15/11 | `EmergenciaBicicleta` | 6 | 12 |

Fuerzas en juego:

1. **Límite del broker.** Desde el 16/11/2026 el broker del Grupo 1
   (EDA) acepta como máximo **10 conexiones por IP**. Las que pasan el
   límite se cortan antes de terminar el handshake TLS y kafkajs reintenta
   cada 30 s: esos consumers no leen.
2. **Conexiones por consumer.** kafkajs (2.2.4) arma un cluster propio por
   cada `kafka.consumer()`, con 2 conexiones por broker: una para `Fetch` y
   otra para el resto (metadata, join, heartbeat, commit). Con un solo
   broker anunciado en el mismo `host:9092` del bootstrap, cada consumer son
   2 conexiones.
3. **Deploys en Render.** Render levanta la instancia nueva antes de bajar
   la vieja: durante un deploy las conexiones se duplican.
4. **IP compartida.** En el plan free de Render la salida a internet es por
   IPs compartidas entre servicios de la región. Si otro servicio que usa el
   mismo broker sale por la misma IP, comparte el cupo.
5. **Aislamiento.** Con un consumer por listener, un tópico con problemas
   no frenaba a los demás.

## Decisión

Vamos a consumir **todos los eventos externos con un solo consumer de
Kafka y un solo `group.id`**. Un listener es una clase que interpreta un
tipo de evento; no abre conexiones.

- `EventConsumer` (`src/listeners/event-consumer.js`) crea el único
  `kafka.consumer()`, se suscribe a los tópicos de todos los listeners
  (cada uno con su `fromBeginning`) y en `eachMessage` le pasa el mensaje al
  listener de su tópico (`listener.onMessage`).
- `group.id` = `com.citypass.emergencias.eventos`
  (`KAFKA_CONSUMER_GROUP_PREFIX` cambia el prefijo, que tiene que seguir
  empezando con el namespace).
- Dos listeners en el mismo tópico son un error de configuración y fallan
  al arrancar.
- Para no perder el aislamiento entre tópicos, el consumer procesa una
  partición por tópico en paralelo (`partitionsConsumedConcurrently` =
  cantidad de tópicos).
- El supervisor con reintentos y `/health` se mantienen: `/health` sigue
  mostrando un elemento por listener, todos con el estado del consumer.
- **Regla para lo que venga:** sumar un listener no puede sumar
  conexiones al broker. Un cambio que abra otra conexión al 9092 (otro
  consumer, un admin, el producer de Kafka) tiene que decir cuántas suma y
  cómo queda frente al límite.

### Migración

- El grupo nuevo arranca sin offsets y relee desde el principio. Lo
  anterior a `EDA_CONSUMER_SINCE` se saltea sin procesar; lo posterior que
  ya se había procesado lo descarta la deduplicación de
  `eventos_procesados`, que no tiene purga.
- Los 6 grupos por listener quedan sin miembros. No ocupan conexiones; EDA
  los puede borrar.
- Corte directo, sin convivencia: con los 6 consumers ya estamos por
  encima del límite.

## Alternativas consideradas

| Alternativa | Decisión | Motivo |
|---|---|---|
| Un consumer y un grupo para todos los tópicos (elegida) | **Se hace** | 2 conexiones en régimen (4 en un deploy), independiente de la cantidad de listeners. |
| Seguir con un consumer por listener | Descartada | 12 conexiones hoy y 2 más por cada listener nuevo; ya supera el límite. |
| Pedirle a EDA que suba el límite | Complementaria, no alcanza | No toca código, pero cada listener nuevo nos vuelve a acercar al techo, y depende de otro equipo. |
| Apagar listeners para entrar en 10 | Descartada | Se pierden casos de negocio y queda sin margen para deploys. |
| Compartir un solo cluster de kafkajs entre consumers | Descartada | kafkajs no lo permite: cada `consumer()` arma su propio cluster y su pool. |
| Un consumer por módulo de origen (Espacios, Reclamos, Movilidad) | Descartada | 6 conexiones hoy y crece con cada módulo nuevo; mezcla el límite técnico con la organización del negocio. |

## Consecuencias

### Esperadas

Se hace más fácil:

- Respetar el límite de EDA: 2 conexiones, 4 durante un deploy.
- Sumar listeners sin pensar en conexiones.
- Ver el estado del consumo en un solo lugar.

Se hace más difícil o cuesta:

- **Caída conjunta:** si el consumer se cae, dejan de leer todos los
  tópicos a la vez (antes caía uno solo). El supervisor lo vuelve a
  levantar con backoff.
- **Relectura al migrar:** el primer arranque relee todos los tópicos
  desde `EDA_CONSUMER_SINCE` y consulta la dedup por cada mensaje. Es
  trabajo extra una sola vez.
- Un rebalanceo del grupo (por ejemplo, durante un deploy) pausa todos los
  tópicos juntos.

Neutras:

- La lógica de cada listener (reglas, dedup, factory, decodificación Avro)
  no cambia.
- `/health` mantiene su forma.

### No esperadas (riesgos a vigilar)

- **Un tópico lento todavía puede afectar a los demás** si tiene varias
  particiones o si la concurrencia no alcanza: el paralelismo es por
  partición, no por tópico. Hoy cada tópico tiene una partición.
- **Un mensaje que falla con error transitorio** se reintenta en su
  partición; con la concurrencia, los demás tópicos siguen. Si el error
  hace caer el consumer entero, se caen todos.
- **Purga futura de `eventos_procesados`:** si se agrega una limpieza, una
  relectura desde el principio (grupo nuevo o offsets vencidos a los 7 días)
  podría reprocesar eventos ya borrados de esa tabla.
- **IP compartida de Render:** aunque bajemos a 2, otro servicio con la
  misma IP puede agotar el cupo. Hay que confirmar con EDA si el límite se
  puede asociar al cliente o al token en vez de a la IP.
- **Volver al producer de Kafka** (`EDA_PUBLISHER_MODE=kafka`) suma 2
  conexiones: 4 en régimen, 8 en un deploy.

## Relacionados

- [Consumir eventos externos con un listener y reglas por evento](consumir-eventos-externos-con-un-listener-y-reglas-por-evento.md): esta decisión aclara cómo se implementa su "punto de entrada único"; no lo reemplaza.
- [Usar Kafka como bus de eventos](usar-kafka-como-bus-de-eventos.md)
- [Elegir hosting cloud](elegir-hosting-cloud.md)

## Historial

- 2026-11-16: creación del ADR (estado: propuesta)
