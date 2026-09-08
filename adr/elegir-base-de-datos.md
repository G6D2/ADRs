# Elegir base de datos

- Estado: propuesta
- Fecha: 2026-09-07
- Decisores: @valenfiumana, @juanimoli, @Salterm27 (Producto), @Matiamistadi (Back)
- Reemplaza a: —
- Reemplazada por: —

Propuesto en el issue [#9](https://github.com/G6D2/ADRs/issues/9).

## Contexto

El backend de Emergencias necesita persistencia relacional para el MVP:
emergencias, su ciclo de vida de estados y las entidades mínimas asociadas.
El ADR [Elegir hosting cloud](elegir-hosting-cloud.md) define dónde corre el
backend (Render), pero no dónde vive la base de datos.

Fuerzas en juego:

- Costo cero hasta el Go Live.
- El equipo es junior y tiene dos sprints para el MVP: la operación de la base
  tiene que ser prácticamente nula.
- El trabajo es intermitente (semanas de parciales, fines de semana largos).
  Una base que se apaga sola por inactividad y requiere reactivación manual es
  un riesgo concreto para las demos.
- En la segunda entrega hay que integrarse con los productos de los otros
  grupos, y la identidad de usuarios la provee el componente de autenticación
  de la plataforma (Grupo 2). Cualquier feature propietaria que adoptemos ahora
  es acoplamiento a desarmar después.
- Los archivos adjuntos de una emergencia (foto, audio) se van a alojar fuera
  de la base, en un object storage todavía por definir. Que el proveedor de
  base de datos incluya storage **no** es un criterio de decisión.

Al momento de proponer esta decisión existía un proyecto de Supabase creado
por el equipo, sin esquema ni datos productivos. Desde entonces el backend
adoptó Neon en la práctica: el `README`, `db/README.md` y `.env.example` de
`emergencias-backend` documentan Neon como opción recomendada para el equipo y
para el secret `DATABASE_URL` del pipeline de CD.

## Decisión

- Vamos a usar **Neon** (PostgreSQL serverless) como base de datos para la
  primera entrega.
- Vamos a usarlo **solo como PostgreSQL administrado**: driver estándar
  (`pg`), sin apoyarnos en features propietarias del proveedor.
- Vamos a versionar las **migraciones en el repositorio** del backend
  (`db/migrations`, Flyway) y aplicarlas desde el pipeline de CD. No se
  modifica el esquema desde la consola web.
- Vamos a configurar la conexión por la variable de entorno `DATABASE_URL`,
  usando el endpoint *pooled* con SSL. La credencial nunca se commitea.
- Vamos a desarrollar localmente contra un **PostgreSQL en contenedor**
  (`docker-compose` del repo) aplicando las mismas migraciones. Nadie
  desarrolla contra la base compartida.
- Vamos a guardar en la base **solo metadatos y una URL** de los adjuntos,
  nunca los archivos.
- Vamos a alinear `render.yaml` con esta decisión: `DATABASE_URL` se define
  como secret del servicio y se elimina la base `emergencias-db` de Render
  declarada en el blueprint.
- Vamos a revisar la decisión cuando tengamos que integrarnos con el bus de
  EDA.

## Alternativas consideradas

| Servicio | Pros | Contras |
|---|---|---|
| **Neon** | Plan gratuito sin pausa por inactividad: el proyecto sigue disponible después de semanas sin uso. Conexión IPv4 sin configuración especial. Hasta 10 branches de base por proyecto para aislar pruebas. | Es solo PostgreSQL: no incluye storage ni autenticación. El compute se suspende tras 5 minutos de inactividad y la primera consulta paga la reanudación. |
| Supabase | Ya teníamos un proyecto creado. Incluye Storage, Auth, dashboard y API generada. | El proyecto se pausa tras una semana de inactividad y hay que reactivarlo manualmente. La conexión directa es IPv6, lo que obliga a usar el pooler desde Render. La superficie extra (Auth, API generada, RLS) invita a acoplamientos que después hay que desarmar para integrarse con el Grupo 2. |
| Render Postgres (free) | Mismo proveedor que el backend: una sola cuenta y latencia mínima. | La instancia gratuita **expira a los 30 días de creada** y se elimina tras un período de gracia. No llega viva a la segunda entrega. |
| PostgreSQL autogestionado (contenedor o VM propia) | Control total y sin límites de plan gratuito. | Backups, actualizaciones y disponibilidad quedan a cargo del equipo: trabajo de operación que no podemos sostener en dos sprints. |

## Consecuencias

Se hace más fácil:

- La base no se pausa por inactividad: no hay ritual manual de reactivación
  antes de cada demo ni riesgo de llegar a la entrega con el proyecto dormido.
- La conexión desde Render funciona sin resolver problemas de compatibilidad
  IPv4/IPv6.
- Cambiar de proveedor más adelante es, en la práctica, cambiar el valor de
  `DATABASE_URL`, porque no dependemos de features propietarias.
- Cada integrante desarrolla contra su propia base local, sin pisar los datos
  de los demás ni los de la demo.

Se hace más difícil:

- Perdemos el Storage y la autenticación que Supabase traía incluidos: los
  adjuntos van a requerir un proveedor adicional (decisión diferida) y la
  identidad queda del lado del componente de autenticación de la plataforma,
  que es a donde queríamos que fuera de todos modos.
- El compute se suspende tras 5 minutos de inactividad: la primera consulta
  después de un rato paga la reanudación, que se suma al arranque en frío del
  servicio en Render.
- El plan gratuito limita a 0,5 GB de almacenamiento y 100 horas de compute
  por proyecto por mes. Alcanza de sobra para el MVP, pero hay que
  monitorearlo.
- La credencial de Supabase que circuló por el chat del equipo debe rotarse o
  el proyecto eliminarse, aunque ya no se use.

## Historial

- 2026-09-07: creación del ADR (estado: propuesta)
