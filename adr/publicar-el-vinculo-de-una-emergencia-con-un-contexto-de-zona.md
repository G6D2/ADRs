# Publicar el vínculo de una emergencia con un contexto de zona

- Estado: propuesta
- Fecha: 2026-11-13
- Decisores: (a confirmar en la revisión del PR; hace falta Security por la ubicación, Analítica como consumidor y el Grupo 1 por el catálogo del bus)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

[Elevar temporalmente la prioridad con contextos de zona](elevar-temporalmente-la-prioridad-con-contextos-de-zona.md)
sube un nivel la prioridad de despacho de las emergencias que se crean
cerca de un evento en curso (hoy, un evento comunitario de Espacios). En el
backend eso quedó así:

- Al crear la emergencia, `linkZoneContexts` la compara con los contextos
  en curso y, si está a no más de `radio_metros` (500 m) del punto del
  evento, guarda el par en `emergencia_contexto_zona`. Se mide con la
  coordenada exacta, una sola vez, y el vínculo no se recalcula.
- La suba no se guarda: `dispatchPriority` y `priorityContext` se calculan
  al leer y solo los ve el dispatcher por HTTP.
- `EmergenciaPriorizada` es interna: no está en el contrato del bus y el
  outbox la marca `SKIPPED`. La "Salida" de aquel ADR, que prometía
  publicarla al dar de alta o de baja un contexto, no se implementó.

Para el resto de los módulos, el backend publicó una guía
(`docs/prioridad-por-contexto-para-consumidores.md`) para que **recalculen**
el vínculo con lo que ya viaja por el bus: `EventoComunitarioPublicado`,
`EventoCancelado`, `EmergenciaCreada` y `EmergenciaActualizada`.

Analítica necesita **contar las emergencias elevadas por evento**. Con la
guía tendría que armar su propia tabla de vínculos, y esa tabla no puede
coincidir con la nuestra:

- **Coordenadas:** solo recibe la coordenada redondeada a 3 decimales
  (~100 m, ver [Publicar coordenadas aproximadas en claro en los eventos del bus](publicar-coordenadas-aproximadas-en-claro-en-los-eventos-del-bus.md)).
  Cerca del borde de los 500 m el resultado cambia de lado.
- **Radio:** Espacios no lo informa; es un valor nuestro, configurable, que
  se guarda por contexto al registrarlo.
- **Vigencia:** depende de cuándo *recibimos* cada publicación del evento
  (primera publicación sin `inicio`, emergencia creada justo antes de que
  llegue la publicación). La propia guía reconoce que eso no se puede
  reproducir desde afuera.
- **Mantenimiento:** cada cambio de umbral, radio o regla en el backend
  separa en silencio los números de Analítica de los del tablero.

Es el peor tipo de dato malo: parece correcto y nadie se entera de que no
lo es. Y deja dos equipos manteniendo la misma lógica.

Fuerzas y restricciones:

- **Una sola fuente de verdad.** El vínculo ya existe en nuestra base; la
  pregunta es cómo sale, no cómo se calcula.
- **Atomicidad.** Lo que se publica tiene que coincidir con lo que quedó
  guardado. El outbox transaccional ya resuelve eso para los demás eventos.
- **Entrega *at-least-once*.** El bus puede entregar duplicados.
- **Inmutabilidad.** Un evento publicado no se corrige. Si cambia una regla,
  los consumidores tienen que poder distinguir vínculos viejos de nuevos.
- **Privacidad.** Un vínculo dice "esta emergencia estuvo a menos de 500 m
  de este punto". Por eso el backend lo borra junto con la ubicación al
  anonimizar (72 h, `LOCATION_RETENTION_HOURS`). Para todas las categorías
  salvo `PANICO` eso es menos preciso que la coordenada de ~100 m que ya
  publicamos. Para `PANICO`, el ADR de coordenadas tiene pendiente bajar a
  2 decimales (~1 km): ahí el vínculo daría más precisión que la
  coordenada.
- **Catálogo.** Todo tipo de evento nuevo se registra en el gateway del
  Grupo 1, y los contratos de Emergencias se cerraron en
  G6D2/demo-repository#48.
- **Estabilidad de `EmergenciaCreada`.** La consumen cinco módulos; no
  conviene cambiarla por la necesidad de uno.

## Decisión

Vamos a publicar el vínculo que ya calculamos, como un evento propio, en
lugar de pedir a los consumidores que lo recalculen.

1. **Evento nuevo `EmergenciaVinculadaAContextoZona`**
   (`com.citypass.emergencias.EmergenciaVinculadaAContextoZona`), uno por
   cada fila de `emergencia_contexto_zona`. `data`:

   | Campo | Contenido |
   |---|---|
   | `emergenciaId` | Id de la emergencia |
   | `contextoId` | Id del contexto en Emergencias |
   | `contextoTipo` | `EVENTO_ESPACIO_PUBLICO` o `ALERTA_CLIMATICA` |
   | `contextoOrigen` | Módulo que originó el contexto (`espacios`) |
   | `referenciaExterna` | Id en el módulo de origen (el `eventoId` de Espacios). Es la clave para agrupar por evento |
   | `radioMetros` | Radio con el que se decidió el vínculo |
   | `vinculadaEn` | Cuándo se creó el vínculo (`created_at` de la fila) |
   | `reglaVersion` | Versión de la regla de vinculación. Arranca en `1`; sube si cambia la forma de decidir el vínculo |
   | `backfill` | `true` si el evento salió de la carga inicial (punto 6) y no del alta de la emergencia |

   No lleva coordenadas, ni de la emergencia ni del evento, ni datos del
   ciudadano.

2. **Mismo commit que el vínculo.** El evento se encola en `outbox_events`
   en la misma sentencia que inserta la fila del vínculo. Si el vínculo
   existe, el evento existe; si la transacción falla, no existe ninguno. Si
   la fila ya estaba (`ON CONFLICT DO NOTHING`), no se encola otro.

3. **Significado.** "Esta emergencia quedó vinculada a este contexto al
   crearse, y mientras los dos estuvieron vigentes su prioridad de despacho
   estuvo un nivel arriba". Analítica cuenta eso; no recalcula vigencias.
   Una emergencia vinculada estuvo elevada en la práctica, porque al
   crearse está abierta y el contexto en curso.

4. **Clave de deduplicación: `(emergenciaId, contextoId)`.** Un consumidor
   que recibe dos veces el mismo par lo cuenta una vez.

5. **Correcciones como hechos nuevos.** Nunca se reescribe un evento
   publicado. Si un vínculo se tiene que anular, se publica un evento de
   desvinculación (fuera del alcance de este ADR). Un cambio de regla sube
   `reglaVersion`.

6. **Carga inicial (backfill).** Un script encola, una vez y con
   `backfill: true`, los vínculos que ya existen y no tienen evento. Es
   idempotente. **Solo puede recuperar los vínculos que siguen en la
   base**: los de emergencias ya anonimizadas se borraron, así que la
   historia de Analítica empieza unas 72 h antes del despliegue.

7. **Privacidad.**
   - Aplica la regla del ADR de coordenadas para eventos nuevos con
     ubicación: el vínculo no puede dar más precisión que la coordenada
     publicada.
   - **[A definir con Security]** Mientras no esté decidida la precisión de
     `PANICO`, **no se publican vínculos de emergencias `PANICO`**. Si
     Security confirma 3 decimales para `PANICO`, se publican; si baja a 2,
     quedan afuera y Analítica lo sabe por este ADR.
   - Al anonimizar la ubicación de una emergencia se borran, además del
     vínculo, sus eventos de vínculo que sigan en `outbox_events`. Lo que ya
     salió al bus queda bajo la retención del Grupo 1, igual que las
     coordenadas.

8. **Activación.** El encolado queda detrás de una variable
   (`EVENTO_VINCULO_CONTEXTO_ENABLED`, apagada por defecto) hasta que el
   tipo esté registrado en el catálogo del gateway. Con la variable
   apagada no se encola nada; al encenderla se corre el backfill para no
   perder los vínculos del medio.

9. **Documentación.** El JSON Schema del evento va en `docs/contratos/`
   junto a los otros, y la guía para consumidores pasa a recomendar este
   evento. El recálculo queda como alternativa para quien no pueda
   consumirlo.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **Evento propio desde el outbox** (elegida) | Sale de la misma fuente que usa el tablero. Atómico con el vínculo. No toca contratos existentes. Admite vínculos posteriores a la creación o desvinculaciones a futuro sin cambiar el contrato. | Un tipo más para registrar y mantener compatible. Backfill limitado por la retención. |
| Que los consumidores recalculen (la guía actual) | Sin cambios en Emergencias. | Resultados distintos cerca del borde y en los casos que dependen de cuándo recibimos el evento. Dos equipos mantienen la misma lógica. Datos malos que no se notan. |
| Campo `contextosZona` en `EmergenciaCreada` | Sin tipo nuevo. El vínculo hoy se decide al crear. | Cambia un contrato que consumen cinco módulos por la necesidad de uno. Ata el vínculo al alta: si mañana se vincula tarde, no hay dónde publicarlo. |
| Publicar `EmergenciaPriorizada` (lo que decía la "Salida" del ADR de contextos) | Ya existe el builder. | Es un evento de orden de despacho por incidente, interno a propósito. Mezclaría dos conceptos y su contenido cambia con el motor de riesgo. |
| Exportar la tabla (vista, API o archivo para Analítica) | Simple para un solo consumidor. | Acopla a Analítica con nuestro esquema. Otro canal a mantener fuera del bus, sin las garantías del outbox. |

## Consecuencias

### Positivas

- Analítica cuenta exactamente lo que vio el dispatcher, con un `GROUP BY
  referenciaExterna` sobre eventos que ya consume, sin geometría.
- La lógica de vinculación vive en un solo lugar. Cambiar el radio o la
  regla no rompe a los consumidores: lo ven en `radioMetros` y
  `reglaVersion`.
- Cualquier otro módulo que necesite el mismo dato lo tiene sin trabajo
  extra.

### Negativas

- Un tipo de evento más para registrar en el gateway y mantener compatible
  hacia atrás.
- Hasta que Security decida, los vínculos de `PANICO` no se publican y el
  conteo de Analítica no los incluye.
- La historia anterior al despliegue se pierde salvo las ~72 h que siguen
  en la base.
- El vínculo es un dato derivado de la ubicación que sale del módulo; una
  vez en el bus, su retención no la controlamos.

### Neutras

- `EmergenciaCreada`, `EmergenciaActualizada` y `EmergenciaPriorizada` no
  cambian.
- El ADR de contextos de zona sigue vigente; este reemplaza solo su
  "Salida", que no se había implementado. Si el equipo prefiere, se puede
  marcar aquel como reemplazado parcialmente.
- El vencimiento por horario de un contexto sigue sin publicar evento; para
  contar vínculos no hace falta.

## Historial

- 2026-11-13: creación del ADR (estado: propuesta)
