# Usar SonarCloud como análisis estático complementario

- Estado: propuesta
- Fecha: 2026-09-24
- Decisores: @KevinAlajarin (DevOps), @valenfiumana (Security), @juanimoli (Producto)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

El pipeline de CI ya cubre lint, build, tests unitarios y de integración con
umbral de cobertura del 80% en Jest y Vitest. Con eso garantizamos que el
código compila, cumple estilo y está cubierto por tests, pero no medimos
**calidad estructural**: duplicación, complejidad ciclomática, code smells,
vulnerabilidades conocidas de dependencias o hotspots de seguridad. Tampoco
tenemos un panel externo donde el resto del grupo o los profesores puedan
consultar el estado de calidad sin correr el pipeline.

Queremos una capa complementaria de análisis estático que se integre con
GitHub Actions, lea los reportes de cobertura de Jest y Vitest, y sea
gratuita para proyectos privados o al menos accesible con el plan
educativo.

## Decisión

- Vamos a usar **SonarCloud** (SaaS, no SonarQube self-hosted) como
  herramienta de análisis estático complementaria a los tests.
- Vamos a crear **dos proyectos** en la organización `g6d2` (región EU,
  dominio `sonarcloud.io`): `G6D2_emergencias-backend` y `G6D2_frontend`.
- Vamos a agregar un **step de SonarCloud al final del job de tests** en
  ambos workflows, con `continue-on-error: true`, y a alimentarle el
  reporte `lcov` que ya generan Jest y Vitest.
- Vamos a autenticar el step con un secret `SONAR_TOKEN` cargado en cada
  repo, y a configurar el proyecto vía `sonar-project.properties` en la
  raíz.
- Vamos a **mantener a SonarCloud como radar de calidad, no como gate
  bloqueante**: un Quality Gate rojo de Sonar no traba el merge. El gate
  efectivo del pipeline sigue siendo lint + build + tests + cobertura ≥
  80% + integración.
- Vamos a revisar el reporte de Sonar como parte del review manual de cada
  PR y priorizar los issues de tipo "vulnerability" y "bug".

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **SonarCloud (SaaS)** | Gratis para proyectos públicos; integración nativa con GitHub PRs; sin infraestructura que mantener; consume `lcov` de Jest y Vitest sin adaptadores. | Dependencia de servicio externo. Requiere el secret `SONAR_TOKEN`. Como no es gate, un rojo se puede ignorar. |
| SonarQube self-hosted | Control total sobre reglas y datos. Sin límites de análisis. | Hay que hostear, mantener y actualizar un servidor. Fuera del alcance de un equipo junior con presupuesto cero. |
| Codecov | Muy focalizado en cobertura, buen UI para diffs. | Solo cobertura: no cubre code smells, complejidad ni vulnerabilities. |
| CodeClimate | Similar a Sonar en features. | Menos difundido en la cátedra; onboarding adicional para el equipo. |
| No usar ninguna herramienta externa | Cero configuración. | Sin visibilidad de deuda técnica, complejidad ni vulnerabilities. Los reviews manuales quedan como única defensa. |

## Consecuencias

Se hace más fácil:

- Ver un panel público de calidad y cobertura sin correr el pipeline.
- Detectar duplicación, complejidad alta y code smells que ESLint no
  reporta.
- Justificar la dimensión de calidad de la rúbrica con métricas
  independientes del propio equipo.
- Sumar Sonar como gate bloqueante en el futuro sin cambiar de herramienta:
  bastaría con quitar `continue-on-error: true` del step.

Se hace más difícil:

- Un Quality Gate rojo puede pasar inadvertido si nadie mira el reporte:
  hay que incorporar la lectura de Sonar al checklist de review.
- El análisis suma segundos al pipeline y depende de la disponibilidad de
  SonarCloud.
- La rotación del `SONAR_TOKEN` es un paso operativo adicional.

## Historial

- 2026-09-24: creación del ADR (estado: propuesta)
