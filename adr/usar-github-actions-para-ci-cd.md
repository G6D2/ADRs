# Usar GitHub Actions como plataforma de CI/CD

- Estado: aceptada
- Fecha: 2026-09-24
- Decisores: @KevinAlajarin (DevOps), @juanimoli (Producto), @Salterm27 (Scrum Master)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Los repositorios `emergencias-backend` y `Frontend` viven en la organización
`G6D2` de GitHub. Necesitamos automatizar antes de cada merge a `main`:
(1) validación estática y estilística (lint), (2) ejecución de los tests
unitarios y de integración con reporte de cobertura, y (3) deploy
automático a Render cuando el commit llega a `main`. También queremos
notificaciones ante fallos, comentario de cobertura sobre los PRs, un
despertador para el plan free de Render en el horario de clase y un
mecanismo manual de failover al sitio de backup.

Además, los repos son **privados en plan free de GitHub**, lo que implica
que **no podemos aplicar branch protection** desde la UI o la API (la API
de rulesets devuelve 403 en ese plan). El gate de calidad sobre `main`
tiene que vivir por acuerdo del equipo + revisión manual del PM, no por
una regla técnica forzada por la plataforma. CODEOWNERS tampoco está
uniformemente desplegado: `emergencias-backend` tiene
`.github/CODEOWNERS` con `* @Salterm27` (el comentario dice "aprobación
del PM" aunque el owner es el Scrum Master, inconsistencia a corregir),
y `Frontend` no tiene archivo CODEOWNERS. Mientras esto no se
uniformice, el gate real es social: PR → aprobación explícita del PM
antes del merge.

El equipo es junior en DevOps, el TPO tiene presupuesto cero y ya usamos
GitHub para código, issues, PRs y CODEOWNERS. Cualquier plataforma externa
implicaría alta y mantenimiento adicionales.

## Decisión

- Vamos a usar **GitHub Actions** como única plataforma de CI/CD para
  `emergencias-backend` y `Frontend`.
- Vamos a tener **cuatro workflows en el backend** (`ci.yml`, `cd.yml`,
  `keep-warm.yml`, `activate-backup.yml`) y **tres en el frontend**
  (`ci.yml`, `cd.yml`, `activate-backup.yml`). El frontend no tiene
  `keep-warm` propio: lo despierta el job in-process del backend pegándole
  a `/api/warmup` dentro de la ventana de clase.
- `ci.yml` dispara en `pull_request` a `main` y en `push` a `main`, con un
  único job `lint-and-test` que corre, en pasos, lint, build (solo en el
  front — el back no tiene build step), tests unitarios con cobertura y
  tests de integración. En el backend la integración usa **Postgres real**
  como service container del runner y aplica migraciones con **Flyway**
  (`npm run migrate`) antes de los tests. En el frontend la integración
  es la del BFF con Vitest, sin base. Además hay un job `validate-pr` que
  exige título y descripción no vacíos.
- `cd.yml` dispara por `workflow_run` cuando `ci.yml` termina en `success`
  sobre `main`, no en el push directo: así deployamos exactamente el SHA
  que CI aprobó, aunque `main` haya avanzado. En el **backend**, el job se
  llama `Migrate & Deploy` y **corre migraciones Flyway contra la base
  productiva** (`DATABASE_URL` secreto, `npm run migrate`) **antes** de
  hacer `curl` al deploy hook de Render. En el **frontend**, el CD solo
  dispara el deploy hook (sin migraciones).
- `keep-warm.yml` (solo backend) corre con cron los jueves en el horario
  de clase (ART) para hacer el primer despertar del servicio en plan free
  de Render; a partir de ahí lo mantiene vivo el job in-process del propio
  backend, que pinguea `/health` propio y `/api/warmup` del frontend.
- `activate-backup.yml` (ambos repos) es manual (`workflow_dispatch`) y
  dispara el deploy hook del sitio backup (cuenta Render personal de
  Kevin) y espera `/health` 200 (back) o `warmed:true` en `/api/warmup`
  (front).
- Vamos a deployar a Render **vía deploy hook** (`curl` autenticado al
  hook) solo después de que CI de `main` esté verde.
- Vamos a manejar credenciales como **GitHub Secrets** a nivel repo, nunca
  hardcodeadas. Los secrets activos son:
  - En ambos repos: `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_TO`
    (notificación por email ante fallos), `RENDER_DEPLOY_HOOK_URL` (deploy
    primary), `RENDER_DEPLOY_HOOK_URL_BACKUP` (deploy backup) y
    `SONAR_TOKEN` (análisis externo).
  - Solo backend: `DATABASE_URL` (usado por `cd.yml` para correr
    migraciones Flyway contra la base de prod).
- Vamos a **medir** cobertura y publicar un comentario único con el
  reporte en cada PR, con **umbral 80%** configurado directamente en Jest
  (back) y Vitest (front). Ese 80% es un gate **interno** del equipo; la
  cátedra pide 60%. El umbral se evalúa dentro del test runner y si falla
  rompe el job de tests, así que CI queda rojo. Lo que **no** hacemos es
  trabar el merge con branch protection: como los repos son privados en
  plan free, el merge se bloquea **por acuerdo** (aprobación explícita
  del PM en el PR), no por una regla técnica forzada por GitHub.
  CODEOWNERS ayuda sólo donde existe hoy (back `* @Salterm27`; front sin
  archivo), así que el gate operativo es el visto bueno del PM.
- Vamos a mantener un job `notify-failure` en cada workflow de CI y CD
  que envíe un email al equipo cuando cualquier job falla.
- Vamos a correr **SonarCloud** como step del job de tests (ambos repos,
  con `continue-on-error: true`, condicionado a `SONAR_TOKEN`). Hoy ya
  está corriendo; el detalle vive en su propio ADR
  (`usar-sonarcloud-como-analisis-complementario.md`).

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **GitHub Actions** | Integración nativa con el repo, PRs y CODEOWNERS. 2000 minutos/mes gratis para privados. Secrets y matrix builds sin infra. Ecosistema amplio de actions oficiales. | Vendor lock-in con GitHub. Los minutos gratis se pueden agotar si crece la suite. |
| GitLab CI | Excelente pipeline as code, entorno maduro. | Obligaría a espejar el repo en GitLab o migrar; rompe el flujo de PRs y CODEOWNERS que ya tenemos. |
| CircleCI | Buena performance, cache configurable. | Otra cuenta más para el equipo, otro sistema de secrets, mayor curva de aprendizaje. |
| Jenkins self-hosted | Control total, sin límite de minutos. | Requiere hostear y mantener un servidor: fuera del alcance de un equipo junior con presupuesto cero. |
| Solo el build de Render (sin CI externo) | Cero configuración adicional. | No corre tests, no lintea, no gatea cobertura. Un PR roto llegaría a producción sin ser detectado. |

## Consecuencias

Se hace más fácil:

- Ver el estado de calidad de cada PR sin salir de GitHub.
- Dejar que el PM vete el merge en PRs con CI rojo o cobertura baja (gate
  por acuerdo del equipo, no por branch protection; CODEOWNERS apoya sólo
  donde el archivo existe — hoy en el back).
- Deployar a Render de forma reproducible: siempre el SHA aprobado por CI,
  disparando el CD vía `workflow_run` sobre `main`.
- Correr migraciones Flyway contra la base productiva en el mismo pipeline,
  sin pasos manuales (back).
- Mantener el servicio del plan free despierto en horario de clase (back)
  y disparar el failover al sitio backup con un click.
- Sumar nuevos jobs (SonarCloud, security scans, tests E2E) como pasos del
  mismo workflow.

Se hace más difícil:

- Que las migraciones Flyway corran contra prod es el paso más riesgoso: un
  cambio malo en `db/migrations` llega a la base real apenas CI pasa.
  Mitigación: revisión obligatoria del PM en PRs que toquen migraciones +
  Flyway como motor idempotente y transaccional.
- Como no hay branch protection técnica, un merge sin aprobación del PM
  es físicamente posible: depende de disciplina del equipo. Mitigación:
  aprobación explícita del PM en cada PR y CODEOWNERS donde esté presente
  (hoy sólo en el back). Falta uniformar CODEOWNERS en el front.
- Toda la línea de build depende de la disponibilidad de GitHub Actions:
  un incidente de la plataforma frena los deploys.
- Los minutos gratis son finitos: si la suite crece mucho o se hacen muchos
  PRs, hay que pagar o cachear más agresivamente.
- Migrar a otra plataforma en el futuro requiere reescribir los workflows.

## Historial

- 2026-09-24: creación del ADR (estado: propuesta).
- 2026-10-19: ajustes del review del 24/09 — se cuentan los cuatro
  workflows del back y los tres del front; se aclara que Postgres real y
  Flyway son solo del back (en el front la integración es BFF con
  Vitest); se agrega que el CD del back corre migraciones Flyway contra
  prod antes del deploy; se corrige "corta el merge" para dejar claro que
  el gate es por acuerdo (los repos privados en plan free no admiten
  branch protection); se completan los secrets reales; se aclara que
  lint/build/tests son steps de un solo job y que el back no tiene build;
  se diferencia el 80% interno del 60% de la cátedra; se marca que Sonar
  ya está corriendo. Pasa a aceptada.
- 2026-10-20: ajustes al review del PR #23 — se refleja el estado real
  de CODEOWNERS (back tiene `.github/CODEOWNERS` con `* @Salterm27` y un
  comentario que menciona al PM; front no tiene archivo), por lo que el
  gate operativo es la aprobación explícita del PM; se corrige la ruta
  de migraciones a `db/migrations` (no `src/db/migrations`); se saca la
  referencia a `feedback-no-push`, que no existe como artefacto en los
  repos ni en el doc de flujo de trabajo.
