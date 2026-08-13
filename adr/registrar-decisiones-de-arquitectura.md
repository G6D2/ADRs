# Registrar decisiones de arquitectura

- Estado: aceptada
- Fecha: 2026-08-13
- Decisores: equipo de arquitectura
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Necesitamos registrar las decisiones de arquitectura que se toman en este
proyecto: qué se decidió, en qué contexto, qué alternativas se consideraron y
qué consecuencias se aceptaron. Sin un registro, el conocimiento queda en
conversaciones y personas, y las decisiones se revisitan una y otra vez sin
memoria de por qué se tomaron.

## Decisión

Vamos a usar Architecture Decision Records (ADRs) siguiendo los lineamientos de
<https://github.com/G6D2/architecture-decision-record>:

- Plantilla estilo Michael Nygard: Título, Estado, Contexto, Decisión,
  Consecuencias.
- Un archivo Markdown por decisión en el directorio `adr/`, con nombre en
  minúsculas-con-guiones formado por una frase verbal en imperativo
  (p. ej. `elegir-base-de-datos.md`).
- Estados: propuesta, aceptada, rechazada, obsoleta, reemplazada.
- Inmutabilidad: un ADR aceptado no se reescribe; si la decisión cambia, se
  crea un ADR nuevo que lo reemplaza (`tools/adr supersede`).
- Cada ADR se propone y revisa mediante un pull request.

## Consecuencias

- Las decisiones y su justificación quedan versionadas junto al código y se
  revisan con el mismo flujo (pull requests).
- Los nuevos integrantes pueden reconstruir el porqué de la arquitectura
  leyendo el registro.
- Escribir el ADR tiene un costo: cada decisión significativa requiere
  documentar contexto y consecuencias antes de aceptarse.
- El índice y la validación se automatizan con `tools/adr` y GitHub Actions,
  lo que reduce el mantenimiento manual del registro.

## Historial

- 2026-08-13: creación del ADR (estado: propuesta)
- 2026-08-13: cambio de estado: propuesta → aceptada
