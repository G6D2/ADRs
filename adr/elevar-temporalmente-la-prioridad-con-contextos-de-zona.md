# Elevar temporalmente la prioridad con contextos de zona

- Estado: aceptada
- Fecha: 2026-10-08
- Decisores: @Salterm27, @Matiamistadi (motor de riesgo), @luccaperazzo (reglas de entrada). Los confirma Producto (@juanimoli).
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Necesitamos subir temporalmente la prioridad de las emergencias de una zona mientras dura una situación de la ciudad. Hoy hay dos situaciones confirmadas y la lista va a crecer:

- Un evento en un espacio público (caso de reserva de espacio, con Espacios).
- Una alerta climática (caso de evento climático).

La prioridad de cada emergencia la calcula hoy el motor de riesgo de [Detectar incidentes y priorizar con motor determinístico](detectar-incidentes-y-priorizar-con-motor-deterministico.md). Los eventos de otros módulos entran por [Consumir eventos externos con un listener y reglas por evento](consumir-eventos-externos-con-un-listener-y-reglas-por-evento.md), y la zona de cada emergencia es el barrio, según [Guardar el barrio de la emergencia al crearla](guardar-el-barrio-de-la-emergencia-al-crearla.md).

Las fuerzas que nos obligan a decidir:

- **Es temporal.** La prioridad tiene que volver a su valor cuando la situación termina. Todo diseño que suba la prioridad necesita una forma confiable de bajarla.
- **Las situaciones van a ser varias.** El roadmap las trataba como dos trabajos separados (alertas por zona en la iteración 12, mapper de Espacios en la 13).
- **El equipo es junior y quedan tres iteraciones**, contando la 12 en curso. No podemos sumar lógica difícil de depurar.
- **Render se duerme.** Un proceso que tiene que correr a una hora exacta no es confiable en el plan gratuito.
- **Hay que poder explicarlo.** El dispatcher, y el profesor en la demo, van a preguntar por qué subió una prioridad.

## Decisión

Vamos a modelar un solo concepto, el **contexto de zona**, y a sumar una **prioridad de despacho** calculada al leer. La prioridad efectiva que ya existe no cambia.

**Contexto de zona.** Una tabla `contexto_zona` con estos campos:

| Campo | Para qué |
| --- | --- |
| `tipo` | `EVENTO_ESPACIO_PUBLICO`, `ALERTA_CLIMATICA` y los que vengan |
| `barrio` | La zona afectada. Siempre tiene un valor |
| `vigente_desde`, `vigente_hasta` | La ventana en la que el contexto pesa |
| `origen`, `referencia_externa` | Qué módulo lo emitió y el identificador del objeto en ese módulo |
| `causation_id` | El `eventId` del evento que lo creó, para trazabilidad. La deduplicación la hace el listener |
| `source_updated_at` | Cuándo ocurrió, según el evento o `metadata.receivedAt`. Sirve para descartar lo que llega viejo |
| `dado_de_baja_en` | Si se cerró antes de su vencimiento |

**Prioridad de despacho.** Se calcula en este orden, y no se guarda:

1. La prioridad efectiva de hoy: `effectivePriority`, el máximo entre la severidad y la prioridad del incidente (`src/domain/severity.js`).
2. Un nivel más si existe un contexto vigente en el barrio de la emergencia en el momento de la lectura.
3. La prioridad manual, cuando exista, por encima de todo.

**El cálculo está en SQL.** El listado ordena y pagina en la base (`ORDER BY GREATEST(...)` con `LIMIT/OFFSET` en `findPage`). El ajuste va en una vista o en una expresión dentro de esa query, con un `EXISTS` sobre `contexto_zona` por barrio y vigencia. Si se sumara en JS después de traer la página, el orden se rompería entre páginas. El detalle usa la misma expresión.

**Entrada.** Un contexto se da de alta o de baja por un servicio de aplicación. Lo invocan las reglas del listener o el dispatcher desde el tablero:

| Evento | Efecto |
| --- | --- |
| `com.citypass.espacios.EventoComunitarioPublicado` | Alta de `EVENTO_ESPACIO_PUBLICO`, con el barrio derivado de `latitud` y `longitud`, y la ventana `inicio`–`fin` |
| `com.citypass.espacios.EventoCancelado` | Baja del contexto de ese `eventoId` |
| `com.citypass.espacios.CriticidadClimaticaConfirmada` | Alta de `ALERTA_CLIMATICA`, un contexto por cada barrio de `ubicaciones`, con la ventana `vigenciaDesde`–`vigenciaHasta` |
| `com.citypass.espacios.CriticidadClimaticaFinalizada` | Baja de los contextos de ese `alertaId` |

**Alertas de toda la ciudad.** Las acotamos por barrio: no hay contextos con barrio nulo. Una alerta sin ubicaciones no genera contextos y queda solo en el aviso. Cargarla en todos los barrios subiría todo un nivel, y lo único que cambiaría en la cola es que las altas quedarían empatadas con las críticas.

**Aviso climático.** Cuando llega una alerta climática avisamos, haya o no emergencias abiertas: un banner en el tablero del dispatcher y un evento al bus, con nombre tentativo `ContextoZonaActivado`. Si al cerrar los contratos ningún módulo lo consume, se saca y queda el banner.

**Salida.** Vamos a publicar `EmergenciaPriorizada` cuando se da de alta o de baja un contexto y hay emergencias abiertas en ese barrio. El evento incluye el contexto como motivo y su `causationId`.

**Tablero.** El dispatcher ve la prioridad de despacho y el motivo, por ejemplo "elevada por alerta climática en Palermo".

### Reglas

Son valores de arranque con un criterio, y quedan configurables por entorno.

1. **La zona es el barrio.** No usamos radios ni geometría.
2. **El ajuste es de un nivel**, con tope en la prioridad máxima. Un nivel alcanza para reordenar la cola sin que una emergencia menor pase por delante de una crítica real.
3. **Los contextos no se acumulan.** Dos contextos vigentes en el mismo barrio suben un nivel, no dos. Así evitamos que todo termine en prioridad máxima un día de tormenta con festival.
4. **La prioridad manual gana siempre**, cuando exista (issue #25 del board). Mientras no esté hecha, el contexto funciona igual: no hay nada que respetar.
5. **Aplica a todos los tipos de emergencia**, incluido el tipo nuevo de tránsito.
6. **Todo contexto nace con vencimiento.** Si el evento de origen no trae fecha de fin, la regla del listener pone una por defecto.
7. **Un contexto se puede dar de baja antes de su vencimiento**, por evento del módulo de origen o a mano.
8. **Lo que llega viejo no pisa lo nuevo.** Una baja puede llegar antes que su alta, así que toda escritura aplica solo si es más nueva que `source_updated_at`, y la baja es lógica. Sale del diseño de `docs/prioridad-eventos-publicos.md` del backend.

### Plan B

Si el listener no llega a la demo, los contextos se dan de alta desde el tablero y el cálculo es el mismo. Si el barrio no llega, queda el aviso en el tablero, sin suba automática.

## Alternativas consideradas

Evaluamos seis. Elegimos la C: es la única donde la prioridad baja sola, sumar una situación nueva no toca el motor de riesgo y se ordena en SQL sin descifrar ubicaciones.

| Alternativa | Cómo funciona | A favor | En contra |
| --- | --- | --- | --- |
| A. Un factor por caso | El motor de riesgo suma un factor "alerta climática" y otro "evento en espacio público", cada uno con su tabla. Es lo que decía el roadmap. | Directo de entender. Cada caso se desarrolla por separado. | Cada situación nueva es una tabla, un factor y una pantalla más. Duplica la lógica de vigencia y de zona. |
| B. Pisar la prioridad guardada | Al llegar el evento se actualiza la prioridad de las emergencias abiertas de la zona. Al terminar, se revierte. | No cambia las consultas ni el tablero. | Hay que acordarse de bajarla: necesita un job a horario, que Render dormido no garantiza. Se pierde la prioridad original y no se sabe por qué subió. Dos situaciones superpuestas se pisan al revertir. |
| **C. Contexto de zona + prioridad de despacho calculada** | Una tabla genérica de contextos con vigencia. La prioridad guardada no se toca; la de despacho se calcula al leer. | La prioridad baja sola al vencer el contexto. Una situación nueva es una regla. El motivo queda a la vista. | Toda lectura de prioridad pasa por el cálculo. El vencimiento por horario no publica evento. |
| D. Solo prioridad manual (#25) | El dispatcher ve un aviso de la situación y sube la prioridad a mano. | Casi no hay código nuevo. | No es automático, que es lo que pide el caso. No escala con muchas emergencias en una tormenta. Nadie la baja después. |
| E. Servicio aparte que recalcula | Un consumer en otro repo y otra infraestructura escucha los eventos y actualiza las prioridades. | Aísla la integración del backend. | Segundo deploy y segundo servicio que se duerme. Base compartida o API interna. Mantiene el problema de B: sigue pisando la prioridad. |
| F. Zona por radio (`docs/prioridad-eventos-publicos.md` del backend) | La zona guarda coordenadas y radio. Al leer, se descifran las emergencias candidatas en la aplicación, se mide la distancia y se ordena en SQL con la lista de ids. | Más preciso que el barrio. Con la lista de ids, paginar sigue siendo correcto. | Descifra ubicaciones en cada lectura, y hay que definir un radio por tipo de evento. Una alerta climática no tiene radio natural. El barrio igual hace falta, porque lo piden otros módulos. Si se acepta este ADR, hay que alinear ese documento. |

La alternativa D no se descarta del todo: la prioridad manual queda como complemento, con precedencia sobre el cálculo automático.

## Consecuencias

Se hace más fácil:

- Que la prioridad vuelva sola cuando vence el contexto, sin scheduler ni job a horario.
- Que los casos de reserva de espacio y de evento climático compartan el mismo código. Cada situación nueva cuesta una regla.
- Conservar la prioridad original y mostrar el motivo de la suba.
- Calcular la prioridad de despacho sin infraestructura nueva. La entrada automática de contextos sí depende del listener y de su hosting. Mientras no esté, los contextos se dan de alta desde el tablero.

Se hace más difícil, y lo aceptamos a propósito:

- **Toda lectura de prioridad tiene que pasar por el cálculo.** Si una consulta ordena sin el ajuste, el tablero queda mal ordenado. Lo resolvemos con una vista o expresión compartida y un test que cubra el orden entre páginas.
- **El vencimiento por horario no publica evento.** Los otros módulos se enteran de la suba, pero no de la baja automática. Si alguno lo necesita, se suma un job de un minuto que detecta transiciones.
- **El barrio es una zona gruesa.** Una emergencia a veinte cuadras del festival sube igual si está en el mismo barrio.
- **Un nivel fijo no distingue gravedad.** Una alerta roja y una amarilla pesan lo mismo en esta versión.

Riesgos:

- **Depende de que la emergencia guarde el barrio**, que hoy no lo guarda. Esa decisión ya está aprobada, pero falta implementarla, y bloquea esta.
- **Contextos que nunca se cierran.** Lo mitiga la regla 6: todo contexto nace con vencimiento.

Impacto en el roadmap:

- El trabajo de la iteración 12 (alertas por zona y factor en el motor de riesgo) pasa a ser este mecanismo genérico.
- El caso de reserva de espacio deja de ser el primero a recortar: sobre esta base cuesta una regla.

### Revisión y preguntas abiertas

Revisamos esta decisión si un módulo necesita enterarse de la baja automática, si el barrio resulta demasiado grueso en las pruebas cruzadas, o si aparece una situación que tiene que pesar más de un nivel.

Respondido al 2026-11-02:

- [x] Quién emite la alerta climática: Espacios, con `CriticidadClimaticaConfirmada` y `CriticidadClimaticaFinalizada`.
- [x] Qué evento publica Espacios para un evento comunitario: `EventoComunitarioPublicado`. No trae barrio, pero sí coordenadas, y el barrio se deriva. Su `zona` es texto libre.

Queda por confirmar:

- [ ] `ReservaConfirmada` trae `espacioId`, `inicio` y `fin`, sin ubicación. Una reserva sin evento comunitario no genera contexto, salvo que Espacios sume las coordenadas.
- [ ] Si Movilidad, Reclamos o Rutas necesitan el evento de baja de prioridad.
- [ ] El vencimiento por defecto de un contexto que llega sin fecha de fin.

## Historial

- 2026-10-08: creación del ADR (estado: propuesta)
- 2026-10-16: se separan en ADRs propios el listener de eventos externos y el barrio de la emergencia; la deduplicación pasa al listener; se corrige la afirmación de que no agrega infraestructura.
- 2026-11-02: se aplica el review del 2026-10-18: prioridad de despacho, cálculo en SQL, aviso climático, prioridad manual y alertas de toda la ciudad. Se suman los eventos reales de Espacios, el orden de llegada y el plan B, y la zona por radio queda como alternativa descartada.
- 2026-11-05: cambio de estado: propuesta → aceptada
