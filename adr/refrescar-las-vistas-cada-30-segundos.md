# Refrescar las vistas cada 30 segundos

- Estado: propuesta
- Fecha: 2026-11-16
- Decisores: @Salterm27 (la propuso y la implementó en el front) y Producto (@juanimoli), que la revisó.
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Las vistas se mantienen al día por **polling**: cada cierto tiempo el
navegador pide los datos al BFF de Next.js, y el BFF se los pide al
backend. No hay push (ni WebSocket ni SSE). La decisión de usar polling
está en
[Adoptar Next.js con App Router y un BFF](adoptar-next-js-con-app-router-y-un-bff.md),
que menciona 60 s para `/ops` y 5 s para el ciudadano. Esos valores ya no
eran los del código; hasta este cambio eran:

| Vista | Intervalo | Pedidos por minuto y pestaña |
|---|---|---|
| Dispatcher (`/ops`, lista y mapa) | 15 s (`POLLING_INTERVAL_MS`) | 4 |
| Ciudadano (`/mis-alertas`, detalle) | 5 s (`CITIZEN_POLLING_INTERVAL_MS`) | 12 |
| Oficial (`/oficial`) | 5 s (`CITIZEN_POLLING_INTERVAL_MS`) | 12 |

Cada refresco es un solo pedido (el tablero pide todos los estados
abiertos juntos). El polling del dispatcher se pausa con la pestaña
oculta y no superpone pedidos (`createPoller`).

Fuerzas en juego:

1. **Tráfico del BFF al backend.** Todos los pedidos salen del servidor
   del front hacia la URL pública del backend en `onrender.com`, que pasa
   por Cloudflare. El 15/11/2026 Cloudflare empezó a responder ese tramo
   con un desafío anti-bot ("Just a moment...", HTTP 429), y el tablero
   quedó sin datos. Menos pedidos bajan la exposición a ese bloqueo,
   aunque no lo resuelven.
2. **Plan free de Render:** recursos limitados en ambos servicios.
3. **Frescura de los datos.** En un sistema de emergencias, cuánto tarda
   el dispatcher en ver un pánico nuevo, o el ciudadano en ver que lo
   despacharon, importa.

## Decisión

Vamos a refrescar **todas las vistas cada 30 segundos** (2 pedidos por
minuto y por pestaña): dispatcher, ciudadano y oficial.

- `POLLING_INTERVAL_MS` y `CITIZEN_POLLING_INTERVAL_MS` pasan a `30_000`
  (`app/lib/api.js` en G6D2/Frontend).
- Se mantienen la pausa con la pestaña oculta y la regla de no superponer
  pedidos.
- **No cambia** el refresco de la sesión (canje del refresh token del
  Login Federado): no es un refresco de datos y depende de la vida del
  token.
- Las acciones del usuario (validar, asignar, cancelar, avanzar estado)
  siguen recargando los datos en el momento, sin esperar al próximo ciclo.
- Es la medida para el Hito 2. Si se acepta
  [Notificar cambios de emergencias con Server-Sent Events](notificar-cambios-de-emergencias-con-server-sent-events.md),
  este polling pasa a ser el respaldo del stream.

## Alternativas consideradas

| Alternativa | Decisión | Motivo |
|---|---|---|
| 30 s en todas las vistas (elegida) | **Se hace** | Un solo valor, fácil de razonar; baja el tráfico a la mitad en el dispatcher y a la sexta parte en ciudadano y oficial. |
| Dejar 15 s / 5 s | Descartada | Es el volumen que estaba en juego cuando apareció el bloqueo de Cloudflare. |
| Intervalos distintos por vista (p. ej. dispatcher 30 s, ciudadano y oficial 10 s) | Descartada por ahora | Le daría más frescura a quien espera una ayuda, pero con más tráfico. Se puede revisar si 30 s resulta lento en uso real. |
| 60 s, como dice el ADR de Next.js | Descartada | Un minuto sin ver un pánico nuevo es demasiado para el dispatcher. |
| Push con SSE o WebSockets | Descartada por ahora | El ADR de Next.js ya la deja para cuando haya instancias que no se duerman; en el plan free de Render el backend se suspende y corta las conexiones. Es la solución de fondo. |
| Pedir los datos desde el navegador directo al backend | Descartada | Saltaría el bloqueo de Cloudflare, pero rompe el diseño del BFF (token en cookie `httpOnly`). |
| Backoff ante 429 (espaciar los reintentos si el backend o Cloudflare rechazan) | Complementaria, no se hace ahora | Ayudaría a no insistir contra un bloqueo; se puede sumar aparte. |

Fuera de este ADR: llevar el tráfico del BFF al backend por la red privada
de Render, que es lo que de verdad saca a Cloudflare del medio.

## Consecuencias

### Esperadas

Se hace más fácil:

- Menos carga sobre el BFF, el backend y la base: de 4 a 2 pedidos por
  minuto por pestaña de dispatcher, y de 12 a 2 por pestaña de ciudadano
  u oficial.
- Menos exposición al desafío anti-bot de Cloudflare en el tramo BFF →
  backend.
- Un único intervalo para documentar y probar.

Se hace más difícil o cuesta:

- **Datos hasta 30 s viejos** en todas las vistas:
  - el dispatcher puede tardar hasta 30 s en ver una emergencia nueva
    (antes 15 s);
  - el ciudadano puede tardar hasta 30 s en ver que validaron o
    despacharon su alerta (antes 5 s);
  - el oficial puede tardar hasta 30 s en ver una asignación nueva (antes
    5 s).
- La sensación de "tiempo real" en el seguimiento del ciudadano baja
  bastante.

Neutras:

- El refresco de la sesión y los tiempos de los tokens no cambian.
- El backend no cambia.

### No esperadas (riesgos a vigilar)

- **No alcanza para el 429:** el desafío de Cloudflare depende de la IP
  del BFF y no solo del volumen. Si sigue apareciendo con 30 s, el arreglo
  es la red privada de Render, no un intervalo más largo.
- **Pedidos simultáneos:** todas las pestañas que se abren a la vez
  refrescan en sincronía cada 30 s. Con pocos usuarios no importa; con
  muchos, convendría agregar un desfase aleatorio.
- **Backend dormido:** el plan free suspende el backend tras 15 minutos
  sin tráfico. Con una pestaña abierta, 30 s lo sigue manteniendo
  despierto; sin pestañas abiertas, el primer pedido vuelve a pagar el
  arranque en frío, como antes.
- **El ciudadano reintenta a mano:** sin ver cambios en 5 s, puede
  recargar la página o volver a tocar el botón de pánico. El botón ya
  tiene límite (5 por minuto), pero conviene mostrar "actualizado hace
  N s" en el seguimiento.
- **El ADR de Next.js queda desactualizado** en los valores de polling
  (60 s / 5 s). Es un ADR aceptado y no se edita; este ADR deja registro
  de los valores vigentes.

## Relacionados

- [Adoptar Next.js con App Router y un BFF](adoptar-next-js-con-app-router-y-un-bff.md)
- [Elegir hosting cloud](elegir-hosting-cloud.md)
- [Notificar cambios de emergencias con Server-Sent Events](notificar-cambios-de-emergencias-con-server-sent-events.md)

## Historial

- 2026-11-16: creación del ADR (estado: propuesta)
- 2026-11-16: revisión de Producto. Se enlaza con el ADR de Server-Sent
  Events. Sigue en propuesta.
