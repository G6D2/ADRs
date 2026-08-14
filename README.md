# Gestión de ADRs

Herramienta de gestión de **Architecture Decision Records (ADRs)** basada en
GitHub y publicada como sitio en **GitHub Pages**, siguiendo los lineamientos
de
[G6D2/architecture-decision-record](https://github.com/G6D2/architecture-decision-record).

📖 **Sitio publicado: <https://g6d2.github.io/ADRs/>**

## Estructura

```
adr/                          Registro de decisiones (un .md por decisión)
├── README.md                 Índice generado automáticamente
├── templates/plantilla.md    Plantilla (estilo Michael Nygard)
└── *.md                      ADRs
tools/adr                     CLI de gestión (bash, sin dependencias)
tools/site/                   Assets del sitio de GitHub Pages
.github/CODEOWNERS            Gobernanza: quién aprueba los PRs
.github/workflows/            Validación, índice y despliegue del sitio
```

## Convenciones (según los lineamientos)

- **Plantilla estilo Michael Nygard**: Título, Estado, Contexto, Decisión,
  Consecuencias — más metadatos (fecha, decisores) e historial con fechas.
- **Nombres de archivo**: frase verbal en imperativo, en minúsculas con
  guiones. Ejemplos: `elegir-base-de-datos.md`, `formatear-timestamps.md`,
  `gestionar-contrasenas.md`.
- **Estados**: `propuesta` → `aceptada` | `rechazada`, y luego
  `obsoleta` o `reemplazada`.
- **Inmutabilidad**: un ADR aceptado no se reescribe. Si la decisión cambia,
  se crea un ADR nuevo que reemplaza al anterior y ambos quedan enlazados.

## Índice de decisiones

Ver el [sitio publicado](https://g6d2.github.io/ADRs/) o el índice en el
repositorio: [adr/README.md](adr/README.md).

## Flujo de trabajo en GitHub

1. **Proponer**: abrir un issue con la plantilla «Propuesta de ADR» para
   discutir la decisión, o directamente crear una rama y ejecutar
   `tools/adr new "<título>"`.
2. **Redactar**: completar Contexto, Decisión y Consecuencias en el archivo
   generado. El ADR nace con estado `propuesta`.
3. **Revisar**: abrir un pull request. El workflow `adr-lint` valida la
   convención y que el índice esté actualizado; la decisión se discute en la
   revisión del PR.
4. **Aceptar**: antes del merge, ejecutar
   `tools/adr status <archivo> aceptada` (o `rechazada` si se documenta el
   descarte). El merge del PR formaliza la decisión.
5. **Evolucionar**: si más adelante la decisión cambia, usar
   `tools/adr supersede` para crear el ADR que la reemplaza — nunca editar la
   decisión original.

El workflow `adr-index` regenera el índice automáticamente en cada push a la
rama principal que toque `adr/`, por lo que el índice nunca queda
desactualizado.

## Uso de la CLI

Desde la raíz del repositorio:

```bash
# Crear un ADR nuevo (crea adr/elegir-base-de-datos.md y actualiza el índice)
tools/adr new "elegir base de datos"

# Listar los ADRs con estado y fecha
tools/adr list

# Cambiar el estado (queda registrado en el historial del ADR)
tools/adr status elegir-base-de-datos.md aceptada

# Reemplazar una decisión por una nueva (enlaza ambos ADRs)
tools/adr supersede elegir-base-de-datos.md "migrar a postgresql"

# Validar todos los ADRs (secciones, estado, fecha)
tools/adr lint

# Regenerar el índice adr/README.md
tools/adr index

# Generar el sitio de GitHub Pages en _site/ (para probarlo localmente)
tools/adr site
```

## Sitio en GitHub Pages

El registro se publica como sitio estático: un índice con búsqueda y filtros
por estado, más un visor por ADR con su historial. No tiene dependencias
externas ni backend; es de solo lectura y la gestión sigue siendo por pull
requests.

- Está publicado en <https://g6d2.github.io/ADRs/>.
- El workflow [`adr-pages`](.github/workflows/adr-pages.yml) genera el sitio
  con `tools/adr site` y lo despliega en cada push a `main`, por lo que el
  sitio siempre refleja el estado del registro. También puede lanzarse a mano
  desde la pestaña Actions (`workflow_dispatch`).
- Para probarlo localmente: `tools/adr site && python3 -m http.server -d _site`
  y abrir <http://localhost:8000>.

> GitHub Pages sirve el sitio públicamente. Si el repositorio vuelve a ser
> privado, la publicación requiere un plan de pago (Team o Enterprise).

## Gobernanza de aprobaciones

Los equipos (Back, Front, Desarrollo) proponen ADRs mediante pull requests,
pero **solo el equipo de Producto o el administrador pueden aprobarlos**. Esto
se apoya en tres piezas:

**1. Acceso de los equipos** (Settings → Collaborators and teams → **Add
teams**). Todos los equipos deben ser colaboradores del repositorio con
permiso de **escritura** (`Write`):

| Equipo | Permiso | Para qué |
|---|---|---|
| `producto` | Write | Requisito de GitHub para ser code owner válido y poder aprobar. |
| `back`, `front`, `desarrollo` | Write | Crear ramas y abrir PRs sin tener que forkear. |

> **Importante**: GitHub exige permiso de escritura para que un code owner sea
> válido. Mientras `@G6D2/producto` no esté agregado, esa entrada del
> `CODEOWNERS` se ignora en silencio y solo cuenta la aprobación de
> `@Salterm27`. GitHub marca las entradas inválidas con un aviso al abrir el
> archivo `CODEOWNERS` en la web.

**2. Propietarios del código**: [`.github/CODEOWNERS`](.github/CODEOWNERS)
declara a `@G6D2/producto` y a `@Salterm27` como propietarios de todo el
repositorio.

**3. Ruleset sobre `main`** (Settings → Rules → Rulesets → **New branch
ruleset**), que es lo que hace exigible al `CODEOWNERS`:

1. **Target branches**: `main` (o "Default branch").
2. Activar **Require a pull request before merging**, con
   **Required approvals: 1** y **Require review from Code Owners**.
3. Recomendado: **Dismiss stale pull request approvals when new commits
   are pushed** y **Block force pushes**.

Con las tres piezas, las aprobaciones de otros equipos no habilitan el merge:
solo cuentan las de Producto o las del administrador.
