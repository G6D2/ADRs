# Notificar cambios de emergencias con Server-Sent Events

- Estado: propuesta
- Fecha: 2026-11-13
- Decisores: (a confirmar en la revisión del PR)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Hoy todas las vistas se enteran de los cambios de una emergencia por
**polling** contra `GET /api/emergencies` (y el detalle):

- Dispatcher (`/ops`): cada 60 s.
- Ciudadano (seguimiento de su alerta): cada 5 s, pausado con la pestaña oculta.
- Oficial (`/oficial`, MVP nuevo): cada 5 s.

El ADR [Adoptar Next.js con App Router y un BFF](adoptar-next-js-con-app-router-y-un-bff.md)
eligió polling para esta etapa y dejó anotado que **debería** evolucionar a
Server-Sent Events (SSE) o WebSockets. Con la vista del oficial esa deuda
se vuelve visible:

- **Latencia operativa.** Una emergencia asignada tarda hasta 60 s en aparecer
  en el tablero del dispatcher como "en camino" o "en lugar", y el oficial
  depende del intervalo para ver una asignación nueva. En un sistema de
  emergencias esos segundos son el problema, no un detalle.
- **Tráfico inútil.** Casi todas las consultas devuelven lo mismo que la
  anterior. Bajar el intervalo para ganar latencia multiplica la carga sobre
  el backend y la base.
- **Más consumidores.** Cada rol nuevo (oficial, y a futuro refuerzos u
  observaciones) suma su propio ciclo de polling con su propio intervalo.

Fuerzas y restricciones que condicionan la solución:

- **El navegador nunca habla directo con el backend.** Todo pasa por el BFF
  de Next.js, que guarda el JWT en una cookie `httpOnly` y lo reenvía como
  `Bearer`. Cualquier canal en tiempo real tiene que respetar eso.
- **Los Route Handlers de Next.js pueden devolver respuestas en streaming**
  (`ReadableStream`), pero **no hacen upgrade de la conexión a WebSocket**.
  Un WebSocket obliga a montar un servidor Node propio delante de Next o a
  que el navegador se conecte directo al backend con el token.
- **El flujo de datos es de una sola vía.** Las acciones de los usuarios
  (validar, asignar, en camino, resolver, cancelar) ya son `PATCH`
  validados por la máquina de estados. Lo que falta es que el servidor
  **avise**; no hay nada que el cliente necesite mandar por un canal
  persistente.
- **Autorización por rol y ownership.** Un ciudadano solo ve sus
  emergencias, un oficial solo las asignadas a él y el dispatcher todas
  (`canViewEmergency` y el filtro de `GET /api/emergencies`). El canal no
  puede abrir otra puerta con reglas distintas.
- **Hosting en Render (plan free).** Los servicios se duermen y se
  reinician; cada deploy corta las conexiones abiertas. Cualquier conexión
  persistente se va a caer y hay que reconectar sin perder cambios.
- **El backend ya contempla más de una instancia** (la detección de
  incidentes usa un advisory lock de Postgres por eso). Un aviso emitido en
  memoria de una instancia no llega a los clientes conectados a otra.
- **Los cambios de estado ya pasan por una sola transacción** que escribe la
  emergencia, el historial (`emergency_status_history`) y el outbox del bus.
  Es el punto natural para emitir el aviso.

## Decisión

Vamos a reemplazar el polling como mecanismo principal por **Server-Sent
Events (SSE)** usados como **señal de invalidación**, pasando por el BFF.

1. **Backend: endpoint de stream.** `GET /api/emergencies/stream` responde
   `text/event-stream`. Requiere `citizen`, `dispatcher` u `officer` (el
   mismo middleware que el listado). Al conectar envía un evento `ready`;
   después, un comentario de *heartbeat* cada ~25 s para que proxies y el
   edge de Render no corten la conexión por inactividad.

2. **Backend: origen de los avisos con `LISTEN/NOTIFY` de Postgres.** La
   transacción que cambia el estado (y la que crea la emergencia) ejecuta
   `pg_notify('emergencias_cambios', ...)` con `{ id, status, userId,
   assignedOfficerId }`. Postgres entrega el `NOTIFY` solo si la transacción
   hace commit, así que nunca se avisa un cambio que no quedó guardado. Cada
   instancia del backend mantiene una conexión `LISTEN` y reparte el aviso a
   sus clientes SSE. Así funciona con varias instancias sin sumar
   infraestructura.

3. **El evento no lleva datos de la emergencia.** Cada instancia filtra con
   las mismas reglas que el listado (dispatcher: todo; oficial:
   `assignedOfficerId === sub`; ciudadano: `userId === sub`) y al cliente le
   manda solo `event: emergency-changed` con `data: {"id": "..."}`. El
   cliente, al recibirlo, vuelve a pedir el listado o el detalle por los
   endpoints de siempre. La autorización y el recorte de campos siguen en un
   solo lugar.

4. **BFF: pasar el stream.** `app/api/backend/[...path]` reenvía
   `api/emergencies/stream` como respuesta en streaming, con el `Bearer` de la
   cookie, **sin** los reintentos de `lib/backend-fetch.js` (que son para
   requests cortos) y propagando el corte cuando el navegador se desconecta.

5. **Vencimiento del token.** El backend cierra el stream al llegar el `exp`
   del JWT. El cliente reconecta; si la reconexión da 401, pasa por
   `/api/auth/refresh` (el mismo flujo que `apiFetch`) antes de reintentar.

6. **Cliente: hook común.** Un hook (`useEmergencyStream`) usado por
   dispatcher, ciudadano y oficial abre un `EventSource` a
   `/api/backend/api/emergencies/stream`, recarga ante `emergency-changed` y
   **recarga todo al (re)conectar**, porque mientras estuvo desconectado
   pudo perder avisos. Con la pestaña oculta cierra la conexión y al volver
   reconecta y recarga.

7. **El polling queda como red de seguridad, no se elimina.** Con el stream
   conectado el intervalo pasa a ser largo (orden de 60–120 s); si el stream
   no está disponible se vuelve al intervalo actual. Un aviso perdido nunca
   deja una vista desactualizada para siempre.

Se implementa por etapas: primero backend (stream + `NOTIFY`) y la vista del
oficial, después dispatcher y ciudadano.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **SSE vía BFF + `LISTEN/NOTIFY`** (elegida) | HTTP común: pasa por el BFF con la misma cookie, sin exponer el token. `EventSource` reconecta solo. Sin infraestructura nueva. Funciona con varias instancias. La autorización sigue en los endpoints existentes. | Una conexión abierta por pestaña en BFF y backend. Hay que manejar el vencimiento del token y la reconexión. Solo servidor → cliente. |
| **WebSockets** | Bidireccional, estándar para tiempo real. | Los Route Handlers de Next no hacen upgrade: requiere un servidor Node propio delante de Next o conexión directa navegador → backend, lo que expone el JWT o exige otro esquema de auth (ticket de un solo uso, CORS, CSRF). La bidireccionalidad no se usa: las acciones ya son `PATCH`. |
| **Seguir con polling y bajar los intervalos** | Cero cambios de arquitectura. | La latencia mejora a costa de multiplicar consultas que casi siempre devuelven lo mismo. No escala con más roles. |
| **Long polling** | Funciona sobre HTTP sin streaming. | Más complejo que SSE para el mismo resultado; un request nuevo por cada aviso. |
| **Avisos desde un `EventEmitter` en memoria** | Lo más simple de implementar. | Solo funciona con una instancia del backend: un cambio hecho en otra instancia no llega. |
| **Consumir los eventos del bus (Kafka) para alimentar el stream** | Reutiliza los eventos de dominio que ya se publican. | El bus es *at-least-once*, pasa por el outbox con su demora de poll y por el gateway del Grupo 1. Suma latencia y dependencia externa para algo que es interno del módulo. |
| **Servicio administrado (Pusher, Ably, Supabase Realtime)** | Resuelve conexiones, reconexión y escala. | Dependencia y costo externos; los avisos (aunque mínimos) salen de nuestra infraestructura; hay que integrar su autenticación con nuestros roles. |

## Consecuencias

### Positivas

- Los cambios de estado llegan a las vistas en uno o dos segundos en lugar
  de hasta 60 s.
- Menos tráfico: las consultas se hacen cuando hubo un cambio, no por reloj.
- Un solo mecanismo para todos los roles, en lugar de un intervalo distinto
  por pantalla.
- La autorización y el recorte de campos no se duplican: el stream solo
  avisa y los datos salen de los endpoints existentes.
- No suma infraestructura: usa Postgres, que ya está.

### Negativas

- Cada pestaña abierta mantiene una conexión en el BFF y otra en el backend.
  Hay que vigilar el límite de conexiones y la memoria de las instancias.
- El backend necesita una conexión dedicada de Postgres por instancia para el
  `LISTEN`, fuera del pool, con reconexión propia.
- Más casos a probar: reconexión, vencimiento del token, pestaña oculta,
  deploy en medio de una sesión.
- Render corta las conexiones en cada deploy o reinicio; la experiencia
  depende de que la reconexión y la recarga funcionen bien.
- No está confirmado si, en el plan free de Render, una conexión SSE abierta
  evita que el servicio se duerma. No se cuenta con eso: el keep-warm actual
  sigue siendo necesario.

### Neutras

- El polling no desaparece; queda como respaldo con intervalo largo.
- `pg_notify` limita el payload a 8000 bytes; con solo ids y estado sobra.
- Si más adelante hiciera falta un canal bidireccional (por ejemplo, chat
  con el oficial), se reevalúa WebSockets en un ADR nuevo que reemplace a
  este.

## Historial

- 2026-11-13: creación del ADR (estado: propuesta)
