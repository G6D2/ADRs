# Guardar el barrio de la emergencia al crearla

- Estado: propuesta
- Fecha: 2026-10-16
- Decisores: @juanimoli (Producto), @Matiamistadi y @luccaperazzo (Back), @valenfiumana (Seguridad)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

La ubicación de una emergencia (latitud y longitud) se guarda cifrada. Por eso la base no puede calcular distancias ni filtrar por zona: cualquier operación geográfica requiere descifrar en memoria, emergencia por emergencia.

En el Hito 2 aparecen necesidades que trabajan por zona y no por punto exacto:

- [Elevar temporalmente la prioridad con contextos de zona](elevar-temporalmente-la-prioridad-con-contextos-de-zona.md) necesita saber, al leer, si una emergencia está en una zona afectada.
- Las respuestas predeterminadas que publicamos (ver [Publicar las respuestas requeridas como un evento por tipo de recurso](publicar-las-respuestas-requeridas-como-un-evento-por-tipo-de-recurso.md)) le sirven a otros módulos con una referencia de zona.
- Reclamos ya trabaja por barrio, y es la clave de zona que comparten los módulos de la ciudad.

Hoy la emergencia no guarda ninguna referencia de zona. Estaba previsto en B5.

## Decisión

- Vamos a guardar el **barrio** de cada emergencia **en claro**, en una columna propia, al momento de crearla.
- Vamos a derivar el barrio en el backend a partir de la ubicación descifrada en memoria, contra el **mismo catálogo de barrios que usa Reclamos**, con un cálculo de punto en polígono. No vamos a sumar extensiones geográficas a la base.
- Vamos a que una emergencia fuera de los límites del catálogo quede con barrio vacío. Esa emergencia no participa de ninguna lógica por zona.
- Vamos a que las emergencias originadas por un evento externo reciban el barrio desde su mapper (ver [Consumir eventos externos con un listener y mappers por origen](consumir-eventos-externos-con-un-listener-y-mappers-por-origen.md)): lo toma del evento si viene, o lo deriva igual que en el flujo interno.
- Vamos a que el barrio no se recalcule: es el de la ubicación al crear la emergencia.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **Barrio en claro, derivado al crear** | Consultas por zona con un `WHERE` simple. Clave compartida con Reclamos. No agrega infraestructura. | Expone en claro una ubicación aproximada. El barrio es una zona gruesa. |
| Descifrar la ubicación en cada consulta por zona | No guarda ningún dato de ubicación en claro. | Cada lectura del tablero descifra todas las emergencias abiertas. Costoso y difícil de paginar y ordenar. |
| Ubicación sin cifrar + extensión geográfica (PostGIS) | Radios y distancias exactas en la base. | Revierte la decisión de cifrar la ubicación. Suma una extensión a la base gestionada. |
| Pedirle el barrio al ciudadano | Trivial de guardar. | Agrega fricción en una emergencia y el dato no es confiable. No cubre el botón de pánico. |
| Geohash o grilla propia | Precisión configurable. | Nadie más en la ciudad usa esa clave: no se puede cruzar con Reclamos ni con los eventos de otros módulos. |

## Consecuencias

Se hace más fácil:

- Resolver por zona sin descifrar ubicaciones: contextos de zona, filtros del tablero y payloads hacia otros módulos.
- Cruzar datos con otros módulos que ya hablan en barrios.

Se hace más difícil:

- **Privacidad:** el barrio es una ubicación aproximada guardada en claro. Requiere revisión de Seguridad antes de implementarse.
- El catálogo de barrios pasa a ser una dependencia: si cambian los límites, hay que actualizarlo, y las emergencias viejas conservan el barrio que tenían.
- El barrio es grueso: dos emergencias en extremos opuestos de un barrio grande se tratan igual.

Pendiente de definir:

- Si el barrio sigue la misma política de retención que la ubicación o se conserva como dato agregado (con Seguridad).
- Fuente y formato del catálogo de barrios que usa Reclamos.

## Historial

- 2026-10-16: creación del ADR (estado: propuesta)
