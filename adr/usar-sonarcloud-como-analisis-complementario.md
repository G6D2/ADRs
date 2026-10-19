# Usar SonarCloud como análisis estático complementario

- Estado: aceptada
- Fecha: 2026-09-24
- Decisores: @KevinAlajarin (DevOps), @valenfiumana (Security), @juanimoli (Producto)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

El pipeline de CI ya cubre lint, build (solo front), tests unitarios y de
integración con umbral de cobertura del 80% en Jest y Vitest. Con eso
garantizamos que el código compila, cumple estilo y está cubierto por
tests, pero no medimos **calidad estructural**: duplicación, complejidad
ciclomática, code smells, vulnerabilidades conocidas de dependencias o
hotspots de seguridad. Tampoco tenemos un panel externo donde el resto
del grupo o los profesores puedan consultar el estado de calidad sin
correr el pipeline.

Queremos una capa complementaria de análisis estático que se integre con
GitHub Actions y lea los reportes de cobertura de Jest y Vitest. El
costo es cero mientras los **proyectos en SonarCloud** sean públicos
(visibilidad en SonarCloud, no en GitHub), aunque los repos de código
sean privados.

## Decisión

- Vamos a usar **SonarCloud** (SaaS, no SonarQube self-hosted) como
  herramienta de análisis estático complementaria a los tests.
- Vamos a crear **dos proyectos** en la organización `g6d2` (región EU,
  dominio `sonarcloud.io`): `G6D2_emergencias-backend` y `G6D2_frontend`.
  Ambos quedan con **visibilidad pública en SonarCloud** para que los
  profes puedan abrirlos sin cuenta ni invitación — esto es independiente
  de que los repos de GitHub sigan siendo privados: SonarCloud ofrece
  análisis gratis mientras el proyecto Sonar sea público, no exige repo
  público. (Nota operativa: hoy los dos proyectos están creados en
  SonarCloud pero con visibilidad "private"; falta pasarlos a "public"
  desde Administration → Permissions. Mientras no se haga, el acceso
  anónimo devuelve "Project doesn't exist".)
- Vamos a agregar un **step de SonarCloud al final del job `lint-and-test`**
  en ambos workflows de CI, con `continue-on-error: true`, y a alimentarle
  el reporte `lcov` que ya generan Jest y Vitest.
- Vamos a autenticar el step con un secret `SONAR_TOKEN` cargado en cada
  repo, y a configurar el proyecto vía `sonar-project.properties` en la
  raíz.
- El scope del análisis es distinto en cada repo y lo asumimos
  explícitamente:
  - Back: `sonar.sources=src` (todo el código del backend, con
    `src/tests/**`, `src/index.js` y `src/swagger.js` excluidos).
  - Front: `sonar.sources=lib,app/lib`. Esto deja **fuera** del análisis
    las ~55 unidades de `app/**` que no son `app/lib`, los componentes
    React y las rutas BFF. La decisión es consciente: `lib` y `app/lib`
    concentran la lógica pura y testeable; `app/**` fuera de `lib` es
    mayormente JSX y rutas de Next que generan ruido en Sonar sin aportar
    señal. Las métricas de duplicación y complejidad del front son, por
    lo tanto, parciales por diseño.
- Vamos a **mantener a SonarCloud como radar de calidad, no como gate
  bloqueante**: un Quality Gate rojo de Sonar no traba el merge. El gate
  efectivo del pipeline sigue siendo lint + build + tests + cobertura ≥
  80% + integración. El umbral 80% es gate interno del equipo; la cátedra
  pide 60%.
- Vamos a revisar manualmente el reporte de Sonar en los PRs grandes o
  cuando el checkmark de Sonar en el PR venga rojo, priorizando los
  issues de tipo "vulnerability" y "bug". No formalizamos todavía esa
  revisión en una plantilla de PR (los repos aún no tienen
  `PULL_REQUEST_TEMPLATE.md`); si creamos una, agregarla ahí es un paso
  trivial.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **SonarCloud (SaaS)** | Gratis mientras los proyectos SonarCloud sean públicos, incluso con repo GitHub privado; integración nativa con GitHub PRs; sin infraestructura que mantener; consume `lcov` de Jest y Vitest sin adaptadores. | Dependencia de servicio externo. Requiere el secret `SONAR_TOKEN`. Como no es gate, un rojo se puede ignorar. |
| SonarQube self-hosted | Control total sobre reglas y datos. Sin límites de análisis. | Hay que hostear, mantener y actualizar un servidor. Fuera del alcance de un equipo junior con presupuesto cero. |
| Codecov | Muy focalizado en cobertura, buen UI para diffs. | Solo cobertura: no cubre code smells, complejidad ni vulnerabilities. |
| CodeClimate | Similar a Sonar en features. | Menos difundido en la cátedra; onboarding adicional para el equipo. |
| No usar ninguna herramienta externa | Cero configuración. | Sin visibilidad de deuda técnica, complejidad ni vulnerabilities. Los reviews manuales quedan como única defensa. |

## Consecuencias

Se hace más fácil:

- Ver un panel externo de calidad y cobertura sin correr el pipeline
  (una vez que los proyectos pasen a visibilidad pública en SonarCloud).
- Detectar duplicación, complejidad alta y code smells que ESLint no
  reporta, al menos en las partes del código cubiertas por el scope de
  Sonar (todo el back; `lib` y `app/lib` del front).
- Justificar la dimensión de calidad de la rúbrica con métricas
  independientes del propio equipo.
- Sumar Sonar como gate bloqueante en el futuro sin cambiar de herramienta:
  bastaría con quitar `continue-on-error: true` del step.

Se hace más difícil:

- Un Quality Gate rojo puede pasar inadvertido si nadie mira el reporte:
  mitigación informal mientras no haya plantilla de PR.
- La cobertura de duplicación y complejidad en el frontend es parcial
  por el scope elegido; la métrica no refleja la calidad de `app/**`
  fuera de `app/lib`.
- La action oficial que usamos hoy (`SonarSource/sonarcloud-github-action@v3`)
  está marcada como **deprecada**. El reemplazo recomendado es
  `SonarSource/sonarqube-scan-action`. Queda como deuda operativa migrar
  el step en ambos repos.
- El análisis suma segundos al pipeline y depende de la disponibilidad de
  SonarCloud.
- La rotación del `SONAR_TOKEN` es un paso operativo adicional.
- Los comentarios actuales en `ci.yml` de ambos repos dicen "SonarCloud
  gratis exige repo público", lo que es incorrecto: la condición es
  proyecto público en SonarCloud. Queda como deuda corregir ese
  comentario en PRs separados en `emergencias-backend` y `Frontend`.

## Historial

- 2026-09-24: creación del ADR (estado: propuesta).
- 2026-10-19: ajustes del review del 24/09 — se explica que la gratuidad
  depende de la visibilidad del proyecto en SonarCloud, no del repo
  GitHub; se marca como nota operativa que los proyectos están creados
  pero todavía privados en SonarCloud y hay que pasarlos a public para
  que los profes accedan; se asume explícitamente el scope reducido del
  análisis del front (`lib,app/lib`); se registra que la action en uso
  está deprecada y queda como deuda migrarla; se saca la afirmación de
  que "revisamos Sonar en cada PR" (no hay plantilla donde quede
  anotado) y se reemplaza por la práctica real; se diferencia el 80%
  interno del 60% de la cátedra; se registra la deuda de corregir el
  comentario del workflow sobre "repo público". Pasa a aceptada.
