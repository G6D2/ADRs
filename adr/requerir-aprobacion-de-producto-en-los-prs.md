# Requerir aprobación de Producto en los PRs

- Estado: aceptada
- Fecha: 2026-08-13
- Decisores: Producto, administrador del repositorio
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Los equipos de la organización (Back, Front, Desarrollo) proponen decisiones
de arquitectura mediante pull requests. Sin una regla de gobernanza, cualquier
persona con permiso de escritura puede aprobar y mergear un PR, y una decisión
podría formalizarse sin la revisión del equipo de Producto ni del
administrador.

## Decisión

Vamos a restringir la aprobación de los pull requests al equipo de Producto y
al administrador del repositorio:

- `.github/CODEOWNERS` declara a `@G6D2/producto` y a `@Salterm27` como
  propietarios de todo el repositorio.
- Un ruleset sobre la rama `main` exige pull request con al menos una
  aprobación y activa «Require review from Code Owners»: solo cuentan las
  aprobaciones de los propietarios declarados.
- Los demás equipos conservan permiso de escritura para crear ramas y abrir
  PRs, pero sus aprobaciones no habilitan el merge.

## Consecuencias

- Ninguna decisión se formaliza sin el visto bueno de Producto o del
  administrador, y la aprobación queda registrada en el PR.
- Producto se convierte en cuello de botella deliberado: los PRs de ADRs
  esperan su revisión.
- La regla se aplica en la plataforma (GitHub), no por convención: no depende
  de que los equipos la recuerden.
- El ruleset debe configurarse una única vez en Settings → Rules → Rulesets
  (ver README, sección «Gobernanza de aprobaciones»).

## Historial

- 2026-08-13: creación del ADR (estado: propuesta)
- 2026-08-13: cambio de estado: propuesta → aceptada
