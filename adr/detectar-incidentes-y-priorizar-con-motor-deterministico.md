# Detectar incidentes y priorizar con motor determinístico

- Estado: propuesta
- Fecha: 2026-09-09
- Decisores: @juanimoli, @Matiamistadi
- Reemplaza a: —
- Reemplazada por: —

Propuesto y aprobado como decisión de diseño en el issue [#10](https://github.com/G6D2/ADRs/issues/10).
La aceptación formal del ADR se realizará antes del merge, luego de la aprobación de Producto en este PR.

## Contexto

Este ADR se apoya en [Modelar incidentes con identificador propio](modelar-incidentes-con-identificador-propio.md), que introduce `incidentId` e `Incident` como agregado propio, separado de `correlationId`. Con ese modelo de identidad resuelto, falta definir cómo se detecta que varias emergencias forman un mismo incidente y cómo eso escala la prioridad — de forma determinística, auditable y testeable, sin bloquear nunca la creación ni el despacho de una emergencia si el mecanismo de agrupación falla.

Hoy no existe ningún mecanismo de priorización automática: `severity` es un `TEXT` que el cliente envía sin validación server-side, sin enum ni CHECK constraint, y no hay ningún `PriorityService`. La columna `emergency_type` existe en la base pero se pierde en el camino entre el frontend y el `INSERT` — nunca llega a persistirse. La ubicación se cifra a nivel de aplicación antes de guardarse; ninguna consulta SQL puede filtrar ni calcular distancia sobre ella, hay que descifrar en memoria.

## Decisión

- Vamos a definir un contrato `IncidentDetectionStrategy` con dos implementaciones que solo deciden membresía, nunca score ni prioridad: `RuleBasedGrouper` (estrategia base, siempre disponible: mismo `emergencyType` + ventana temporal configurable + Haversine < X + mínimo de ciudadanos únicos) y `DbscanGrouper` (estrategia primaria de ML a nivel de diseño, ver el ADR de IA/ML).
- Vamos a que un Risk Engine determinístico sea el único componente que calcule `incidentPriority`, a partir del `Incident` ya agrupado (cantidad de reportes, usuarios únicos, cercanía, recencia, severidad), y que garantice `priority = max(priorityIndividual, incidentPriority)` — ninguna señal de agrupación o de IA puede reducir una prioridad individual ya establecida.
- Vamos a distinguir explícitamente falla técnica de resultado válido en la estrategia de agrupación: si `DbscanGrouper` falla técnicamente (excepción, timeout, dependencia no disponible), se usa `RuleBasedGrouper` como fallback; si `DbscanGrouper` corre bien y no encuentra ningún cluster, no se fuerza el fallback — es un resultado válido del algoritmo.
- Vamos a correr la detección dentro del mismo proceso del backend, como job periódico cada 60 segundos, reutilizando el patrón ya existente en `src/security/location-retention.js`, controlable con la variable de entorno `INCIDENT_DETECTION_ENABLED` para apagarlo sin desplegar — sin microservicio nuevo ni Kafka Streams para esta entrega.
- Vamos a invertir el orden de implementación respecto al orden de diseño: `RuleBasedGrouper` se construye primero, en el Hito 1 (24/09), con estos valores de arranque para la demo — de entorno, no calibrados, se ajustan con datos sintéticos en Sprint 3 — `INCIDENT_RADIUS_M=300`, `INCIDENT_WINDOW_MIN=15`, `INCIDENT_MIN_CITIZENS=2`. `DbscanGrouper` se implementa recién en Sprint 3-4, sobre el mismo contrato, con demo sobre datos sintéticos — es cuando la cátedra evalúa el componente de IA en la entrega integrada.
- Vamos a resolver, como prerrequisito técnico antes de implementar lo anterior: (a) la persistencia de `emergencyType` con un enum cerrado de 4 tipos y CHECK en base (hoy se pierde entre el controller y el repository), y (b) la normalización y validación server-side de `severity` a una escala ordinal en español y mayúsculas — `BAJA < MEDIA < ALTA < CRITICA` — con CHECK en base y valor por defecto calculado por tipo en lugar de aceptar el que mande el cliente; la misma convención de nomenclatura aplica al estado de la emergencia (`PENDIENTE`, `VALIDADA`, ...). Sin esta normalización, `max()` no tiene un resultado bien definido.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| Reglas por umbral fijo, como única estrategia | Trivial de implementar y explicar. | Salto discontinuo entre niveles de prioridad; umbrales difíciles de justificar con precisión; no deja margen para incorporar clustering después sin rediseñar. |
| Scoring ponderado determinístico, como única estrategia | Escalada gradual y defendible. | Por sí sola no aporta ningún componente de ML. |
| DBSCAN como único mecanismo, decidiendo membresía y prioridad a la vez | Aparenta ser la solución más orientada a ML. | Mezcla membresía (donde el ML aporta valor) con prioridad (que debe ser determinística y estar disponible aunque el ML falle); si DBSCAN se cae, el sistema queda sin poder calcular ninguna escalada. |
| **Dos estrategias de agrupación desacopladas de un único Risk Engine, con fallback ante falla técnica** | La prioridad queda siempre disponible, simple y testeable sin importar qué estrategia de agrupación esté activa; permite sumar DBSCAN como ML real de esta entrega sin comprometer la robustez. | Exige mantener dos implementaciones de agrupación y su contrato de fallback — más disciplina de diseño que una sola estrategia. |

## Consecuencias

Se hace más fácil:

- Probar la prioridad con un invariante simple (`resultado ≥ piso`), independientemente de qué estrategia de agrupación esté activa.
- Incorporar DBSCAN como el componente de ML de esta entrega sin comprometer la robustez del sistema si falla o todavía no está implementado.
- Reutilizar infraestructura existente (`location-retention.js`) para el job periódico, sin sumar una cola de mensajería nueva.

Se hace más difícil:

- Mantener dos implementaciones de agrupación (`RuleBasedGrouper` y `DbscanGrouper`) y su contrato de fallback.
- Calibrar los umbrales/pesos de `RuleBasedGrouper`, que siguen siendo manuales.
- El job corre sobre datos descifrados en memoria — costo de CPU que hoy no existe, aunque acotado a la escala de esta cursada.
- Demostrar `DbscanGrouper` en Sprint 3-4 con datos sintéticos: hay que dejar explícito en la demo que esos datos no representan patrones reales de la ciudad, solo el mecanismo del algoritmo.

## Historial

- 2026-09-09: creación del ADR (estado: propuesta), formalizando la decisión tomada en G6D2/ADRs#10.
