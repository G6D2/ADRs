# Modelar incidentes con identificador propio

- Estado: aceptada
- Fecha: 2026-09-09
- Decisores: @juanimoli, @Matiamistadi
- Reemplaza a: —
- Reemplazada por: —

Propuesto y aprobado como decisión de diseño en el issue [#10](https://github.com/G6D2/ADRs/issues/10).
La aceptación formal del ADR se realizará antes del merge, luego de la aprobación de Producto en este PR.

## Contexto

El módulo de Emergencias no detecta cuándo múltiples reportes de distintos ciudadanos, cercanos en espacio y tiempo, corresponden a un mismo incidente. Antes de poder detectarlos (ver el ADR "Detectar incidentes y priorizar con motor determinístico") hace falta resolver el modelo de identidad: hoy cada emergencia tiene un `emergencyId` (UUID generado en `PanicButtonController.triggerPanicButton()`), y el Roadmap define conceptualmente un `correlationId` para trazar el ciclo de vida de una emergencia individual (pánico → priorización → despacho), aunque no estaba implementado. No existía ningún concepto de agrupación entre emergencias distintas.

## Decisión

- Vamos a introducir `incidentId` como concepto nuevo e independiente de `correlationId`, con `Incident` como agregado propio (tabla `incidents`, en una migración nueva cuyo número se define al implementarla), en vez de reutilizar `correlationId` para agrupar emergencias.
- Vamos a definir `correlationId` como igual a `emergencyId` en el momento de creación de la emergencia, persistido en una columna propia — sin generarlo por separado — para cumplir el contrato ya asumido por el Roadmap y por Analítica.
- Vamos a agregar `incident_id` como columna nullable/FK en `emergencies`, que se completa cuando el detector de incidentes determina que la emergencia pertenece a un incidente.
- Vamos a limitar el alcance del Hito 1 a que un `Incident` solo pueda crecer (sumar emergencias); la fusión o división de incidentes queda fuera de alcance.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| Reutilizar `correlationId` para agrupar | No agrega ningún campo nuevo al modelo de datos. | Rompe la semántica ya definida (trazar el ciclo de vida de UNA emergencia); obliga a mutar un identificador ya publicado cuando se detecta la agrupación después del hecho, rompiendo trazabilidad para quien ya lo haya indexado. |
| **Introducir `incidentId` separado, con `Incident` como entidad propia** | Cada identificador conserva una única responsabilidad; `correlationId` no se toca; permite consultar el estado agregado de un incidente (reportes, usuarios únicos, prioridad) con una sola query. | Requiere una tabla, una FK y una migración nuevas; hay que definir reglas de reconciliación para incidentes que se fusionan o se dividen. |
| No modelar `Incident`, calcular agregados on-the-fly | Evita una tabla nueva. | No deja un lugar único y barato donde consultar el estado del incidente; cada consulta del dashboard del Dispatcher recalcularía el agregado sobre N filas. |

## Consecuencias

Se hace más fácil:

- Mantener `correlationId` e `incidentId` con responsabilidades separadas, sin mezclar trazabilidad individual con agrupación colectiva.
- Consultar el estado de un incidente (reportes, usuarios únicos, prioridad) con una sola query desde el dashboard del Dispatcher.
- Implementar `correlationId`: se deriva de `emergencyId`, ya existente, sin lógica nueva de generación.

Se hace más difícil:

- Mantener una tabla, una FK y una migración adicionales.
- Representar correctamente un escenario donde dos incidentes detectados por separado en realidad correspondían al mismo evento: en el Hito 1 un incidente solo crece, no se fusiona ni se divide — queda documentado como limitación conocida.

## Historial

- 2026-09-09: creación del ADR (estado: propuesta), formalizando la decisión tomada en G6D2/ADRs#10.
- 2026-09-10: cambio de estado: propuesta → aceptada
