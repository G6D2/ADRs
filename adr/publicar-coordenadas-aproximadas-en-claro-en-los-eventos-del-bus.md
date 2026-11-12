# Publicar coordenadas aproximadas en claro en los eventos del bus

- Estado: propuesta
- Fecha: 2026-11-12
- Decisores: @juanimoli (Producto), @valenfiumana (Security), @KevinAlajarin (DevOps)
- Reemplaza a: —
- Reemplazada por: —

> **Pendiente de verificación de Security (@valenfiumana).** Este ADR
> revierte la decisión de Security del 2026-11-02. No pasa a `aceptada`
> hasta que Valentina confirme la precisión (3 decimales en general y si
> `PANICO` baja a 2) y el tratamiento de la retención en el bus.

## Contexto

`EmergenciaCreada` y `EmergenciaActualizada` viajan por el bus EDA a todos
los módulos suscriptos. La ubicación de una emergencia es PII: en un botón
de pánico, el punto exacto puede ser el domicilio del ciudadano.

El 2026-11-02 Security decidió no publicar coordenadas en claro en el bus.
Hasta ahora la ubicación sale:

- en `data.barrio`, en claro (ver
  [Guardar el barrio de la emergencia al crearla](guardar-el-barrio-de-la-emergencia-al-crearla.md));
- en `data.ubicacion.latitud/longitud`, exacta y **cifrada** con
  AES-256-GCM con una clave propia del bus (`BUS_LOCATION_ENCRYPTION_KEY`,
  distinta de la de la base, PR #82).

Esa decisión no quedó registrada como ADR, y en la práctica ningún
consumidor tiene la clave del bus ni descifra. Revisamos los cinco módulos
que nos consumen el 2026-11-12:

| Módulo | Qué pasa hoy |
|---|---|
| Movilidad | Lee `latitud`/`longitud` numéricas en la raíz; como no están, descarta todas nuestras `EmergenciaCreada`. |
| Residuos | No puede bloquear zonas ni desviar rutas. Trabaja con un radio de 300 m y confirmó que el barrio no le sirve. La prueba integrada con ellos está frenada por esto. |
| Analítica | Guarda la ubicación en `null`. |
| Reclamos | Escala por barrio, pero no registra el incidente vial. |
| Espacios | Cae a barrio: afecta toda la zona en vez de un radio de 500 m. |

Fuerzas en juego:

- **Privacidad:** minimizar la PII que sale del módulo. El bus tiene N
  consumidores autenticados, pero no todos necesitan el punto exacto.
- **Retención:** nuestra base anonimiza la ubicación a las 72 h
  (`LOCATION_RETENTION_HOURS`), pero un evento publicado queda en el bus
  según la retención del Grupo 1, que no controlamos.
- **Integración:** el caso de uso CU-INT-01 requiere que otros módulos
  reaccionen a la ubicación de la emergencia. El freeze es el 2026-11-18.
- **Catálogo:** los schemas registrados de `EmergenciaCreada` y
  `EmergenciaActualizada` ya tienen `latitud` y `longitud` opcionales
  (`["null","double"]`) en la raíz de `data`, así que no hace falta
  cambiar el catálogo.

## Decisión

- Vamos a publicar `data.latitud` y `data.longitud` **en claro, como
  número, redondeadas a 3 decimales** (~100 m) en `EmergenciaCreada` y
  `EmergenciaActualizada` (PR #92 del backend).
- **[A definir con Security]** Para `categoria = PANICO` vamos a redondear
  a **2 decimales** (~1 km), porque el punto suele ser el domicilio o la
  posición de la persona. El resto de las categorías son hechos en la vía
  pública (incendio, infraestructura, disturbio, tránsito, preventiva).
- Vamos a publicar las dos coordenadas o ninguna: si la emergencia no
  tiene ubicación, o ya fue anonimizada por retención, van en `null`.
- La coordenada exacta sigue viajando cifrada en `data.ubicacion`, sin
  cambios, para quien en el futuro tenga la clave del bus.
- Quien necesite el punto exacto lo pide por HTTP autenticado
  (`GET /api/emergencies/{id}`), como establecía la decisión del
  2026-11-02.
- Nunca publicamos `address`, `description` ni datos del ciudadano.
- Esta decisión aplica a cualquier evento nuevo de Emergencias que lleve
  ubicación.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **Coordenadas redondeadas en claro** (elegida) | Funciona con lo que los consumidores ya leen. Sin cambios en el catálogo. 100 m alcanza para radios de 300–500 m. | Sale del módulo una ubicación aproximada que no controlamos después. |
| Solo cifradas (lo de hoy) | Minimiza PII. | Nadie descifra: la integración no funciona para ningún módulo. |
| Compartir la clave del bus con los consumidores | Punto exacto, sin cambiar el contrato. | Distribuir una clave simétrica a cinco equipos equivale a publicar en claro, y además agrega gestión de claves. |
| Solo barrio | Ya está en claro. | No sirve para radios: Residuos y Movilidad lo confirmaron. |
| Consulta HTTP del punto exacto en cada evento | Control de acceso por consumidor. | Cada módulo tiene que integrar nuestra API; acoplamiento síncrono; no llega antes del freeze. |

## Consecuencias

Se hace más fácil:

- Que Movilidad, Residuos, Espacios, Reclamos y Analítica reaccionen a
  la ubicación sin descifrar nada.
- Cerrar la prueba integrada del CU-INT-01 antes del freeze.

Se hace más difícil:

- Controlar la ubicación una vez publicada: queda en el bus y en las
  bases de los consumidores más allá de nuestras 72 h de retención.
- Volver atrás: si después se decide no publicarla, los consumidores ya
  dependen del campo.

Neutras:

- La precisión queda en una constante del backend (`COORD_DECIMALS`);
  cambiarla no rompe el contrato.
- Hay que avisar a los consumidores qué precisión esperar y que el campo
  puede venir en `null`.

## Historial

- 2026-11-02: Security decide no publicar coordenadas en claro en el bus
  (decisión registrada fuera del repo de ADRs).
- 2026-11-12: Producto decide publicar coordenadas aproximadas (PR #92 del
  backend). Creación del ADR (estado: propuesta), pendiente de la revisión
  de Security sobre la precisión para `PANICO`.
