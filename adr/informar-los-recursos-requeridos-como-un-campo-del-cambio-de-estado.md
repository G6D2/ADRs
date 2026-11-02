# Informar los recursos requeridos como un campo del cambio de estado

- Estado: propuesta
- Fecha: 2026-10-16
- Decisores: @juanimoli (Producto), @Matiamistadi y @luccaperazzo (Back), @valenfiumana (Seguridad)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Ante una emergencia, Emergencias decide **qué tipo de recurso hace falta** (Bomberos, Policía o Ambulancia). La propuesta original era publicar un tipo de evento por recurso: `BomberosSolicitados`, `PoliciaSolicitada` y `AmbulanciaSolicitada`.

Al revisar quién los iba a leer, encontramos que:

- En CityPass+ no hay un módulo de bomberos, policía ni ambulancias.
- Al 2026-11-02, ningún repo de Reclamos, Espacios, Movilidad, Residuos ni Analítica se suscribe a esos eventos. Lo que leen de nosotros es `EmergenciaCreada` y `EmergenciaActualizada`.
- Guardamos la ubicación cifrada y con retención de 72 horas. Cualquier grupo se puede suscribir a los tipos de otro, así que mandar la ubicación exacta a un tópico nuevo va en contra de esa decisión.

Cómo se calculan los recursos se decide en [Calcular los recursos requeridos con una tabla de reglas](calcular-los-recursos-requeridos-con-una-tabla-de-reglas.md).

## Decisión

- Vamos a **no registrar un tipo de evento por recurso** mientras no haya un consumidor concreto.
- Vamos a mostrar los recursos requeridos en el **tablero del dispatcher**, que es quien los usa hoy.
- Vamos a publicarlos como un campo `recursosRequeridos` en el **evento de cambio de estado** que ya tenemos, cuando la emergencia pasa a `VALIDATED`. Ese evento hoy se llama `EmergenciaEstadoActualizado`; Espacios y Analítica esperan `EmergenciaActualizada`, y el nombre se cierra en el contrato de salida.
- Vamos a que el catálogo de recursos sea **cerrado y nuestro**: `BOMBEROS`, `POLICIA` y `AMBULANCIA`. Sumar un recurso es una decisión de Producto y no requiere un tipo de evento nuevo.
- Vamos a informar los recursos tanto para emergencias creadas en nuestra plataforma como para las originadas por eventos de otros módulos.
- Vamos a mandar el **barrio siempre** (ver [Guardar el barrio de la emergencia al crearla](guardar-el-barrio-de-la-emergencia-al-crearla.md)) y a **no sumar ubicación exacta** por este campo. Si aparece un consumidor que la necesita para despachar, se decide con el visto bueno de Seguridad.
- Vamos a **no incluir** la descripción libre ni datos del ciudadano, igual que en el resto de nuestros eventos.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **Un campo en el evento de cambio de estado** | Sin registros nuevos. Llega a quienes ya leen ese evento. Sumar un recurso no cambia el contrato. | Un futuro consumidor tiene que filtrar por el campo. |
| Un tipo de evento por recurso (propuesta original) | Cada módulo se suscribe solo a lo suyo. Payload específico por recurso. | Tres tipos para ningún consumidor, y uno más por cada recurso nuevo. |
| Un evento genérico `RespuestaSolicitada` con un campo `recurso` | Un solo registro y un solo tópico. | Es un tipo nuevo que tampoco tiene consumidor. |
| Un evento por módulo destinatario (`LimpiezaRequerida`, `SanidadRequerida`) | Cada módulo recibe una orden directa. | Acopla Emergencias a quién responde: cada módulo nuevo obliga a cambiar nuestro contrato. Contradice la arquitectura orientada a eventos. |
| No publicar los recursos: cada módulo interpreta `EmergenciaCreada` | No hay nada nuevo. | La regla "qué recurso hace falta" se duplica en cada módulo, con riesgo de que no coincidan. |

## Consecuencias

Se hace más fácil:

- Mostrar la respuesta en la demo sin depender de que otro módulo la consuma.
- Sumar un recurso sin renegociar contratos ni registrar tipos.
- No ampliar la exposición de la ubicación exacta.

Se hace más difícil:

- Si aparece un consumidor por recurso, va a tener que filtrar el evento de cambio de estado o pedir un tipo propio. En ese caso se abre un ADR nuevo.
- El evento de cambio de estado gana un campo opcional que solo viene en `VALIDATED`.

Pendiente de definir:

- El nombre definitivo del evento de cambio de estado.
- Si `EmergenciaCreada` lleva coordenadas: Espacios las lee para marcar incidentes. Es una decisión aparte, con el visto bueno de Seguridad.

## Historial

- 2026-10-16: creación del ADR (estado: propuesta)
- 2026-11-02: se aplica el review del 2026-10-18. Al no haber consumidores, los eventos por recurso pasan a ser un campo del cambio de estado. Se saca el argumento de los permisos por tópico, que el gateway no ofrece, y queda barrio en vez de ubicación exacta. Cambia el título.
