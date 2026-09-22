# Actualización de la Arquitectura Frontend: Next.js, App Router, BFF y Gestión de Estado

- Estado: propuesta
- Fecha: 2026-09-22
- Decisores: Equipo de Frontend
- Reemplaza a: usar-reactjs-para-el-frontend.md
- Reemplazada por: —

## Contexto

Este ADR actualiza la arquitectura y las tecnologías implementadas en el frontend respecto a las definiciones iniciales del proyecto.

Durante el desarrollo se evidenció que un esquema tradicional de SPA (React conectándose directamente al backend de Express) era insuficiente para cubrir los requerimientos técnicos y operativos de la aplicación:
- **Seguridad de credenciales:** Almacenar tokens JWT en el navegador (`localStorage` o memoria) exponía las sesiones ante ataques de XSS.
- **Arranque en frío (Cold Start de Render):** En el entorno de despliegue, el backend se duerme tras periodos de inactividad y tarda hasta 60 segundos en levantar; comunicarse directo desde el navegador arrojaba errores `502/503` inmediatos.
- **Riesgo de operaciones duplicadas:** Los reintentos a ciegas en llamadas de creación (`POST`) ante fallos de red podían generar alertas de emergencia duplicadas en la base de datos.
- **Límites de APIs externas:** Consultar OpenStreetMap (Nominatim) directamente desde el cliente violaba sus políticas de tasa de uso (1 req/s) y provocaba bloqueos de IP y errores de CORS.
- **Definición sobre el estado global:** Se evaluó si incorporar un gestor de estado global (como Redux o Zustand), pero resultaba innecesario y contraproducente dado que la mayoría de los datos son estado del servidor (*server state*).

Por estas razones, se formaliza la actualización arquitectónica hacia Next.js con App Router, la implementación del patrón BFF y la consolidación del stack tecnológico actual del frontend.

## Decisión

Vamos a actualizar la arquitectura del frontend adoptando **Next.js (App Router) con React 19**, implementando una capa **BFF (Backend For Frontend)** y prescindiendo de un gestor de estado global (como Redux o Zustand).

### 1. ¿Por qué elegimos un BFF (Backend For Frontend)?
En lugar de que el navegador hable directo con el backend Express o con servicios externos, **todas las peticiones pasan primero por endpoints de Next.js (`/api/*`)**. Elegimos este patrón por cinco razones concretas:

- **Seguridad total de sesiones (Cookies `httpOnly`):** El BFF recibe las credenciales, consulta al backend y guarda el token JWT en una cookie protegida (`httpOnly`, `sameSite: lax`, `secure`). **El código JavaScript del navegador jamás tiene acceso al token**, eliminando de raíz el riesgo de robo de credenciales por XSS.
- **Absorber el Cold Start de Render en silencio:** El BFF (`lib/backend-fetch.js`) intercepta los errores 502/503/504 y reintenta la conexión automáticamente hasta 6 veces (esperando 4 segundos entre intentos). El usuario no ve pantallas de error mientras el servidor despierta.
- **Idempotencia automática para evitar duplicados:** Antes de reintentar un `POST` o `PATCH`, el BFF genera un identificador único (`Idempotency-Key`). Si la primera petición llegó a crearse en el backend pero la respuesta se perdió, el backend reconoce la clave y no duplica la emergencia.
- **Control de servicios externos y CORS:** Para la geocodificación inversa (`/api/geocode`), el BFF centraliza las llamadas a Nominatim con una cola que respeta el límite de 1 req/s y una caché en memoria de 1 hora. El navegador no sufre bloqueos ni problemas de CORS.
- **Frontend más simple:** Las vistas cliente solo hacen `fetch` a rutas locales relativas (`/api/backend/...`), sin preocuparse por tokens, URLs del servidor ni lógica compleja de reintentos.

### 2. ¿Por qué NO usamos un State Manager Global (Redux / Zustand)?
Decidimos explícitamente no instalar Redux, Zustand ni herramientas similares por los siguientes motivos:

- **Casi todo es Estado de Servidor (*Server State*):** Las alertas, los estados y los oficiales no son datos que nacen en el cliente; vienen de la base de datos. Ponerlos en Redux genera una copia duplicada que exige sincronización manual constante y produce datos desactualizados (*stale state*).
- **Los roles no comparten datos en memoria:** El ciudadano navega en `/(citizen)` y el despachador en `/ops`. No existe estado compartido que justifique un almacén global en la raíz de la app.
- **El estado vive donde se usa:**
  - El despachador consulta emergencias cada 60 s y hace una fusión incremental en su propio componente (`[...nuevas, ...existentes]`) para resaltar casos nuevos con un pulso visual de 6 segundos.
  - El ciudadano consulta el estado de su alerta cada 5 s y pausa el ciclo si minimiza la pestaña.
  - Manejar estos ciclos dentro de cada pantalla con hooks nativos (`useState`, `useEffect`) es más limpio, evita fugas de memoria y no ensucia un store global.
- **La URL es la fuente de verdad para filtros:** Las búsquedas, filtros y ordenamientos de tablas se manejan con parámetros de URL y `useMemo`. Esto permite usar el botón "Atrás" del navegador y compartir links directos sin configurar nada en un store.
- **Avisos compartidos resueltos con React nativo:** Para lo único que se necesitaba compartir (avisar en la interfaz si el backend está tardando más de 3 segundos en responder), usamos `useSyncExternalStore` nativo de React con un emisor de apenas 30 líneas (`backendWaitingStore`), sin sumar dependencias.

### 3. Otras tecnologías adoptadas en el proyecto
- **Mantine UI v9:** Componentes complejos listos para usar (modales de asignación de oficiales y descarte, notificaciones flotantes tipo toast y barras de scroll independientes en el Kanban).
- **Tailwind CSS v4:** Utilidades de estilo rápidas basadas en tokens semánticos del sistema *"Civic Guardian"* configurados en `@theme` (colores con contraste accesible AA).
- **Leaflet:** Mapas interactivos livianos para ubicar incidentes sin costos de licencias propietarias.
- **Iconografía (Tabler Icons y Material Symbols):** Tabler Icons para las herramientas técnicas del despachador y Material Symbols para las acciones simples del ciudadano.
- **Vitest:** Suite de pruebas ultrarrápida nativa en ESM, separando tests unitarios de lógica pura y tests de integración para los endpoints del BFF.

## Consecuencias

### Positivas
- **Seguridad robusta:** Cero tokens expuestos en el navegador.
- **Experiencia tolerante a fallos:** El usuario no sufre los arranques en frío de Render ni ve caídas temporales.
- **Operaciones seguras:** Imposible duplicar emergencias en reintentos gracias a la idempotencia.
- **Código liviano y directo:** Sin boilerplate de reducers, acciones o selectores; bundle pequeño y carga rápida.

### Negativas
- **Salto extra de red:** Cada petición hace una escala en el servidor de Next.js antes de llegar a Express (costo en milisegundos imperceptible para la escala del proyecto).
- **Requiere servidor Node.js:** El frontend no se puede desplegar como sitio estático puro (S3/HTML plano) porque necesita ejecutar el BFF y el middleware.

### Neutras
- **Sincronización por Polling:** Actualmente las vistas consultan periódicamente al servidor (60 s en despacho, 5 s en ciudadano). A futuro, esto deberia cambiar.

## Historial

- 2026-09-22: creación del ADR (estado: propuesta)
