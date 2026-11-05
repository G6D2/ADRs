# Calcular los recursos requeridos con una tabla de reglas

- Estado: aceptada
- Fecha: 2026-10-16
- Decisores: @juanimoli (Producto), @Matiamistadi y @luccaperazzo (Back)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

[Informar los recursos requeridos como un campo del cambio de estado](informar-los-recursos-requeridos-como-un-campo-del-cambio-de-estado.md) define qué publicamos: un campo `recursosRequeridos` (Bomberos, Policía o Ambulancia). Falta decidir cómo lo calculamos y cuándo sale.

Una misma respuesta se puede originar en disparadores distintos:

- **Internos:** un usuario reporta un incendio en nuestra plataforma y se crea la emergencia por HTTP.
- **Externos:** otro módulo informa una situación por el bus. Por ejemplo, Movilidad publica `com.citypass.movilidad.EmergenciaBicicleta`. Entra por el listener de [Consumir eventos externos con un listener y reglas por evento](consumir-eventos-externos-con-un-listener-y-reglas-por-evento.md).

El tipo de emergencia no alcanza. En el caso de la bici, el grupo acordó el 2026-10-15 que la respuesta depende de las condiciones: si la persona no puede caminar y sangra, hacen falta bomberos y ambulancia. Movilidad las manda en `triage` (`puedeMoverse`, `estaSangrando`, `vehiculoInvolucrado`).

Además, el CU-05 dice que el dispatcher confirma que la emergencia es real antes de mover recursos. Una falsa alarma no puede pedir una ambulancia.

## Decisión

- Vamos a separar **disparadores** de **recursos** con una **tabla de reglas** en código: una función pura que recibe el tipo de emergencia y las condiciones, y devuelve los recursos. Ejemplos ilustrativos: `INCENDIO → [BOMBEROS]`, `MEDICA → [AMBULANCIA]`, y para la bici, "no puede moverse y sangra" `→ [BOMBEROS, AMBULANCIA]`. La tabla la arma Lucca.
- Vamos a **calcular al crear y publicar al validar**. Al crear la emergencia, guardamos los recursos sugeridos y el dispatcher los ve en el tablero. Al pasar a `VALIDATED`, se publican en el evento de cambio de estado, en la misma transacción y por el outbox. Si el dispatcher descarta la emergencia, no se publica nada.
- Vamos a que un **disparador externo siempre cree una emergencia propia**, por el mismo servicio que el flujo interno. Así el dispatcher la ve y la valida, y el punto anterior vale igual para los dos flujos.
- Vamos a que la regla del listener **solo traduzca campos**: pasa el tipo y las condiciones del evento, y no decide recursos. La decisión está en un solo lugar. Si Movilidad no recibió respuesta del triage (`respondido = false`), se aplica lo que corresponde al tipo.
- Vamos a **no usar un Template Method ni una factory por recurso**. Con un solo campo, los tres templates cambiarían únicamente un string. Se justificaría si cada recurso tuviera su evento y sus campos propios, y hoy no los tiene. El Template Method del listener es otra cosa y no cambia.
- Vamos a que la tabla de reglas sea un **módulo puro**: no conoce Postgres ni Kafka.

## Alternativas consideradas

| Opción | Pros | Contras |
|---|---|---|
| **Tabla de reglas + cálculo al crear + publicación al validar** | Una sola decisión para los dos flujos. Respeta el CU-05. Reutiliza el outbox. Fácil de explicar y de testear. | Hay que guardar los recursos sugeridos. Cambiar una regla requiere deploy. |
| Reglas + Template por recurso + factory al outbox (propuesta original) | Cada recurso crece con su payload propio. | Sin un evento por recurso, es una clase por string. Tres conceptos más para el equipo, sin un consumidor que los pida. |
| Publicar al crear, sin esperar la validación | Atómico con la creación. Más simple. | Una falsa alarma pediría una ambulancia. Contradice el CU-05. |
| Un handler por caso de uso | Cada caso queda autocontenido. | Un segundo caso que pide Bomberos duplica la decisión. |
| Todo entra por el listener, incluido nuestro `EmergenciaCreada` | Un único punto de entrada. | La respuesta interna depende de que el bus esté disponible y no funciona con el bus apagado. |
| Reglas en base de datos editables desde el tablero | Producto cambia reglas sin deploy. | Una pantalla y validaciones más. Reglas sin versionado ni tests. No se justifica con tres recursos. |

## Consecuencias

Se hace más fácil:

- Sumar un disparador o una condición: se agrega una fila a la tabla, sin tocar los flujos.
- Testear sin infraestructura: la tabla es una función pura.
- Que el dispatcher vea qué recursos se sugieren antes de confirmar.
- Mostrar la respuesta en la demo, aunque ningún otro módulo consuma el campo.

Se hace más difícil:

- Cambiar una regla requiere deploy.
- Toda emergencia externa pasa por validación del dispatcher, incluso si el otro módulo ya hizo un triage.
- Si aparece un consumidor por recurso, hay que revisar esta decisión junto con la del campo.

Pendiente de definir:

- El catálogo inicial de reglas, con Producto.
- Dónde se guardan los recursos sugeridos (hay que sumar una columna en `emergencies`).
- Si `triage.prioridadSugerida` de Movilidad define la severidad inicial de la emergencia.

## Historial

- 2026-10-16: creación del ADR (estado: propuesta)
- 2026-11-02: se aplica el review del 2026-10-18: los recursos se calculan al crear y se publican al validar, las reglas reciben tipo y condiciones, y un disparador externo siempre crea una emergencia propia. Con el campo del ADR de recursos, se saca el Template Method y la factory por recurso. Cambia el título.
- 2026-11-05: cambio de estado: propuesta → aceptada
