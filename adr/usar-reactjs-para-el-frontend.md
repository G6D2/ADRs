# Usar ReactJS para el frontend

- Estado: aceptada
- Fecha: 2026-08-27
- Decisores: equipo
- Reemplaza a: —
- Reemplazada por: —

Propuesto en el issue [#5](https://github.com/G6D2/ADRs/issues/5).

## Contexto

Necesitamos definir con qué tecnología vamos a desarrollar el frontend de la
aplicación. La idea es tener una interfaz web dinámica, que consuma las APIs
del backend y que nos permita ir agregando funcionalidades sin que el código
se vuelva difícil de mantener.

También nos interesa poder reutilizar componentes, separar bastante bien la
parte visual de la lógica del backend y usar una tecnología que tenga buena
documentación, bastante comunidad y librerías disponibles.

Otro punto importante es que React ya es conocido por parte del equipo, así
que no tendríamos que sumar una curva de aprendizaje grande solamente para
arrancar el proyecto.

## Decisión

- Vamos a utilizar **ReactJS** para desarrollar el frontend de la aplicación.
- La interfaz se va a organizar en componentes reutilizables y React se va a
  encargar principalmente del manejo de la UI y de sus estados.
- La comunicación con el backend se va a hacer mediante las APIs definidas
  por el sistema.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **ReactJS** | Ya es conocido por el equipo, sin curva de aprendizaje adicional para arrancar el proyecto. Permite componentes reutilizables y separar bien la UI de la lógica del backend. Buena documentación, comunidad y librerías disponibles. | Depende de librerías adicionales del ecosistema para resolver routing, manejo de estado o requests. Da bastante libertad de estructura, lo que puede generar código inconsistente si no se define un criterio claro. |
| Angular | Ofrece una solución bastante completa, con muchas herramientas integradas. | Más pesado de lo necesario para el alcance del proyecto. Estructura más rígida. Implica una curva de aprendizaje mayor para parte del equipo. |
| Vue.js | Curva de aprendizaje bastante amigable. Sería una opción válida para el proyecto. | El equipo tiene más experiencia previa con React. No se encontró una ventaja concreta que justifique cambiar de tecnología. |
| JavaScript / HTML / CSS sin framework | No depende de un framework o librería específica. | A medida que crece la aplicación, manejar estados, reutilizar componentes y mantener organizada la interfaz puede volverse más complejo. |

## Consecuencias

Se hace más fácil:

- Trabajar con componentes reutilizables y mantener una estructura
  relativamente modular.
- Aprovechar una gran cantidad de librerías y documentación existente de
  React.
- Que distintos integrantes del equipo trabajen sobre diferentes partes del
  frontend.

Se hace más difícil:

- Depender del ecosistema de React, lo que probablemente implique incorporar
  librerías adicionales para resolver cuestiones como routing, manejo de
  estado o requests.
- Mantener una estructura consistente entre las distintas partes del
  proyecto, ya que React da bastante libertad si no se define un criterio
  claro desde el principio.

## Historial

- 2026-08-27: creación del ADR (estado: propuesta)
- 2026-09-07: cambio de estado: propuesta → aceptada
