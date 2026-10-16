# Publicar las respuestas requeridas como un evento por tipo de recurso

- Estado: propuesta
- Fecha: 2026-10-16
- Decisores: @juanimoli (Producto), @Matiamistadi y @luccaperazzo (Back), Grupo 1 (EDA)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Cuando ocurre una emergencia, otros módulos de la ciudad tienen que actuar. Por ejemplo, ante un incendio se despacha a Bomberos, y después responden Limpieza y Sanidad. Emergencias no coordina a esos módulos: decide **qué tipo de recurso hace falta** (Bomberos, Policía o Ambulancia) y lo comunica por el bus. Cada módulo reacciona a lo que le interesa.

[Usar Kafka como bus de eventos](usar-kafka-como-bus-de-eventos.md) fija las reglas del gateway del Grupo 1:

- Tópicos `com.citypass.<namespace>.<EventType>`.
- Nombres en PascalCase y tiempo pasado.
- Registro previo de cada tipo de evento.
- El productor envía solo `data`.

Este ADR define el **contrato** de esos eventos, que es lo que leen los demás equipos. Cómo los generamos internamente se decide en [Generar respuestas predeterminadas con reglas y templates por recurso](generar-respuestas-predeterminadas-con-reglas-y-templates-por-recurso.md).

## Decisión

- Vamos a publicar **un tipo de evento por tipo de recurso**, con nombres tentativos `BomberosSolicitados`, `PoliciaSolicitada` y `AmbulanciaSolicitada`. Los nombres definitivos se acuerdan con el Grupo 1 antes del registro.
- Vamos a que el catálogo de recursos sea **cerrado y nuestro**. Sumar un recurso es una decisión de Producto, que se registra como un tipo de evento nuevo.
- Vamos a publicar estos eventos tanto para emergencias creadas en nuestra plataforma como para las originadas por eventos de otros módulos.
- Vamos a que `data` lleve, como mínimo:
  - `emergencyId` y `correlationId`;
  - `causationId`: el `eventId` del evento externo que lo originó, o vacío si se originó en la plataforma;
  - el origen del disparador;
  - el tipo de emergencia y la prioridad;
  - el barrio (ver [Guardar el barrio de la emergencia al crearla](guardar-el-barrio-de-la-emergencia-al-crearla.md));
  - la ubicación necesaria para despachar el recurso;
  - la fecha y hora de la solicitud.
- Vamos a **no incluir** en `data` la descripción libre ni datos del ciudadano, igual que en el resto de nuestros eventos.
- Vamos a que una emergencia que requiere varios recursos publique un evento por recurso, todos con el mismo `correlationId`.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **Un tipo de evento por recurso** | Cada módulo se suscribe solo al tópico que le corresponde, y los permisos del gateway se pueden dar por tópico. Payload específico por recurso. | Tres registros ante el gateway, uno más por cada recurso nuevo. |
| Un evento genérico `RespuestaSolicitada` con un campo `recurso` | Un solo registro y un solo tópico. | Cada consumidor recibe todo y filtra. No se puede restringir el acceso por recurso. El payload común termina siendo la unión de todos. |
| Un evento por módulo destinatario (`LimpiezaRequerida`, `SanidadRequerida`) | Cada módulo recibe una orden directa. | Acopla Emergencias a quién responde: cada módulo nuevo obliga a cambiar nuestro contrato. Contradice la arquitectura orientada a eventos. |
| No publicar respuestas: cada módulo interpreta `EmergenciaCreada` | No hay eventos nuevos. | La regla "qué recurso hace falta" se duplica en cada módulo, con riesgo de que no coincidan. Tampoco cubre las emergencias originadas afuera. |

## Consecuencias

Se hace más fácil:

- Que cada módulo consuma solo lo que le corresponde, sin conocer nuestras reglas.
- Sumar un módulo que reacciona a un recurso existente, sin cambios en Emergencias.
- Trazar la cadena completa: emergencia → recurso solicitado → respuestas de otros módulos.

Se hace más difícil:

- Dependemos del Grupo 1 para registrar cada tipo de evento antes de publicarlo en producción.
- Publicar la ubicación hacia otros módulos amplía su exposición. Requiere el visto bueno de Seguridad sobre qué precisión enviar.
- Cambiar el payload implica renegociarlo con todos los consumidores.

Pendiente de definir:

- Nombres definitivos y registro ante el Grupo 1.
- Qué módulos consumen cada evento y qué campos necesitan.
- Precisión de la ubicación publicada.

## Historial

- 2026-10-16: creación del ADR (estado: propuesta)
