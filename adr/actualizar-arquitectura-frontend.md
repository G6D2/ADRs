# Adoptar Next.js con App Router y un BFF en el frontend

- Estado: propuesta
- Fecha: 2026-09-22
- Decisores: equipo, Producto
- Reemplaza a: [Usar ReactJS para el frontend](usar-reactjs-para-el-frontend.md)
- Reemplazada por: —

## Contexto

Este ADR actualiza la arquitectura y las tecnologías implementadas en el frontend respecto a las definiciones iniciales del proyecto.

Durante el desarrollo se evidenció que un esquema tradicional de SPA (React conectándose directamente al backend de Express) era insuficiente para cubrir los requerimientos técnicos y operativos de la aplicación:
- **Seguridad de credenciales:** Almacenar tokens JWT en el navegador (`localStorage` o memoria) exponía las sesiones ante ataques de XSS.
- **Arranque en frío (Cold Start de Render):** En el entorno de despliegue gratuito, el backend entra en suspensión tras periodos de inactividad y puede demorar entre 30 y 60 segundos en responder. Si el navegador llama directamente, arroja errores `502/503/504` inmediatos. El frontend necesita absorber este tiempo mediante reintentos controlados y estados de espera visuales sin degradar la experiencia de usuario.
- **Riesgo de operaciones duplicadas:** En llamadas de mutación (`POST`/`PATCH`), si la red se corta luego de que el backend procesó la solicitud o el proxy responde con un timeout `504`, los reintentos a ciegas pueden generar duplicados (por ejemplo, alertas de emergencia repetidas). El frontend debe contar con un mecanismo de identificación única de peticiones.
- **Límites de APIs externas y geocodificación:** Consultar servicios de geocodificación inversa directamente desde el cliente viola políticas de tasa de uso (como el límite estricto de 1 req/s de OpenStreetMap/Nominatim) y expone la aplicación a bloqueos por IP y errores de CORS.
- **Definición sobre el estado global:** Se evaluó si incorporar un gestor de estado global (como Redux o Zustand), pero resultaba innecesario y contraproducente dado que la mayoría de los datos son estado del servidor (*server state*).

Por estas razones, se formaliza la actualización arquitectónica hacia Next.js con App Router, la implementación del patrón BFF y la consolidación del stack tecnológico actual del frontend.

## Decisión

Vamos a actualizar la arquitectura del frontend adoptando **Next.js (App Router) con React 19**, implementando una capa **BFF (Backend For Frontend)** y prescindiendo de un gestor de estado global (como Redux o Zustand).

### 1. ¿Por qué elegimos un BFF (Backend For Frontend)?
En lugar de que el navegador hable directo con el backend Express o con servicios externos, **todas las peticiones pasan primero por endpoints de Next.js (`/api/*`)**. Elegimos este patrón por cinco razones concretas:

- **Seguridad total de sesiones (Cookies `httpOnly`):** El BFF recibe las credenciales, consulta al backend y guarda el token JWT en una cookie protegida (`httpOnly`, `sameSite: lax`, `secure`). **El código JavaScript del navegador jamás tiene acceso al token**, eliminando de raíz el riesgo de robo de credenciales por XSS.
- **Absorber el Cold Start de Render:** El BFF (`lib/backend-fetch.js`) intercepta los errores `502`, `503` y `504` y reintenta la conexión automáticamente hasta 6 veces con pausas de 4 segundos. Este ciclo de 24 segundos cubre la gran mayoría de los arranques en frío. Para los casos donde el servidor demora más en responder, el indicador de espera (`backendWaitingStore`, que detecta esperas mayores a 3 segundos) cubre el resto informando al usuario en pantalla de manera transparente y sin disparar pantallas de error.
- **Preparación para idempotencia (`Idempotency-Key`):** El BFF genera y propaga un identificador único en el encabezado `Idempotency-Key` para llamadas de mutación (`POST` y `PATCH`). Sin embargo, **el backend de Express todavía no honra este encabezado (deuda técnica registrada)**. Mientras tanto, el reintento automático solo se garantiza seguro para métodos de consulta (`GET`); para `POST`/`PATCH`, el riesgo de duplicación ante un `504` o corte de conexión luego de procesar la petición sigue existiendo hasta que el backend implemente el almacenamiento y chequeo de claves.
- **Control de servicios externos y geocodificación:** Para la geocodificación inversa (`/api/geocode`, `lib/geocode-server.js`), el BFF centraliza las llamadas utilizando **BigDataCloud como proveedor primario** y **Nominatim (OpenStreetMap) como respaldo**. Dispone de una caché en memoria de 24 horas (`CACHE_TTL_MS`) y una cola de peticiones a 1 req/s aplicada exclusivamente a Nominatim para respetar sus términos de uso. De este modo, el navegador no expone claves ni sufre bloqueos de IP o problemas de CORS.
- **Frontend más simple:** Las vistas cliente solo hacen `fetch` a rutas locales relativas (`/api/backend/...`), sin preocuparse por tokens, URLs del servidor ni lógica compleja de reintentos.

### 2. ¿Por qué NO usamos un State Manager Global (Redux / Zustand)?
Decidimos explícitamente no instalar Redux, Zustand ni herramientas similares por los siguientes motivos:

- **Casi todo es Estado de Servidor (*Server State*):** Las alertas, los estados y los oficiales no son datos que nacen en el cliente; vienen de la base de datos. Ponerlos en Redux genera una copia duplicada que exige sincronización manual constante y produce datos desactualizados (*stale state*).
- **Los roles no comparten datos en memoria:** El ciudadano navega en `/(citizen)` y el despachador en `/ops`. No existe estado compartido que justifique un almacén global en la raíz de la app.
- **El estado vive donde se usa:**
  - El despachador consulta emergencias cada 60 s y realiza una fusión incremental en su propio componente (`[...nuevas, ...existentes]`) para detectar y priorizar casos nuevos.
  - El ciudadano consulta el estado de su alerta cada 5 s y pausa el ciclo si minimiza u oculta la pestaña.
  - Manejar estos ciclos dentro de cada pantalla con hooks nativos (`useState`, `useEffect`) es más limpio, evita fugas de memoria y no ensucia un store global.
- **Filtros y búsquedas contenidos en el componente:** Las búsquedas, filtros y estados de selección (pills) viven directamente en el estado local de cada vista (`useState` y `useMemo`). No se requiere un store global para datos efímeros que solo interesan a la pantalla activa.
- **Avisos compartidos resueltos con React nativo:** Para lo único que se necesitaba compartir (avisar en la interfaz si el backend está tardando más de 3 segundos en responder), usamos `useSyncExternalStore` nativo de React con un emisor de apenas 30 líneas (`backendWaitingStore`), sin sumar dependencias.

### 3. Otras tecnologías adoptadas en el proyecto
- **Mantine UI v9:** Componentes complejos listos para usar (modales de asignación de oficiales y descarte, notificaciones flotantes tipo toast y barras de scroll independientes en el Kanban).
- **Tailwind CSS v4:** Utilidades de estilo rápidas basadas en tokens semánticos del sistema *"Civic Guardian"* configurados en `@theme` (colores con contraste accesible AA).
- **Leaflet:** Mapas interactivos livianos para ubicar incidentes sin costos de licencias propietarias.
- **Iconografía (Tabler Icons y Material Symbols):** Tabler Icons para las herramientas técnicas del despachador y Material Symbols para las acciones simples del ciudadano.
- **Vitest:** Suite de pruebas ultrarrápida nativa en ESM, separando tests unitarios de lógica pura y tests de integración para los endpoints del BFF.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **Next.js (App Router) + BFF** (Elegida) | Centraliza autenticación con cookies `httpOnly`, absorbe el cold start mediante reintentos en servidor y feedback visual, orquesta geocodificación con caché y cola, y simplifica las llamadas cliente. | Requiere entorno de ejecución Node.js (no estático) y añade un salto de red adicional. |
| **SPA tradicional (Vite / CRA directo a Express)** | Despliegue estático simple y sin intermediarios. | **Descartada:** Obligaría a almacenar tokens JWT en el navegador (`localStorage`/memoria) aumentando el riesgo de XSS; no puede absorber el arranque en frío de Render en servidor (trasladaría errores 502/503 al usuario); y expondría al cliente a bloqueos de CORS y rate limit en geocodificación. |
| **Remix (React Router 7)** | Excelente arquitectura de SSR y manejo de loaders/actions para interactuar con APIs. | **Descartada:** No ofrece ventajas diferenciales sobre Next.js para este alcance y la documentación y soporte del stack elegido (Mantine v9 y Tailwind v4) contaban con mayor madurez y adopción en Next.js. |
| **Gestor global (Redux Toolkit / Zustand)** | Estado predecible y accesible desde cualquier componente de la aplicación. | **Descartada:** Complejidad y boilerplate innecesarios. La enorme mayoría de los datos es *server state* que no se comparte entre roles (despacho y ciudadano no interactúan en memoria). |
| **TanStack Query (React Query)** | Estándar de la industria para gestión, caché y deduplicación de *server state*. | **Descartada:** Aunque resuelve elegantemente el estado de servidor, para la escala y frecuencia de este MVP un polling simple y controlado con hooks nativos (`useEffect`, `useState`, `useSyncExternalStore`) es suficiente y evita sumar dependencias o capas de abstracción adicionales al bundle. |

## Consecuencias

### Positivas
- **Seguridad robusta:** Cero tokens expuestos en el navegador gracias a cookies `httpOnly`.
- **Experiencia tolerante a fallos:** El usuario no sufre los arranques en frío de Render ni ve caídas temporales gracias al retry en BFF y al store de espera visual.
- **Preparación de contratos de red:** Cliente y BFF ya cuentan con encabezados estandarizados (`Idempotency-Key`) listos para cuando el backend active su validación.
- **Código liviano y directo:** Sin boilerplate de reducers, acciones o selectores; bundle pequeño y carga rápida.

### Negativas
- **Salto extra de red:** Cada petición hace una escala en el servidor de Next.js antes de llegar a Express (costo en milisegundos imperceptible para la escala del proyecto).
- **Requiere servidor Node.js:** El frontend no se puede desplegar como sitio estático puro (S3/HTML plano) porque necesita ejecutar el BFF y el middleware.
- **Riesgo residual de duplicación en mutaciones (deuda en backend):** Dado que el backend aún no procesa `Idempotency-Key`, reintentar `POST` o `PATCH` ante timeouts (`504`) o desconexiones post-procesamiento puede derivar en duplicados en la base de datos hasta saldar la deuda técnica.

### Neutras
- **Sincronización por polling:** Actualmente las vistas consultan periódicamente al servidor (cada 60 s en despacho de `/ops` y cada 5 s en seguimiento ciudadano). Se optó por polling en esta etapa porque la infraestructura gratuita en Render suspende el backend ante inactividad, lo que cortaría y complicaría la reconexión de enlaces persistentes. A futuro, **debería** evolucionar hacia Server-Sent Events (SSE) o WebSockets para contar con actualización en tiempo real push y reducir el tráfico innecesario una vez que la infraestructura cuente con instancias activas continuas.

## Historial

- 2026-09-22: creación del ADR (estado: propuesta)
