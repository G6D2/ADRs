# Responder a las alertas climáticas por barrio

- Estado: aceptada
- Fecha: 2026-11-18
- Decisores: Producto (@juanimoli), que lo decidió el 2026-11-18. La propuesta es de @SJTG (Santino), que la redactó en el PR #48 de este repositorio y en la rama `feat/recuperada-confirmacion-clima`, y la implementó en el backend (PR #146 y #147). Parte del ADR anterior, de @Salterm27 (reglas de negocio e implementación) y @KevinAlajarin (redacción).
- Reemplaza a: [Responder a las alertas climáticas de Espacios](responder-a-las-alertas-climaticas-de-espacios.md)
- Reemplazada por: —

## Contexto

Espacios Públicos detecta riesgo climático con Open-Meteo. Un administrador lo confirma y publica `com.citypass.espacios.CriticidadClimaticaConfirmada` (schemaId 28). Cuando la alerta vence o se cierra a mano, publica `CriticidadClimaticaFinalizada` (schemaId 27) con el mismo `alertaId`.

| Campo de la confirmación | Contenido |
| --- | --- |
| `alertaId` | Clave de negocio de la alerta. Es la misma en la finalización |
| `nivelCriticidad` | `BAJA`, `MEDIA`, `ALTA` o `CRITICA` |
| `fenomeno` | `TORMENTA_FUERTE`, `LLUVIA_INTENSA`, `GRANIZO`, etc. |
| `ubicaciones` | `[{ latitud, longitud }]`: los puntos que Espacios consulta a Open-Meteo (`WEATHER_LOCATIONS`). Espacios aclara que no son un polígono ni un radio de afectación |
| `barrios` | Barrios que contienen esos puntos |
| `vigenciaDesde`, `vigenciaHasta` | Ventana de la alerta, en epoch ms |

Espacios aclara que el evento informa un hecho y no ordena acciones: cada módulo decide su política por nivel.

[Responder a las alertas climáticas de Espacios](responder-a-las-alertas-climaticas-de-espacios.md), aceptado el 17/11 y en producción desde ese día (backend #122), respondió con dos efectos para `ALTA` y `CRITICA`: un contexto de zona `ALERTA_CLIMATICA` por cada punto de `ubicaciones`, con un radio de 1000 m (`CRITICA`) o 500 m (`ALTA`), y una emergencia `PREVENTIVA` por alerta. Quedaron tres problemas.

**1. El radio fijo deja afuera buena parte del barrio alertado.** Espacios no informa la superficie de la tormenta: en la práctica, una alerta dice «hay riesgo en estos barrios». Pero un círculo de 500 m o 1000 m alrededor del punto consultado no cubre ni siquiera el barrio más chico:

- El radio equivalente de un barrio (el de un círculo con la misma superficie) va de unos 630 m en San Telmo (1,23 km²) a unos 2,2 km en Palermo (15,9 km²), según el catálogo GCBA de 48 barrios que usa el backend. **Ninguno baja de 600 m.**
- Un círculo de 500 m (`ALTA`) cubre, como mucho, el 64 % de San Telmo y el 5 % de Palermo. Con 1000 m (`CRITICA`), como mucho el 20 % de Palermo. «Como mucho», porque el punto no tiene por qué estar en el centro del barrio.
- Por ejemplo, con una alerta en Palermo, una emergencia en Palermo Hollywood, a unos 3 km del punto consultado, no sube aunque esté en el barrio alertado.

**2. Las emergencias que ya estaban abiertas no suben.** Es la regla 1 de [Elevar la prioridad en un radio alrededor del evento](elevar-la-prioridad-en-un-radio-alrededor-del-evento.md): el vínculo se mide una sola vez, al crear la emergencia, porque la ubicación está cifrada y medir después obliga a descifrar. Para los eventos públicos, Producto la confirmó el 17/11. Para una alerta climática no alcanza: cuando llega la alerta, la tormenta ya empezó, y las emergencias abiertas en la zona son justo las que están bajo ella. El barrio de la emergencia, en cambio, se guarda en claro al crearla ([Guardar el barrio de la emergencia al crearla](guardar-el-barrio-de-la-emergencia-al-crearla.md)), así que con una zona por barrio no hace falta descifrar nada.

**3. Emergencias no deja constancia en el bus de que recibió la alerta.** Residuos publica `com.citypass.residuos.CriticidadClimaticaConfirmada` con `alertaId`, `NivelCriticidad` y `fenomeno`, y Analítica ingiere ese tipo. Emergencias no publica nada equivalente: el único rastro es la `EmergenciaCreada` de la `PREVENTIVA`, que existe solo con `ALTA` o `CRITICA`.

Santino propuso los tres cambios y los implementó en el backend (#146 y #147). Los redactó como cambios al ADR anterior (PR #48 y rama `feat/recuperada-confirmacion-clima`), pero ese ADR ya está aceptado y, según la convención del registro, un ADR aceptado no se reescribe. Este lo reemplaza con la decisión de Producto del 18/11 y repite lo que no cambia (niveles, emergencia preventiva, fin de la alerta y aviso), para que se lea solo.

## Decisión

Vamos a responder a la alerta climática con tres efectos:

1. Con `ALTA` o `CRITICA`, **subir la prioridad en los barrios de la alerta**, también la de las emergencias que ya estaban abiertas.
2. Con `ALTA` o `CRITICA`, **programar una emergencia preventiva**, como hasta ahora.
3. Con cualquier nivel, **confirmar por el bus que recibimos la alerta**, con el mismo record que Residuos. Queda **apagado** hasta que Analítica confirme que lo va a ingerir.

### 1. Zona: un contexto por barrio, con el círculo que cubre el barrio

Cada barrio de `barrios` que está en el catálogo de CABA genera un contexto de zona `ALERTA_CLIMATICA`, con un punto y un radio que cubren el barrio entero (`zonaDelBarrio` en `src/domain/barrio.js`):

- **Centro:** el promedio de los vértices del polígono del barrio (catálogo GCBA). Es aproximado: un borde con muchos vértices corre el centro hacia ese borde y agranda el radio.
- **Radio:** la distancia del centro al vértice más lejano, más 100 m. El margen es la misma tolerancia con la que se le asigna barrio a un punto que cae en un hueco del polígono (`NEAREST_BARRIO_MAX_METERS`). Va de 934 m (Coghlan) a 4084 m (Recoleta). Palermo tiene 3837 m.
- El círculo de cada barrio se calcula una vez desde el catálogo y queda en memoria.

**Filtro por barrio.** El círculo se pasa a los barrios vecinos: el de Palermo alcanza pedazos de Belgrano, Colegiales, Recoleta y Villa Crespo. Por eso, **para vincularse, la emergencia tiene que estar dentro del círculo y además ser de ese barrio**. Es el barrio que se le asigna al crearla, en claro. Una emergencia sin barrio (fuera de CABA) no entra en estos contextos.

**Respaldo por punto.** Si la alerta no trae ningún barrio del catálogo, cada punto de `ubicaciones` genera un contexto con su `latitud`, `longitud` y el radio del nivel (punto 3), sin filtro por barrio, como en el ADR anterior. Así una alerta sin barrios no se pierde.

**Referencia.** La referencia externa de cada contexto es `<alertaId>#<índice>`, con la posición del barrio en `barrios` o la del punto en `ubicaciones`. Los barrios que no están en el catálogo y los puntos sin coordenadas numéricas se descartan sin correr el índice de los demás, y cada barrio cuenta una sola vez: la referencia es la misma en cada reentrega. `alertaId` agrupa los contextos para darlos de baja juntos.

**Datos.** Cada contexto por barrio guarda `barrios = [su barrio]` en la columna `barrios` de V33, que hasta ahora servía solo para mostrar la alerta. Un contexto por punto guarda `barrios = null`. `linkZoneContexts` exige el barrio solo cuando el contexto lo tiene, así que los eventos públicos, que no tienen barrios, no cambian. No hace falta migración.

Lo demás funciona igual que un evento público:

- la ventana del contexto es `vigenciaDesde`–`vigenciaHasta`;
- la suba se calcula en SQL al leer, sin descifrar nada;
- es de un nivel, con tope en la prioridad máxima;
- varios contextos sobre la misma emergencia no se acumulan;
- las `PREVENTIVA` no suben.

### 2. Emergencias que ya estaban abiertas

Al dar de alta cada contexto climático, en la misma transacción, **también se vinculan las emergencias que ya estaban abiertas en su zona** (`linkOpenEmergenciesToZoneContext` en `EmergencyRepository`). Alcanza a las emergencias que:

- no están en un estado terminal (`CLOSED`, `CANCELLED` o `DISCARDED`);
- no son `PREVENTIVA`;
- tienen ubicación y no está anonimizada: sin coordenadas no hay vínculo, y una anonimizada ya perdió sus vínculos.

Cómo se decide quién está en la zona:

- **Zona por barrio:** alcanza con que el barrio guardado en la emergencia sea el del contexto. Se resuelve en SQL, sin descifrar nada: todo punto del barrio cae dentro de su círculo, y el filtro ya exige el barrio.
- **Zona por punto:** se descifran las emergencias abiertas y se mide la distancia en memoria. Es el costo que el ADR de radio quería evitar, pero acá pasa una vez por contexto, al darlo de alta, y solo en las alertas sin barrios del catálogo.

El vínculo se escribe igual que al crear la emergencia: los dos momentos usan `writeZoneContextLinks`. Un vínculo que ya existe queda como estaba (`ON CONFLICT DO NOTHING`), así que una reentrega no lo duplica, y con el evento de vínculo encendido sale `EmergenciaVinculadaAContextoZona` en la misma sentencia. Si la alerta ya se finalizó, no se da de alta nada y tampoco se vincula.

**Solo para alertas climáticas (Producto, 18/11).** Los eventos públicos siguen con la regla 1 del ADR [Elevar la prioridad en un radio alrededor del evento](elevar-la-prioridad-en-un-radio-alrededor-del-evento.md): solo sube lo que se crea con el contexto vigente. Este ADR no cambia esa regla: abre una excepción para las alertas.

**Versión de la regla.** Como cambia la forma de decidir el vínculo, `reglaVersion` de `EmergenciaVinculadaAContextoZona` pasa de 1 a 2 (ver [Publicar el vínculo de una emergencia con un contexto de zona](publicar-el-vinculo-de-una-emergencia-con-un-contexto-de-zona.md)). Cuando la emergencia ya estaba abierta, `vinculadaEn` es el momento en que se dio de alta el contexto.

### 3. Niveles

| Nivel | Contextos de zona | Radio de las alertas sin barrios | Emergencia preventiva | Confirmación al bus |
| --- | --- | --- | --- | --- |
| `CRITICA` | Sí | 1000 m | Sí | Sí, cuando esté encendida |
| `ALTA` | Sí | 500 m | Sí | Sí, cuando esté encendida |
| `MEDIA` | No | — | No | Sí, cuando esté encendida |
| `BAJA` | No | — | No | Sí, cuando esté encendida |

Con barrios, el nivel solo decide si hay contextos: el radio es el que cubre cada barrio. Sin barrios, el radio depende del nivel, porque una alerta crítica afecta un área mayor que una alta. Los dos radios por nivel son configurables por entorno (`CONTEXTO_ZONA_RADIO_CLIMA_CRITICA_METROS` y `CONTEXTO_ZONA_RADIO_CLIMA_ALTA_METROS`). Cada contexto guarda el radio con el que se registró.

`MEDIA` y `BAJA` no suben prioridades ni crean emergencias, en línea con Espacios, donde esos niveles no son accionables. Solo se confirman (punto 6).

### 4. Emergencia preventiva

Sin cambios respecto del ADR anterior. Con `ALTA` o `CRITICA` se crea **una** emergencia `PREVENTIVA` por `alertaId`, con el mismo esquema que la cobertura preventiva de eventos:

- `causaTipo = EVENTO_CLIMATICO` y `causaId = alertaId`;
- la deduplica el índice único de causa de las `PREVENTIVA`;
- se crea sin ciudadano, en `PENDING` y con `source = EXTERNAL`;
- `EmergenciaCreada` sale por el outbox, en la misma transacción.

La ubicación es el primer punto válido de `ubicaciones`. Si no hay ninguno, queda para validación manual. La severidad sale del nivel: `ALTA` → `high` y `CRITICA` → `critical`. La descripción lleva el nivel, el fenómeno y los barrios.

Ante una alerta alta o crítica, el dispatcher tiene en el tablero una tarea concreta de despliegue preventivo, y Analítica puede contar las emergencias por evento climático (`causaTipo = EVENTO_CLIMATICO`).

Una alerta que ya venció cuando llega no crea la emergencia. Pasa, por ejemplo, al releer el tópico.

### 5. Fin de la alerta

Sin cambios respecto del ADR anterior. `CriticidadClimaticaFinalizada`:

- **Da de baja todos los contextos de la alerta,** por barrio o por punto. Las emergencias vinculadas bajan solas, porque la suba se calcula al leer.
- **Cancela la emergencia preventiva si nadie la despachó** (`PENDING` o `VALIDATED`). Publica `EmergenciaActualizada`. Si ya avanzó, queda como está.
- **Gana aunque llegue antes que la confirmación.** Deja una marca de baja, y la confirmación tardía no crea contextos ni vincula emergencias.

### 6. Confirmación por el bus

Al recibir `CriticidadClimaticaConfirmada`, de cualquier nivel, Emergencias publica `com.citypass.emergencias.CriticidadClimaticaConfirmada`:

| Campo | Contenido |
| --- | --- |
| `alertaId` | El de la alerta de Espacios |
| `NivelCriticidad` | El `nivelCriticidad` recibido, tal como llegó. Va con mayúscula inicial, como en el schema de Residuos |
| `fenomeno` | El recibido, tal como llegó |

- **Es el mismo record que publica Residuos** (`com.citypass.residuos.CriticidadClimaticaConfirmada`), que Analítica ya ingiere: puede leer los dos con el mismo schema.
- Sale por el outbox, en la misma transacción que la emergencia y los contextos de la alerta, con `correlationId = alertaId`.
- Una reentrega del mismo evento no la duplica: la frena la deduplicación por `metadata.eventId`.
- Si falta alguno de los tres campos, no se confirma.
- Es una constancia de recepción: sale aunque la alerta llegue vencida o ya finalizada.

**Apagada hasta que Analítica confirme (Producto, 18/11).** Se encola solo con `CONFIRMACION_CLIMA_ENABLED=true`, y por defecto está apagada, como el evento de vínculo. Se prende cuando se cumplan las dos condiciones:

- Analítica confirma que va a ingerir el tipo de Emergencias;
- el tipo está registrado en el catálogo del gateway. Si no lo está, el gateway responde 404 y cada fila del outbox queda `FAILED`.

El tópico se puede cambiar con `TOPIC_CONFIRMACION_CLIMA`.

### 7. Aviso climático

No se implementa un evento `ContextoZonaActivado`:

- La alerta ya está en el bus: la publica Espacios para todos los módulos, y ningún módulo pidió un evento nuestro de activación. La confirmación del punto 6 no es una activación: solo dice que Emergencias recibió la alerta.
- El vínculo de cada emergencia con la alerta sale por `EmergenciaVinculadaAContextoZona`, con `contextoTipo = ALERTA_CLIMATICA`.
- Para el dispatcher, el aviso es la emergencia `PREVENTIVA` y el banner del tablero (Frontend #79), que lee `GET /api/contextos-zona/activos` (backend #121). Esa ruta suma `zona` (`BARRIOS` o `PUNTOS`) y, con zona por barrio, devuelve `radioMetros` en `null`, porque cada barrio tiene su radio. El banner ya lo muestra como «sube un nivel en esos barrios» (Frontend #93).
- El motivo de cada suba se ve en `priorityContext`, por ejemplo «Alerta climática CRITICA: tormenta fuerte».

### 8. Implementación

Sigue la estructura del ADR de listeners:

- Un paquete `src/listeners/alerta-climatica/` con un listener por evento.
- **Reglas de la confirmación:** `01-crear-emergencia-climatica`, `02-registrar-contextos-climaticos` y `03-publicar-confirmacion-clima`.
- **Reglas de la finalización:** `01-cancelar-emergencia-climatica` y `02-dar-de-baja-contextos-climaticos`.
- Handlers registrados en el `OutcomeFactory`, en la transacción del evento.
- Deduplicación por `metadata.eventId`, como el resto de los listeners.

Piezas de este ADR:

- **Zona:** `zonasDeLaAlerta` (`src/domain/alerta-climatica.js`) arma una zona por barrio del catálogo o, si no hay ninguno, una por punto. `zonaDelBarrio` (`src/domain/barrio.js`) calcula el círculo de cada barrio.
- **Vínculo:** `linkZoneContexts` (al crear la emergencia, con el filtro por barrio), `linkOpenEmergenciesToZoneContext` (al dar de alta el contexto; la llama el handler de contextos climáticos) y `writeZoneContextLinks` (la escritura compartida), en `src/repositories/emergency.repository.js`.
- **Confirmación:** la regla `03-publicar-confirmacion-clima` y `confirmacion-clima.handler.js`, que la encola en el outbox. `CriticidadClimaticaConfirmada` se suma a los tipos que el publisher manda al gateway (`PUBLISHED_TYPES`).

El código está en el backend #146 (zona por barrio y emergencias abiertas) y #147 (confirmación), sobre el #122. La regla para los módulos que recalculan la suba está en `docs/prioridad-por-contexto-para-consumidores.md` del backend.

## Alternativas consideradas

### Zona

| Alternativa | A favor | En contra |
| --- | --- | --- |
| Contexto por punto con radio por nivel (ADR anterior, backend #122) | Un solo valor por nivel. No necesita el catálogo de barrios | Ningún barrio tiene un radio equivalente menor a 600 m: el círculo de 500 m cubre como mucho el 64 % del barrio más chico y el 5 % de Palermo. Una emergencia del barrio alertado a 2 km del punto no sube |
| Círculo que cubre el barrio, sin filtrar por barrio | El mismo vínculo por distancia que un evento público | El círculo se pasa a los vecinos: el de Palermo (3837 m) sube emergencias de Belgrano, Colegiales, Recoleta y Villa Crespo, que no están alertados |
| Solo el barrio, sin círculo (la emergencia sube si su barrio está en la alerta) | Lo más simple: una comparación de texto, sin geometría ni centro aproximado | El contexto se queda sin punto ni radio, que es la única forma de zona desde el ADR de radio: `contexto_zona`, `linkZoneContexts`, `priorityContext` y `EmergenciaVinculadaAContextoZona` (con `radioMetros` obligatorio en el contrato) la suponen. Haría falta otro camino de vínculo solo para clima, como el de la zona por barrio que el ADR de radio abandonó |
| Toda la ciudad para `CRITICA` | Una tormenta no respeta barrios | Sube todas las emergencias a la vez, así que la prioridad deja de distinguir nada |
| **Círculo que cubre el barrio y filtro por barrio, con el punto como respaldo** (elegida) | Sube en todo el barrio alertado y solo ahí. Mantiene punto y radio como única forma de zona: el barrio es un filtro. No descifra nada al leer | El centro es aproximado y el círculo de un barrio grande es grande. El vínculo depende de que la emergencia tenga barrio |

### Emergencias que ya estaban abiertas

| Alternativa | A favor | En contra |
| --- | --- | --- |
| Vincular solo al crear la emergencia (regla del ADR de radio) | Un solo momento de vínculo. Nunca descifra ubicaciones | Las emergencias abiertas cuando llega la alerta, que son las que ya están bajo la tormenta, no suben |
| Vincular las abiertas también en los eventos públicos | Una sola regla para todos los contextos | Producto la descartó (18/11): los eventos públicos siguen con la regla que confirmó el 17/11 en el ADR de radio. Además, como los eventos no tienen barrio, obligaría a descifrar las emergencias abiertas en cada alta de evento |
| **Vincular las abiertas al dar de alta el contexto, solo en las alertas climáticas** (elegida) | Suben las emergencias que ya estaban bajo la tormenta. Con zona por barrio no descifra nada | Las alertas y los eventos siguen reglas distintas. Sin barrios, el alta descifra las emergencias abiertas |

### Confirmación

| Alternativa | A favor | En contra |
| --- | --- | --- |
| Sin confirmación propia (ADR anterior) | Un tipo menos en el catálogo | Emergencias no deja constancia en el bus de que recibió la alerta, como sí hace Residuos |
| Confirmación encendida desde ya | Analítica la tiene apenas la quiera | Si nadie la ingiere, es ruido en el bus; si el tipo no está en el catálogo, cada fila del outbox queda `FAILED` |
| **Confirmación con el record de Residuos, apagada hasta que Analítica confirme** (elegida) | Analítica la lee con el schema que ya usa. No publica nada que nadie consuma | Hay que coordinar con Analítica y con el catálogo del gateway para prenderla |

## Consecuencias

Se hace más fácil:

- Que la suba alcance a todo el barrio alertado, y no solo a los alrededores de los puntos que consulta Espacios.
- Que suban las emergencias que ya estaban bajo la tormenta cuando llegó la alerta.
- Mantener una sola forma de zona (punto y radio) para todos los contextos: el barrio se suma como filtro, sin tablas ni migraciones nuevas.
- Que una alerta alta o crítica se vea en el tablero y en Analítica.
- Cuando se prenda la confirmación, que Analítica sepa qué alertas recibió Emergencias, con el mismo schema que las de Residuos.

Se hace más difícil, y lo aceptamos:

- **La suba depende de los barrios que manda Espacios.** Son los barrios donde caen los puntos consultados, no la superficie de la tormenta. Una emergencia de otro barrio no sube, aunque esté bajo la misma tormenta.
- **El círculo de un barrio grande es grande y su centro es aproximado** (Palermo 3837 m, Recoleta 4084 m). No cambia quién sube, porque decide el filtro por barrio, pero lo que se dibuje con el punto y el radio de `priorityContext` abarca más que el barrio.
- **Una emergencia sin barrio no sube por una alerta por barrio.** Pasa con las de fuera de CABA.
- **Si la alerta no trae barrios del catálogo, vuelve a los puntos** con el radio del nivel y sus límites: la suba se limita a los alrededores de los puntos, y el vínculo de las abiertas descifra las emergencias abiertas una vez por contexto.
- **Alertas y eventos públicos siguen reglas distintas para las emergencias abiertas.** Con una alerta sube lo que ya estaba abierto; con un evento, no. Los módulos que recalculan la suba tienen que saberlo: lo explica la guía para consumidores del backend, y `reglaVersion` pasa a 2.
- **Una alerta que empieza más tarde** (`vigenciaDesde` en el futuro) vincula a las abiertas al llegar, pero una emergencia creada entre la llegada y el comienzo no se vincula: el vínculo al crear solo mira los contextos en curso.
- **Queda una ventana de carrera:** una emergencia que se crea mientras corre la transacción del alta puede quedar sin vincular, porque ninguna de las dos ve lo que la otra todavía no confirmó. Es una ventana de milisegundos.
- **Una sola preventiva por alerta, en el primer punto,** aunque la alerta abarque varios barrios.
- **Los niveles son criterios técnicos de demostración de Espacios,** no alertas oficiales.
- **Con la confirmación prendida, Emergencias publica un tipo más,** que tiene que seguir registrado en el catálogo del gateway. Una republicación de Espacios con otro `eventId` manda otra confirmación de la misma alerta: Analítica tiene que deduplicar por `alertaId`.

Al desplegar:

- **Los contextos climáticos que sigan activos cambian de comportamiento hasta que venzan.** Los registró el backend #122: son por punto, con el radio del nivel, y guardaron en `barrios` todos los barrios de la alerta, solo para mostrarlos. Con el filtro nuevo esos barrios se exigen, así que una emergencia nueva sube si está dentro del radio del punto **y** es de alguno de esos barrios. Tampoco vinculan las emergencias abiertas, porque eso pasa solo al dar de alta un contexto. Siguen así hasta su `vigenciaHasta` o hasta que se finalice la alerta. Si `GET /api/contextos-zona/activos` está vacío al desplegar, no hay nada que cuidar.
- **`reglaVersion` es una sola para todos los vínculos.** Desde el despliegue, también los de eventos públicos salen con 2, aunque su regla no cambió.

Neutras:

- `EmergenciaCreada` no cambia: `EVENTO_CLIMATICO` ya está en el enum de `causaTipo`.
- `contexto_zona` no cambia: la columna `barrios` (V33) pasa de dato para mostrar a filtro del vínculo.
- El ADR de radio sigue vigente para los eventos públicos. Este ADR solo abre la excepción de las alertas climáticas.

### Pendientes

- Prender `CONFIRMACION_CLIMA_ENABLED` cuando Analítica confirme que va a ingerir el tipo y el tipo esté en el catálogo del gateway.
- El backend #147 declara hoy `CONFIRMACION_CLIMA_ENABLED: "true"` en `render.yaml`. Para seguir este ADR tiene que quedar en `"false"` hasta que se cumplan las dos condiciones.
- Avisar a los módulos que recalculan la suba del cambio de regla (`reglaVersion` 2 y la sección de la alerta climática de la guía para consumidores).

### Preguntas para Producto

- [x] ¿Subir en todo el barrio de la alerta, o alcanza con los alrededores de los puntos? **En todo el barrio: un contexto por barrio, con el círculo que lo cubre y el filtro por barrio. Sin barrios del catálogo, un contexto por punto con el radio del nivel** (Producto, 18/11).
- [x] ¿Está bien que suban las emergencias que ya estaban abiertas cuando llega la alerta, aunque con los eventos públicos no pase? **Sí, solo para las alertas climáticas. Los eventos públicos siguen con la regla del ADR de radio: solo sube lo que se crea con el contexto vigente** (Producto, 18/11).
- [x] ¿Emergencias confirma por el bus cada alerta que recibe, con el record de Residuos? **Sí, pero apagado hasta que Analítica confirme que lo va a ingerir y el tipo esté en el catálogo del gateway** (Producto, 18/11).

Se mantienen las respuestas del ADR anterior (Producto, 17/11): los radios de 1000 m (`CRITICA`) y 500 m (`ALTA`), que ahora valen para las alertas sin barrios del catálogo; la emergencia `PREVENTIVA` con `ALTA` y `CRITICA`; y el banner del tablero como una vista, sin eventos nuevos en el bus.

## Historial

- 2026-11-18: creación del ADR (estado: propuesta). Reemplaza a [Responder a las alertas climáticas de Espacios](responder-a-las-alertas-climaticas-de-espacios.md) con la propuesta de Santino (PR #48 de este repositorio y rama `feat/recuperada-confirmacion-clima`; backend #146 y #147) y las respuestas de Producto del 18/11.
- 2026-11-18: cambio de estado: propuesta → aceptada
