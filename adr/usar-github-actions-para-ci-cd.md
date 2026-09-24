# Usar GitHub Actions como plataforma de CI/CD

- Estado: propuesta
- Fecha: 2026-09-24
- Decisores: @KevinAlajarin (DevOps), @juanimoli (Producto), @Salterm27 (Scrum Master)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Los repositorios `emergencias-backend` y `Frontend` viven en la organización
`G6D2` de GitHub. Necesitamos automatizar tres cosas antes de cada merge a
`main`: (1) validación estática y estilística (lint), (2) ejecución de los
tests unitarios y de integración con reporte de cobertura, y (3) deploy
automático a Render cuando el commit llega a `main`. También queremos
notificaciones ante fallos, comentarios de cobertura sobre los PRs y un
mecanismo para gatear la calidad (umbral 80% en Jest/Vitest).

El equipo es junior en DevOps, el TPO tiene presupuesto cero y ya usamos
GitHub para código, issues, PRs y CODEOWNERS. Cualquier plataforma externa
implicaría alta y mantenimiento adicionales.

## Decisión

- Vamos a usar **GitHub Actions** como única plataforma de CI/CD para
  `emergencias-backend` y `Frontend`.
- Vamos a tener **un único workflow `ci.yml` por repo** que dispare en
  `pull_request` y en `push` a `main`, con jobs para lint, build, tests
  unitarios con cobertura, tests de integración contra Postgres real
  (servicio efímero del runner) y aplicación de migraciones con Flyway
  antes de correr integración.
- Vamos a **deployar a Render vía deploy hook** (`curl` autenticado al hook)
  solo después de que todos los jobs de `main` estén verdes.
- Vamos a manejar credenciales (`DATABASE_URL`, `SONAR_TOKEN`,
  `RENDER_DEPLOY_HOOK_URL`, SMTP para notificaciones) como **GitHub
  Secrets** a nivel repo, nunca hardcodeadas.
- Vamos a hacer que el pipeline **corte el merge** si la cobertura cae bajo
  el 80% (umbral configurado directamente en Jest y Vitest, no en el
  workflow), y a publicar un comentario con el reporte en cada PR.
- Vamos a mantener un job de `notify-failure` que envíe un email al equipo
  cuando cualquier job del pipeline falla.

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
- Gatear merges por cobertura y por lint sin herramientas externas.
- Deployar a Render de forma reproducible: siempre el mismo commit de `main`
  con los mismos checks verdes.
- Sumar nuevos jobs (SonarCloud, security scans, tests E2E) como pasos del
  mismo workflow.

Se hace más difícil:

- Toda la línea de build depende de la disponibilidad de GitHub Actions:
  un incidente de la plataforma frena los deploys.
- Los minutos gratis son finitos: si la suite crece mucho o se hacen muchos
  PRs, hay que pagar o cachear más agresivamente.
- Migrar a otra plataforma en el futuro requiere reescribir los workflows.

## Historial

- 2026-09-24: creación del ADR (estado: propuesta)
