# Guardar el barrio de la emergencia al crearla

- Estado: aceptada
- Fecha: 2026-10-16
- Decisores: @juanimoli (Producto), @Matiamistadi y @luccaperazzo (Back), @valenfiumana (Seguridad)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

La ubicación de una emergencia (latitud y longitud) se guarda cifrada. Por eso la base no puede calcular distancias ni filtrar por zona: cualquier operación geográfica requiere descifrar en memoria, emergencia por emergencia.

En el Hito 2 aparecen necesidades que trabajan por zona y no por punto exacto:

- [Elevar temporalmente la prioridad con contextos de zona](elevar-temporalmente-la-prioridad-con-contextos-de-zona.md) necesita saber, al leer, si una emergencia está en una zona afectada.
- Los recursos requeridos que publicamos (ver [Informar los recursos requeridos como un campo del cambio de estado](informar-los-recursos-requeridos-como-un-campo-del-cambio-de-estado.md)) le sirven a otros módulos con una referencia de zona.
- Reclamos sube la prioridad de sus reclamos abiertos por barrio cuando recibe `EmergenciaCreada`, y descarta el evento si no trae barrio. Su `barrio` es texto libre, no un catálogo.

Hoy la emergencia no guarda ninguna referencia de zona. Estaba previsto en B5.

## Decisión

- Vamos a guardar el **barrio** de cada emergencia **en claro**, en una columna propia, al momento de crearla. La ubicación exacta (latitud, longitud y dirección) sigue cifrada en reposo.
- Vamos a derivar el barrio en el backend a partir de la ubicación descifrada en memoria, con un cálculo de punto en polígono contra el **catálogo oficial de los 48 barrios de CABA** (datos abiertos). No usamos un catálogo de Reclamos, porque su barrio es texto libre. No vamos a sumar extensiones geográficas a la base.
- Vamos a que una emergencia fuera de los límites del catálogo quede con barrio vacío. Esa emergencia no participa de ninguna lógica por zona.
- Vamos a que, en los eventos externos, el barrio se derive por punto en polígono si hay coordenadas. Si no hay, solo se acepta el texto del evento cuando coincide con el catálogo.
- Vamos a que el barrio no se recalcule: es el de la ubicación al crear la emergencia.
- Vamos a **conservar el barrio después de la retención**. `LOCATION_RETENTION_HOURS` (72 horas desde `closed_at`) anonimiza únicamente la ubicación exacta. El barrio permanece con la fila para la lógica por zona, las respuestas predeterminadas y las métricas.
- Vamos a completar las **emergencias existentes** con un backfill si todavía tienen `location_encrypted`. Las que ya fueron anonimizadas quedan con barrio vacío.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **Barrio en claro, derivado al crear** | Consultas por zona con un `WHERE` simple. Es la clave de zona que ya leen otros módulos. No agrega infraestructura. | Expone en claro una ubicación aproximada. El barrio es una zona gruesa. |
| Descifrar la ubicación en cada consulta por zona | No guarda ningún dato de ubicación en claro. | Cada lectura del tablero descifra todas las emergencias abiertas. Costoso y difícil de paginar y ordenar. |
| Ubicación sin cifrar + extensión geográfica (PostGIS) | Radios y distancias exactas en la base. | Revierte la decisión de cifrar la ubicación. Suma una extensión a la base gestionada. |
| Pedirle el barrio al ciudadano | Trivial de guardar. | Agrega fricción en una emergencia y el dato no es confiable. No cubre el botón de pánico. |
| Geohash o grilla propia | Precisión configurable. | Nadie más en la ciudad usa esa clave: no se puede cruzar con Reclamos ni con los eventos de otros módulos. |

## Consecuencias

Se hace más fácil:

- Resolver por zona sin descifrar ubicaciones: contextos de zona, filtros del tablero y payloads hacia otros módulos.
- Cruzar datos con otros módulos que ya hablan en barrios.

Se hace más difícil:

- **Privacidad:** el barrio es una ubicación aproximada guardada en claro, y sobrevive a la retención de la ubicación exacta. Seguridad lo revisó y lo definió así el 2026-10-21 (@valenfiumana, en este PR).
- El catálogo de barrios pasa a ser una dependencia: si cambian los límites, hay que actualizarlo, y las emergencias viejas conservan el barrio que tenían.
- El barrio es grueso: dos emergencias en extremos opuestos de un barrio grande se tratan igual.
- Reclamos compara el barrio como texto, sin distinguir mayúsculas. Si el suyo no coincide con el nombre oficial, ese reclamo no sube de prioridad.

Pendiente de coordinar:

- Que Reclamos normalice su texto libre contra los 48 barrios oficiales.

## Historial

- 2026-10-16: creación del ADR (estado: propuesta)
- 2026-11-02: se aplica el review del 2026-10-18 con lo que definió Seguridad el 2026-10-21: catálogo oficial de 48 barrios, barrio en eventos externos, retención y emergencias existentes.
- 2026-11-02: cambio de estado: propuesta → aceptada
