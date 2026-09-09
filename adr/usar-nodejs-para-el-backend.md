# Usar NodeJS para el backend

- Estado: aceptada
- Fecha: 2026-08-27
- Decisores: equipo backend
- Reemplaza a: —
- Reemplazada por: —

Propuesto en el issue [#6](https://github.com/G6D2/ADRs/issues/6).

## Contexto

Necesitamos definir con qué tecnología vamos a desarrollar el backend de la
aplicación. El backend va a tener que exponer APIs para el frontend, manejar
la lógica de negocio, conectarse con la base de datos y comunicarse con
otros servicios del sistema. Buscamos una tecnología que nos permita
desarrollar rápido, que tenga buen soporte para APIs REST y que tenga un
ecosistema amplio de librerías.

Otro punto importante es que NodeJS usa JavaScript, por lo que podemos
mantener un lenguaje similar entre frontend y backend. Esto también
simplifica bastante el trabajo del equipo y evita sumar otra tecnología
solo para el servidor.

## Decisión

- Vamos a usar **NodeJS** para desarrollar el backend: exponer las APIs,
  implementar la lógica de negocio y manejar la comunicación con la base de
  datos y otros servicios.
- Vamos a organizar la aplicación en módulos para intentar mantener
  separadas las distintas responsabilidades y evitar que toda la lógica
  quede mezclada.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **NodeJS** | Desarrollo rápido de APIs, ecosistema amplio de librerías (npm). Mismo lenguaje que el frontend (JavaScript), lo que simplifica el trabajo del equipo. Funciona bien con muchas operaciones de entrada/salida. | Da mucha libertad para organizar el proyecto: sin una estructura definida el código puede quedar desordenado. Trabaja principalmente con un único hilo, así que hay que cuidar las operaciones pesadas de CPU. |
| Java con Spring Boot | Tecnología muy usada para backend, con un ecosistema bastante completo. | Agrega bastante estructura y configuración que no se necesita en este proyecto. |
| Python (FastAPI o Django) | Alternativa simple y con bastante documentación. | El equipo tiene más experiencia con JavaScript y NodeJS; no se ve una ventaja importante en cambiar de lenguaje. |
| .NET | Permite construir APIs robustas y tiene buenas herramientas para proyectos grandes. | No es una tecnología tan conocida por el equipo; sumaría complejidad sin aportar una ventaja clara. |

## Consecuencias

Se hace más fácil:

- Desarrollar APIs de forma rápida, gracias a la gran cantidad de
  librerías disponibles en npm.
- Manejar aplicaciones con muchas operaciones de entrada/salida, como
  requests HTTP o consultas a servicios externos.

Se hace más difícil:

- Mantener el código ordenado si no se define una estructura del proyecto
  desde el principio, dado que NodeJS da mucha libertad para organizarlo.
- Manejar operaciones pesadas de CPU, ya que NodeJS trabaja principalmente
  con un único hilo para ejecutar código JavaScript.

## Historial

- 2026-08-27: creación del ADR (estado: propuesta)
- 2026-09-07: cambio de estado: propuesta → aceptada
