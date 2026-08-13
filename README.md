# Gestión de ADRs

Herramienta de gestión de **Architecture Decision Records (ADRs)** basada en
GitHub, siguiendo los lineamientos de
[G6D2/architecture-decision-record](https://github.com/G6D2/architecture-decision-record).

## Estructura

```
adr/                          Registro de decisiones (un .md por decisión)
├── README.md                 Índice generado automáticamente
├── templates/plantilla.md    Plantilla (estilo Michael Nygard)
└── *.md                      ADRs
tools/adr                     CLI de gestión (bash, sin dependencias)
.github/workflows/            Validación e índice automáticos
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

Ver [adr/README.md](adr/README.md).

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
```

