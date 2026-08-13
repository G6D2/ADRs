# Publicar el registro en GitHub Pages

- Estado: aceptada
- Fecha: 2026-08-13
- Decisores: equipo de arquitectura, Producto
- Reemplaza a: —
- Reemplazada por: —

## Contexto

El registro de ADRs vive como archivos Markdown en el repositorio. Leerlos
requiere navegar GitHub, y las personas fuera del flujo de desarrollo
(producto, stakeholders) necesitan una forma más accesible de consultar las
decisiones: buscar, filtrar por estado y leer cada decisión con su historial.

## Decisión

Vamos a publicar el registro como un sitio estático en GitHub Pages:

- `tools/adr site` genera el sitio en `_site/`: un índice con búsqueda y
  filtros por estado, un visor por ADR y un manifiesto `adrs.json`.
- El workflow `adr-pages` regenera y despliega el sitio en cada push a `main`,
  por lo que el sitio siempre refleja el estado del registro.
- El sitio es de solo lectura: no hay dependencias externas ni backend; la
  gestión sigue siendo mediante la CLI y pull requests.

## Consecuencias

- Cualquier persona con el enlace puede consultar las decisiones sin navegar
  el repositorio ni conocer Git.
- El despliegue es automático: no hay pasos manuales de publicación.
- Hay que habilitar Pages una única vez en la configuración del repositorio
  (Settings → Pages → Source: GitHub Actions).
- El sitio agrega superficie de mantenimiento (HTML/CSS/JS propios), acotada
  por no tener dependencias externas.

## Historial

- 2026-08-13: creación del ADR (estado: propuesta)
- 2026-08-13: cambio de estado: propuesta → aceptada
