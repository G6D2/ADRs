# Consumir todos los eventos externos con un solo consumer

- Estado: propuesta
- Fecha: 2026-11-16
- Decisores: @Salterm27 (lo implementó en el PR #112 del backend) y Producto (@juanimoli), que sumó la concurrencia configurable en el PR #115.
- Reemplaza a: —
- Reemplazada por: —

## Contexto

[Consumir eventos externos con un listener y reglas por evento](consumir-eventos-externos-con-un-listener-y-reglas-por-evento.md)
(aceptada el 05/11/2026) fija un único punto de entrada para los eventos
externos y descarta "un consumer por funcionalidad" porque "duplica
deduplicación y conexión". Pero su decisión habla de "un `EventListener`
concreto por tipo de evento" y no distingue entre:

- **listener:** la clase que interpreta un tipo de evento (reglas,
  contexto, resultados);
- **consumer:** la conexión a Kafka con su `group.id`.

El código resolvió esa ambigüedad al revés: desde el 29/10 cada
`EventListener` creaba su propio `kafka.consumer()`, con un grupo por
clase (`emergencias-backend.<Listener>`, y desde el 05/11
`com.citypass.emergencias.<Listener>`). Cada listener nuevo sumaba un
consumer, y como el ADR presentaba sumar un listener como gratis, ningún
PR lo volvió a medir:

| Fecha | Cambio | Listeners | Consumers | Conexiones al broker |
|---|---|---|---|---|
| 05/11 | Cobertura preventiva (#72) | 2 | 2 | 4 |
| 12/11 | Reclamo por ciclovía (#93) | 4 | 4 | 8 |
| 13/11 | `EmergenciaSolicitada` de Movilidad (#100) | 5 | 5 | 10 |
| 15/11 | `EmergenciaBicicleta` (#108) | 6 | 6 | 12 |
| 16/11 | **Un solo consumer (#112)** | 6 | 1 | 2 |
| 16/11 | `ActividadCreada.v3` (#107) | 7 | 1 | 2 |
| 16/11 | `ReservaCreada` (#119) | 8 | 1 | 2 |

Al 17/11 hay 8 listeners y 8 tópicos: `ReservaCreada` es opt-in y su
tópico está cargado en producción (`/health` muestra los 8 corriendo).

Fuerzas en juego cuando se decidió (madrugada del 16/11):

1. **Límite del broker.** El broker del Grupo 1 (EDA) aceptaba como
   máximo **10 conexiones simultáneas por IP**. Las que pasaban el límite
   se cortaban antes de terminar el handshake TLS y kafkajs reintentaba
   cada 30 s: esos consumers no leían. Desde el sexto listener, en
   producción un consumer quedaba en ese ciclo mientras `/health` lo
   mostraba corriendo.
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

**Después de decidir, el límite cambió.** El 16/11 a las 12:25 (una hora
después de que el #112 llegara a `main`), el gateway mergeó su PR #46: el
9092 pasó a limitar las conexiones *nuevas* por frecuencia, a poner cuotas
de Kafka por identidad (el namespace del token) y a dejar un tope de 64
conexiones por IP como respaldo. El motivo fue el mismo problema de IPs
compartidas, que también afectaba a Reclamos.

La decisión sigue valiendo sin el límite de 10: el consumo usa 2 conexiones
sin importar cuántos listeners haya, y hay un solo lugar para mirar su
estado.

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
- **Concurrencia:** el consumer procesa hasta `min(cantidad de tópicos, 3)`
  particiones a la vez (`partitionsConsumedConcurrently`). Con 8 tópicos
  son 3. `KAFKA_CONSUMER_CONCURRENCY` (entero ≥ 1) lo cambia; un valor
  inválido se ignora con un aviso en el log. El tope existe porque cada
  mensaje abre una transacción y el pool de Postgres (`DB_POOL_MAX`, 10) se
  comparte con los requests HTTP y los jobs. Hasta el #115 era una
  partición por tópico.
- El supervisor con reintentos y `/health` se mantienen: `/health` sigue
  mostrando un elemento por listener, todos con el estado del consumer.
- **Regla para lo que venga:** sumar un listener no puede sumar
  conexiones al broker. Un cambio que abra otra conexión al 9092 (otro
  consumer, un admin, el producer de Kafka) tiene que decir cuántas suma y
  por qué.
- La regla la hace cumplir un **test de arquitectura**
  (`src/tests/architecture/kafka-connections.test.js` en el backend): falla
  si `new Kafka()`, `.consumer()`, `.producer()`, `.admin()` o
  `new EventConsumer()` aparecen fuera de su único lugar permitido. Sumar
  uno es actualizar este ADR y después el test.
- **Un listener nuevo no lleva tópico por defecto** hasta confirmar el
  nombre en el catálogo del gateway. Suscribirse a un tópico que no existe
  hace fallar el arranque del consumer, y con un solo consumer eso frena a
  todos los listeners. Mientras tanto el tópico va en su variable de
  entorno, y sin ella el listener queda apagado (`disabled` en `/health`).
  Así entró `ReservaCreada` (#119).

### Migración

- Al desplegar el #112, el grupo nuevo arrancó sin offsets y releyó desde
  el principio. Lo anterior a `EDA_CONSUMER_SINCE` se salteó sin procesar;
  lo posterior que ya se había procesado lo descartó la deduplicación de
  `eventos_procesados`, que no tiene purga.
- Los 6 grupos por listener quedaron sin miembros. No ocupan conexiones;
  EDA los puede borrar.
- Fue un corte directo, sin convivencia: con los 6 consumers ya estábamos
  por encima del límite.

## Alternativas consideradas

| Alternativa | Decisión | Motivo |
|---|---|---|
| Un consumer y un grupo para todos los tópicos (elegida) | **Se hace** | 2 conexiones en régimen (4 en un deploy), independiente de la cantidad de listeners. |
| Seguir con un consumer por listener | Descartada | 12 conexiones con 6 listeners y 2 más por cada listener nuevo; ya superaba el límite. |
| Pedirle a EDA que suba el límite | Complementaria, no alcanzaba | No tocaba código, pero cada listener nuevo nos volvía a acercar al techo, y dependía de otro equipo. EDA terminó cambiando el límite ese mismo día (ver Contexto). |
| Apagar listeners para entrar en 10 | Descartada | Se pierden casos de negocio y queda sin margen para deploys. |
| `EventConsumer` como singleton | Descartada | Garantiza una instancia por proceso, pero el límite era por IP: durante un deploy hay dos procesos igual. Tampoco frena llamadas directas a `kafka.consumer()`, que fue lo que llevó a 12. Complica los tests. El test de arquitectura cubre los dos casos. |
| Compartir un solo cluster de kafkajs entre consumers | Descartada | kafkajs no lo permite: cada `consumer()` arma su propio cluster y su pool. |
| Un consumer por módulo de origen (Espacios, Reclamos, Movilidad) | Descartada | 6 conexiones con los módulos de hoy y crece con cada módulo nuevo; mezcla el límite técnico con la organización del negocio. |

## Consecuencias

### Esperadas

Se hace más fácil:

- Mantener el consumo en 2 conexiones (4 durante un deploy), lejos de
  cualquier tope del broker.
- Sumar listeners sin pensar en conexiones.
- Ver el estado del consumo en un solo lugar.

Se hace más difícil o cuesta:

- **Caída conjunta:** si el consumer se cae, dejan de leer todos los
  tópicos a la vez (antes caía uno solo). El supervisor lo vuelve a
  levantar con backoff.
- **Un tópico roto frena a todos.** Si un tópico no se puede leer, el
  consumer no arranca. Los tópicos con default en el código
  (`ActividadCreada.v3`, `EmergenciaSolicitada`, `ReclamoCreado.v3`,
  `ReclamoActualizado`, `EmergenciaBicicleta`) no se pueden apagar con una
  variable: la variable vacía vuelve al default. Al 17/11 los 8 están
  corriendo.
- **Relectura al migrar:** el primer arranque releyó todos los tópicos
  desde `EDA_CONSUMER_SINCE` y consultó la deduplicación por cada mensaje.
  Fue trabajo extra una sola vez.
- Un rebalanceo del grupo (por ejemplo, durante un deploy) pausa todos los
  tópicos juntos.

Neutras:

- La lógica de cada listener (reglas, dedup, factory, decodificación Avro)
  no cambia.
- `/health` mantiene su forma.

### No esperadas (riesgos a vigilar)

- **Un tópico lento todavía puede frenar a los demás.** Con 8 tópicos y
  concurrencia 3, se procesan a lo sumo 3 particiones a la vez: si tres
  quedan trabadas (un mensaje lento o un error transitorio que se
  reintenta), el resto espera. Subir `KAFKA_CONSUMER_CONCURRENCY` lo
  alivia a costa de conexiones del pool de Postgres. No verificamos
  cuántas particiones tiene cada tópico en el broker de EDA.
- **Un mensaje que falla con error transitorio** se reintenta en su
  partición. Si el error hace caer el consumer entero, se caen todos.
- **Sin orden entre tópicos.** Los tópicos se procesan en paralelo, así que
  un evento de un tópico puede procesarse antes que otro anterior de otro
  tópico. Es lo que deja abierto el issue #88 del backend (una cancelación
  procesada antes que la publicación).
- **Purga futura de `eventos_procesados`:** si se agrega una limpieza, una
  relectura desde el principio (grupo nuevo o offsets vencidos a los 7 días)
  podría reprocesar eventos ya borrados de esa tabla.
- **IP compartida de Render:** con el cambio del gateway el riesgo bajó
  (64 conexiones por IP y cuotas por identidad), pero no desapareció: otro
  servicio con la misma IP sigue compartiendo ese cupo.
- **Volver al producer de Kafka** (`EDA_PUBLISHER_MODE=kafka`) suma 2
  conexiones: 4 en régimen, 8 en un deploy.

## Relacionados

- [Consumir eventos externos con un listener y reglas por evento](consumir-eventos-externos-con-un-listener-y-reglas-por-evento.md): esta decisión aclara cómo se implementa su "punto de entrada único"; no lo reemplaza.
- [Usar Kafka como bus de eventos](usar-kafka-como-bus-de-eventos.md)
- [Elegir hosting cloud](elegir-hosting-cloud.md)

## Historial

- 2026-11-16: creación del ADR (estado: propuesta)
- 2026-11-16: se suma el test de arquitectura que hace cumplir la regla y se descarta el singleton.
- 2026-11-17: se actualiza con lo que pasó después: la concurrencia pasa a `min(cantidad de tópicos, 3)` y se configura con `KAFKA_CONSUMER_CONCURRENCY` (backend #115); el gateway levantó el límite de 10 conexiones por IP (su PR #46) y el contexto lo aclara; la tabla suma `ActividadCreada.v3` y `ReservaCreada` (8 listeners y 8 tópicos); y se agrega la regla de no poner tópico por defecto a un listener nuevo hasta confirmarlo en el catálogo del gateway.
