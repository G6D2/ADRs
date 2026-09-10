# Usar DBSCAN para agrupar emergencias

- Estado: propuesta
- Fecha: 2026-09-09
- Decisores: @juanimoli, @Matiamistadi
- Reemplaza a: —
- Reemplazada por: —

Propuesto y aprobado como decisión de diseño en el issue [#10](https://github.com/G6D2/ADRs/issues/10).
La aceptación formal del ADR se realizará antes del merge, luego de la aprobación de Producto en este PR.

## Contexto

Este ADR se apoya en [Detectar incidentes y priorizar con motor determinístico](detectar-incidentes-y-priorizar-con-motor-deterministico.md), que define el contrato `IncidentDetectionStrategy` (con `RuleBasedGrouper` como estrategia base siempre disponible) y un Risk Engine determinístico como único responsable de calcular `incidentPriority`. Con ese contrato ya definido, falta decidir cuál es concretamente nuestro componente de IA/ML/I+D, evitando construir algo decorativo que no resuelva ningún problema real.

Hoy no existe ningún componente de IA/ML en el código: cero dependencias de IA/ML en el backend, ninguna librería de clustering instalada.

## Decisión

- Vamos a que DBSCAN implemente `DbscanGrouper`, la estrategia primaria de diseño de `IncidentDetectionStrategy` — decide únicamente membresía (qué emergencias pertenecen a un mismo incidente), nunca score ni prioridad. La prioridad la sigue calculando siempre el Risk Engine determinístico definido en el ADR anterior, sobre los miembros que `DbscanGrouper` (o el fallback `RuleBasedGrouper`, ante falla técnica) identifique.
- Vamos a que `DbscanGrouper` corra enteramente dentro del backend, sobre datos ya descifrados en memoria para ese propósito — no envía nada a ningún proveedor externo.
- Vamos a configurar `DbscanGrouper` mediante los parámetros propios de DBSCAN, principalmente `eps` y `minPts`. `emergencyType` y la ventana temporal se utilizarán como prefiltrado de candidatos antes del clustering. Los valores definitivos de DBSCAN no se consideran calibrados en este ADR: se ajustarán en Sprint 3-4 mediante el dataset sintético y quedarán configurables.
- Vamos a invertir el orden de implementación respecto al orden de diseño: `RuleBasedGrouper` se construye primero, en el Hito 1. `DbscanGrouper` se implementa recién en Sprint 3-4, sobre el mismo contrato, con una demo sobre datos sintéticos — es cuando la cátedra evalúa el componente de IA en la entrega integrada. Esos datos sintéticos se van a presentar explícitamente como tales en la demo: no representan patrones reales de la ciudad, solo el mecanismo del algoritmo.
- Vamos a dejar un eventual clasificador de IA externo como stretch goal explícitamente fuera de alcance de esta cursada, salvo que sobre tiempo en Sprint 5. No hay proveedor ni costo definido por ahora; si se retoma, se decide en ese momento con datos agregados y anonimizados, aprobado por Producto. Si en algún momento un PR envía datos de emergencias fuera del backend, requiere revisión obligatoria de Seguridad (@valenfiumana).
- Vamos a que cada decisión de escalado guarde el score, los factores (reportes, usuarios únicos, distancia, tipo), un conjunto fijo de `reasonCodes`, y la versión del algoritmo vigente al momento de la decisión — para que decisiones viejas sigan siendo explicables aunque se recalibren los parámetros después.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| IA solo para severidad individual, sin tocar incidentes | Acotado, fácil de aislar y testear. | No resuelve nada del problema de agrupación de incidentes; queda aislado del contrato ya definido. |
| **DBSCAN como componente de ML, implementando `IncidentDetectionStrategy`** | Clustering no supervisado que no requiere dataset etiquetado ni una fase de entrenamiento supervisado; corre internamente sin enviar datos a terceros; se integra directamente en el contrato de fallback ya definido. | Agrega una dependencia de clustering hoy inexistente y requiere definir/calibrar parámetros como `eps` y `minPts` sobre datos sintéticos en Sprint 3-4. |
| Modelo supervisado de severidad entrenado con datos propios | En teoría es el ML más "vistoso". | No existe dataset etiquetado y no lo va a haber a tiempo dentro de la cursada — se descarta explícitamente. |
| DBSCAN + clasificador de IA externo opcional | Cubre agrupación (DBSCAN) e interpretación cualitativa (clasificador) en un solo diseño. | El clasificador externo suma latencia, costo, y una superficie de riesgo de privacidad que DBSCAN no tiene; no se justifica antes de tener resuelto lo esencial. |

## Consecuencias

Se hace más fácil:

- Cumplir el requisito de IA/ML/I+D con un componente real y con valor, sin construir un segundo sistema de IA redundante con el Risk Engine.
- Reducir la exposición externa de datos: DBSCAN procesa internamente las ubicaciones descifradas en memoria y no requiere enviar información de emergencias a terceros.
- Reutilizar el contrato de fallback ya definido en el ADR anterior, sin lógica nueva.

Se hace más difícil:

- Sumar y mantener una dependencia de librería de clustering.
- Demostrar `DbscanGrouper` en Sprint 3-4 con datos sintéticos: hay que dejar explícito en la demo que esos datos no representan patrones reales de la ciudad, solo el mecanismo del algoritmo.
- Si en el futuro se suma el clasificador externo, va a requerir manejo de timeouts/fallback para una llamada externa que hoy no existe, y revisión de Seguridad antes de cualquier envío de datos.

## Historial

- 2026-09-09: creación del ADR (estado: propuesta), formalizando la decisión tomada en G6D2/ADRs#10.
