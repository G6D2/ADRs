# Responder a las alertas climáticas de Espacios

- Estado: propuesta
- Fecha: 2026-11-16
- Decisores: @Salterm27 (reglas de negocio e implementación), @KevinAlajarin (redacción). Lo confirma Producto (@juanimoli).
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Espacios Públicos detecta riesgo climático con Open-Meteo. Un administrador lo confirma y publica `com.citypass.espacios.CriticidadClimaticaConfirmada` (schemaId 28). Cuando la alerta vence o se cierra a mano, publica `CriticidadClimaticaFinalizada` (schemaId 27) con el mismo `alertaId`. Los dos tipos ya están registrados en el catálogo del gateway.

| Campo de la confirmación | Contenido |
| --- | --- |
| `alertaId` | Clave de negocio de la alerta. Es la misma en la finalización |
| `nivelCriticidad` | `BAJA`, `MEDIA`, `ALTA` o `CRITICA` |
| `fenomeno` | `TORMENTA_FUERTE`, `LLUVIA_INTENSA`, `GRANIZO`, etc. |
| `ubicaciones` | `[{ latitud, longitud }]`: los puntos consultados a Open-Meteo. Espacios aclara que no son un polígono ni un radio de afectación |
| `barrios` | Barrios que contienen esos puntos |
| `vigenciaDesde`, `vigenciaHasta` | Ventana de la alerta, en epoch ms |

Espacios aclara que el evento informa un hecho y no ordena acciones: cada módulo decide su política por nivel. En su módulo, `ALTA` y `CRITICA` bloquean reservas al aire libre, y `BAJA` y `MEDIA` solo dejan la contingencia visible.

Hasta ahora, en Emergencias, la alerta climática estaba prevista pero no decidida:

- El ADR aceptado [Consumir eventos externos con un listener y reglas por evento](consumir-eventos-externos-con-un-listener-y-reglas-por-evento.md) nombra `CriticidadClimaticaConfirmada` entre los eventos a consumir.
- [Elevar temporalmente la prioridad con contextos de zona](elevar-temporalmente-la-prioridad-con-contextos-de-zona.md) la resolvía por barrio y pedía un aviso climático: un banner en el tablero y un evento `ContextoZonaActivado` al bus.
- [Elevar la prioridad en un radio alrededor del evento](elevar-la-prioridad-en-un-radio-alrededor-del-evento.md) reemplazó al anterior. Pasó la zona a un punto con radio y dejó abierta la pregunta: «¿Cómo entra una alerta climática?». También dejó sin implementar el aviso climático.

Además, Analítica tiene una métrica de emergencias por evento climático. Esa métrica cuenta emergencias con `causaTipo = EVENTO_CLIMATICO`, un valor que ya existe en el enum de `EmergenciaCreada` y que hoy nadie emite.

Este ADR cierra esas preguntas con las mismas piezas que ya usan los eventos de Espacios, sin agregar mecanismos nuevos:

- contextos de zona por radio con el vínculo medido al crear;
- coberturas `PREVENTIVA` con una sola emergencia por causa;
- listeners con reglas por evento.

## Decisión

Vamos a responder a la alerta climática con dos efectos, según el nivel: **subir la prioridad en los barrios de la alerta** y **programar una emergencia preventiva**.

### 1. Zona: un contexto por barrio, con el círculo que cubre el barrio

Cada barrio de `barrios` que está en el catálogo de CABA genera un contexto de zona `ALERTA_CLIMATICA`, con un punto y un radio que cubren el barrio entero:

- **Centro:** el promedio de los vértices del polígono del barrio (catálogo GCBA). Es aproximado: un borde con muchos vértices corre el centro hacia ese borde.
- **Radio:** la distancia del centro al vértice más lejano, más 100 m. El margen es la misma tolerancia con que se asigna el barrio a un punto que cae en un hueco del polígono. Va de ~930 m (Coghlan) a ~4,1 km (Recoleta).

El círculo se pasa a los barrios vecinos: el de Palermo alcanza pedazos de Belgrano, Colegiales, Recoleta y Villa Crespo. Por eso, **al vincular, la emergencia además tiene que ser de ese barrio**. Se decide al crear la emergencia, con la coordenada en memoria y el barrio que ya se le asigna (ver [Guardar el barrio de la emergencia al crearla](guardar-el-barrio-de-la-emergencia-al-crearla.md)): no se descifra nada al leer. Una emergencia sin barrio (fuera de CABA) no entra en estos contextos.

Si la alerta no trae ningún barrio del catálogo, **cada punto de `ubicaciones`** genera un contexto con su `latitud`, `longitud` y el radio del nivel (punto 2), sin mirar el barrio. Así una alerta sin barrios no se pierde.

En los dos casos funciona igual que un evento público:

- el vínculo se mide una sola vez, al crear la emergencia;
- la suba se calcula en SQL al leer;
- es de un nivel, con tope en la prioridad máxima;
- varios contextos sobre la misma emergencia no se acumulan;
- las `PREVENTIVA` no suben.

La referencia externa de cada contexto es `<alertaId>#<índice>`, con la posición del barrio en `barrios` o la del punto en `ubicaciones`. `alertaId` agrupa los contextos para darlos de baja juntos. Los barrios fuera del catálogo y los puntos sin coordenadas numéricas se descartan.

La ventana del contexto es `vigenciaDesde`–`vigenciaHasta`.

### 2. Niveles y radio

| Nivel | Contexto de zona | Radio sin barrios | Emergencia preventiva |
| --- | --- | --- | --- |
| `CRITICA` | Sí | 1000 m | Sí |
| `ALTA` | Sí | 500 m | Sí |
| `MEDIA` | No | — | No |
| `BAJA` | No | — | No |

Con barrios, el nivel solo decide si hay contextos: el radio es el que cubre cada barrio. Sin barrios, el radio depende del nivel, porque una alerta crítica afecta un área mayor que una alta. Cada contexto guarda el radio con el que se registró, así que el mecanismo de vínculo no cambia. Los dos radios por nivel son configurables por entorno (`CONTEXTO_ZONA_RADIO_CLIMA_CRITICA_METROS` y `CONTEXTO_ZONA_RADIO_CLIMA_ALTA_METROS`).

`MEDIA` y `BAJA` no generan nada, en línea con Espacios, donde esos niveles no son accionables.

### 3. Emergencia preventiva

Con `ALTA` o `CRITICA` se crea **una** emergencia `PREVENTIVA` por `alertaId`, con el mismo esquema que la cobertura preventiva de eventos:

- `causaTipo = EVENTO_CLIMATICO` y `causaId = alertaId`;
- la deduplica el índice único de causa de las `PREVENTIVA`;
- se crea sin ciudadano, en `PENDING` y con `source = EXTERNAL`;
- `EmergenciaCreada` sale por el outbox, en la misma transacción.

La ubicación es el primer punto válido de la alerta. Si no hay ninguno, queda para validación manual. La severidad sale del nivel: `ALTA` → `high` y `CRITICA` → `critical`. La descripción lleva el nivel, el fenómeno y los barrios.

Es el caso de negocio: ante una alerta alta o crítica, el dispatcher tiene en el tablero una tarea concreta de despliegue preventivo, y Analítica puede contar las emergencias por evento climático.

Una alerta que ya venció cuando llega no crea la emergencia. Pasa, por ejemplo, al releer el tópico.

### 4. Fin de la alerta

`CriticidadClimaticaFinalizada`:

- **Da de baja todos los contextos de la alerta.** Las emergencias vinculadas bajan solas, porque la suba se calcula al leer.
- **Cancela la emergencia preventiva si nadie la despachó** (`PENDING` o `VALIDATED`). Publica `EmergenciaActualizada`. Si ya avanzó, queda como está.
- **Gana aunque llegue antes que la confirmación.** Deja una marca de baja, y la confirmación tardía no crea contextos.

### 5. Aviso climático

No se implementa el evento `ContextoZonaActivado` ni un banner propio:

- La alerta ya está en el bus: la publica Espacios para todos los módulos, y ningún módulo pidió un evento nuestro de activación.
- El vínculo de cada emergencia con la alerta sale por `EmergenciaVinculadaAContextoZona`, con `contextoTipo = ALERTA_CLIMATICA`. Ese evento está definido en [Publicar el vínculo de una emergencia con un contexto de zona](publicar-el-vinculo-de-una-emergencia-con-un-contexto-de-zona.md).
- Para el dispatcher, el aviso es la emergencia `PREVENTIVA` en el tablero. El motivo de cada suba ya se ve en `priorityContext`, por ejemplo «Alerta climática CRITICA: tormenta fuerte».

### 6. Implementación

Sigue la estructura del ADR de listeners:

- Un paquete `src/listeners/alerta-climatica/` con un listener por evento.
- **Reglas de la confirmación:** `01-crear-emergencia-climatica` y `02-registrar-contextos-climaticos`.
- **Reglas de la finalización:** `01-cancelar-emergencia-climatica` y `02-dar-de-baja-contextos-climaticos`.
- Handlers registrados en el `OutcomeFactory`, en la transacción del evento.
- Deduplicación por `metadata.eventId`, como el resto de los listeners.

La base es el código del PR #117 de `emergencias-backend`. Respecto de ese PR cambian dos cosas: el radio pasa a depender del nivel, y `BAJA` y `MEDIA` dejan de registrar contextos.

La zona por barrio suma tres piezas al código del PR #122: el círculo de cada barrio se calcula una vez desde el catálogo (`zonaDelBarrio` en `src/domain/barrio.js`); cada contexto por barrio guarda `barrios = [su barrio]` (columna de V33, sin migración nueva); y `linkZoneContexts` exige el barrio cuando el contexto lo tiene. Los eventos comunitarios no tienen barrios y no cambian.

## Alternativas consideradas

| Alternativa | A favor | En contra |
| --- | --- | --- |
| Zona por barrio medida al leer (ADR anterior) | Alcanza también a las emergencias abiertas antes de la alerta | Vuelve al mecanismo que el ADR de radio abandonó. Haría falta otro camino de vínculo, al leer, solo para clima |
| Toda la ciudad para `CRITICA` | Una tormenta no respeta radios | Sube todas las emergencias a la vez, así que la prioridad deja de distinguir nada |
| Radio único de 2000 m para todos los niveles (PR #117 original) | Un solo valor | Una alerta `BAJA` subiría prioridades y Espacios no la considera accionable. No distingue alta de crítica |
| Solo contextos, sin emergencia preventiva | Menos cambios | El dispatcher no tiene una tarea de despliegue y Analítica no tiene emergencias por evento climático |
| Contexto por punto con radio por nivel (PR #122) | Un solo valor por nivel. No necesita el catálogo de barrios | La suba se limita a los alrededores de los puntos que consulta Espacios: una emergencia del mismo barrio a 2 km no sube |
| Círculo que cubre el barrio, sin filtrar por barrio | El mismo vínculo por distancia que un evento | El círculo se pasa a los vecinos: el de Palermo (~3,8 km) sube emergencias de Belgrano, Colegiales, Recoleta y Villa Crespo |
| **Círculo que cubre el barrio y filtro por barrio al vincular, con el punto como respaldo, y preventiva para `ALTA` y `CRITICA`** (elegida) | Sube en todo el barrio de la alerta y solo ahí. Reusa contextos, vínculo y coberturas; no descifra nada al leer. Sigue la política por nivel de Espacios | El vínculo de las alertas por barrio también depende del barrio de la emergencia |

## Consecuencias

Se hace más fácil:

- Que una alerta alta o crítica se vea en el tablero y en Analítica sin eventos ni tablas nuevas.
- Mantener una sola forma de zona (punto y radio) para todos los contextos.
- Que la suba alcance a todo el barrio de la alerta, y no solo a los alrededores de los puntos que consulta Espacios.

Se hace más difícil, y lo aceptamos:

- **La suba depende de los barrios que manda Espacios.** Son los barrios donde caen los puntos consultados (`WEATHER_LOCATIONS`), no la superficie de la tormenta. Una emergencia de otro barrio no sube, aunque esté bajo la misma tormenta.
- **El círculo de un barrio grande es grande** (Palermo ~3,8 km, Recoleta ~4,1 km). El filtro por barrio evita que alcance a los vecinos, pero el centro es aproximado (promedio de vértices).
- **Una emergencia sin barrio no sube por una alerta por barrio.** Pasa con las de fuera de CABA o sin ubicación.
- **Si la alerta no trae barrios del catálogo, vuelve a los puntos** con el radio del nivel: la suba se limita a sus alrededores.
- **Las emergencias abiertas antes de la alerta no suben.** Es la misma regla del ADR de radio.
- **Una sola preventiva por alerta, en el primer punto,** aunque la alerta tenga varios.
- **Los niveles son criterios técnicos de demostración de Espacios,** no alertas oficiales.

Neutras:

- `EmergenciaCreada` no cambia: `EVENTO_CLIMATICO` ya está en el enum de `causaTipo`.
- Este ADR responde la pregunta abierta del ADR de radio sobre la alerta climática y el aviso climático que había quedado sin implementar.

### Preguntas para Producto

- [ ] ¿Subir en todo el barrio de la alerta es lo que se espera, o alcanza con los alrededores de los puntos?
- [ ] ¿Los radios de 1000 m para `CRITICA` y 500 m para `ALTA` son los correctos para las alertas sin barrios?
- [ ] ¿`ALTA` y `CRITICA` crean una emergencia `PREVENTIVA`, o solo suben la prioridad?
- [ ] ¿Alcanza con la `PREVENTIVA` como aviso al dispatcher, o hace falta un banner aparte?

## Historial

- 2026-11-16: creación del ADR (estado: propuesta)
- 2026-11-17: la zona pasa a ser un contexto por barrio de la alerta, con el círculo que cubre el barrio y el filtro por barrio al vincular. Los puntos con radio por nivel quedan para las alertas sin barrios del catálogo. Se suman las dos alternativas evaluadas (punto con radio por nivel y círculo sin filtro).
