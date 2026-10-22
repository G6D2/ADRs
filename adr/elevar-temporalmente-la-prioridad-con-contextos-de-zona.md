# Elevar temporalmente la prioridad con contextos de zona

- Estado: propuesta
- Fecha: 2026-10-08
- Decisores: @Sebas, @Matiamistadi (motor de riesgo), @luccaperazzo (mappers de entrada). Los confirma Producto (@juanimoli).
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Necesitamos subir temporalmente la prioridad de las emergencias de una zona mientras dura una situación de la ciudad. Hoy hay dos situaciones confirmadas y la lista va a crecer:

- Un evento en un espacio público (caso de reserva de espacio, con Espacios).
- Una alerta climática (caso de evento climático).

La prioridad de cada emergencia la calcula hoy el motor de riesgo de [Detectar incidentes y priorizar con motor determinístico](detectar-incidentes-y-priorizar-con-motor-deterministico.md). Los eventos de otros módulos entran por [Consumir eventos externos con un listener y mappers por origen](consumir-eventos-externos-con-un-listener-y-mappers-por-origen.md), y la zona de cada emergencia es el barrio, según [Guardar el barrio de la emergencia al crearla](guardar-el-barrio-de-la-emergencia-al-crearla.md).

Las fuerzas que nos obligan a decidir:

- **Es temporal.** La prioridad tiene que volver a su valor cuando la situación termina. Todo diseño que suba la prioridad necesita una forma confiable de bajarla.
- **Las situaciones van a ser varias.** El roadmap las trataba como dos trabajos separados (alertas por zona en la iteración 12, mapper de Espacios en la 13).
- **El equipo es junior y quedan seis iteraciones.** No podemos sumar lógica difícil de depurar.
- **Render se duerme.** Un proceso que tiene que correr a una hora exacta no es confiable en el plan gratuito.
- **Hay que poder explicarlo.** El dispatcher, y el profesor en la demo, van a preguntar por qué subió una prioridad.

## Decisión

Vamos a modelar un solo concepto, el **contexto de zona**, y a separar la prioridad en base y efectiva.

**Contexto de zona.** Una tabla `contexto_zona` con estos campos:

| Campo | Para qué |
| --- | --- |
| `tipo` | `EVENTO_ESPACIO_PUBLICO`, `ALERTA_CLIMATICA` y los que vengan |
| `barrio` | La zona afectada |
| `vigente_desde`, `vigente_hasta` | La ventana en la que el contexto pesa |
| `origen`, `referencia_externa` | Qué módulo lo emitió y el identificador del objeto en ese módulo |
| `causation_id` | El `eventId` del evento que lo creó, para trazabilidad. La deduplicación la hace el listener |
| `dado_de_baja_en` | Si se cerró antes de su vencimiento |

**Prioridad base y efectiva.** La base es la prioridad final que ya calcula el motor de riesgo, y se guarda como hoy. La efectiva es la base más el ajuste, si existe un contexto vigente en el barrio de la emergencia en el momento de la lectura. La efectiva no se guarda.

**Entrada.** Un contexto se da de alta o de baja por un comando de aplicación. Ese comando lo invocan los mappers del listener (uno por módulo de origen) o el dispatcher desde el tablero.

**Salida.** Vamos a publicar `EmergenciaPriorizada` cuando se da de alta o de baja un contexto y hay emergencias abiertas en ese barrio. El evento incluye el contexto como motivo y su `causationId`.

**Tablero.** El dispatcher ve la prioridad efectiva y el motivo, por ejemplo "elevada por alerta climática en Palermo".

### Reglas

Son valores de arranque con un criterio, y quedan configurables por entorno.

1. **La zona es el barrio.** No usamos radios ni geometría.
2. **El ajuste es de un nivel**, con tope en la prioridad máxima. Un nivel alcanza para reordenar la cola sin que una emergencia menor pase por delante de una crítica real.
3. **Los contextos no se acumulan.** Dos contextos vigentes en el mismo barrio suben un nivel, no dos. Así evitamos que todo termine en prioridad máxima un día de tormenta con festival.
4. **La prioridad manual (#25) gana siempre.** Si el dispatcher fijó una prioridad, el cálculo automático no la modifica.
5. **Aplica a todos los tipos de emergencia**, incluido el tipo nuevo de tránsito.
6. **Todo contexto nace con vencimiento.** Si el evento de origen no trae fecha de fin, el mapper pone una por defecto.
7. **Un contexto se puede dar de baja antes de su vencimiento**, por evento del módulo de origen o a mano.

## Alternativas consideradas

Evaluamos cinco. Elegimos la C: es la única donde la prioridad baja sola y sumar una situación nueva no toca el motor de riesgo.

| Alternativa | Cómo funciona | A favor | En contra |
| --- | --- | --- | --- |
| A. Un factor por caso | El motor de riesgo suma un factor "alerta climática" y otro "evento en espacio público", cada uno con su tabla. Es lo que decía el roadmap. | Directo de entender. Cada caso se desarrolla por separado. | Cada situación nueva es una tabla, un factor y una pantalla más. Duplica la lógica de vigencia y de zona. |
| B. Pisar la prioridad guardada | Al llegar el evento se actualiza la prioridad de las emergencias abiertas de la zona. Al terminar, se revierte. | No cambia las consultas ni el tablero. | Hay que acordarse de bajarla: necesita un job a horario, que Render dormido no garantiza. Se pierde la prioridad original y no se sabe por qué subió. Dos situaciones superpuestas se pisan al revertir. |
| **C. Contexto de zona + prioridad efectiva calculada** | Una tabla genérica de contextos con vigencia. La prioridad base no se toca; la efectiva se calcula al leer. | La prioridad baja sola al vencer el contexto. Una situación nueva es un mapper. El motivo queda a la vista. | Toda lectura de prioridad pasa por el cálculo. El vencimiento por horario no publica evento. |
| D. Solo prioridad manual (#25) | El dispatcher ve un aviso de la situación y sube la prioridad a mano. | Casi no hay código nuevo. | No es automático, que es lo que pide el caso. No escala con muchas emergencias en una tormenta. Nadie la baja después. |
| E. Servicio aparte que recalcula | Un consumer en otro repo y otra infraestructura escucha los eventos y actualiza las prioridades. | Aísla la integración del backend. | Segundo deploy y segundo servicio que se duerme. Base compartida o API interna. Mantiene el problema de B: sigue pisando la prioridad. |

La alternativa D no se descarta del todo: la prioridad manual queda como complemento, con precedencia sobre el cálculo automático.

## Consecuencias

Se hace más fácil:

- Que la prioridad vuelva sola a la base cuando vence el contexto, sin scheduler ni job a horario.
- Que los casos de reserva de espacio y de evento climático compartan el mismo código. Cada situación nueva cuesta un mapper.
- Conservar la prioridad original y mostrar el motivo de la suba.
- Calcular la prioridad efectiva sin infraestructura nueva. La entrada automática de contextos sí depende del listener y de su hosting. Mientras no esté, los contextos se dan de alta desde el tablero.

Se hace más difícil, y lo aceptamos a propósito:

- **Toda lectura de prioridad tiene que pasar por el cálculo.** Si una consulta ordena por la columna de prioridad base, el tablero queda mal ordenado. Lo resolvemos con una sola función de lectura y un test que lo cubra.
- **El vencimiento por horario no publica evento.** Los otros módulos se enteran de la suba, pero no de la baja automática. Si alguno lo necesita, se suma un job de un minuto que detecta transiciones.
- **El barrio es una zona gruesa.** Una emergencia a veinte cuadras del festival sube igual si está en el mismo barrio.
- **Un nivel fijo no distingue gravedad.** Una alerta roja y una amarilla pesan lo mismo en esta versión.

Riesgos:

- **Depende de que la emergencia guarde el barrio**, que hoy no lo guarda. Esa decisión entra en la iteración 10 y bloquea esta.
- **Contextos que nunca se cierran.** Lo mitiga la regla 6: todo contexto nace con vencimiento.

Impacto en el roadmap:

- El trabajo de la iteración 12 (alertas por zona y factor en el motor de riesgo) pasa a ser este mecanismo genérico.
- El caso de reserva de espacio deja de ser el primero a recortar: sobre esta base cuesta un mapper.

### Revisión y preguntas abiertas

Revisamos esta decisión si un módulo necesita enterarse de la baja automática, si el barrio resulta demasiado grueso en las pruebas cruzadas, o si aparece una situación que tiene que pesar más de un nivel.

Queda por confirmar:

- [ ] Quién emite la alerta climática. Si nadie la reclama, la emitimos nosotros y el contexto se da de alta desde el tablero.
- [ ] Qué evento publica Espacios para una reserva confirmada y si trae barrio y horario. Al 01/10 todavía no publicaba al bus.
- [ ] Si Movilidad, Reclamos o Rutas necesitan el evento de baja de prioridad.
- [ ] El vencimiento por defecto de un contexto que llega sin fecha de fin.

## Historial

- 2026-10-08: creación del ADR (estado: propuesta)
- 2026-10-16: se separan en ADRs propios el listener de eventos externos y el barrio de la emergencia; la deduplicación pasa al listener; se corrige la afirmación de que no agrega infraestructura.
