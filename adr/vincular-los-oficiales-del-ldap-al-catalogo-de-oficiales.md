# Vincular los oficiales del LDAP al catálogo de oficiales

- Estado: pospuesta
- Fecha: 2026-11-15
- Decisores: @Salterm27 (la propuso) y Producto (@juanimoli), que la revisó. Para aceptarla faltan Security y el delegado del módulo.
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Desde el 12/11/2026 el backend corre en `AUTH_MODE=hybrid`: el personal
entra con el token del Login Federado del Grupo 2 (RS256, verificado
contra su JWKS) y los ciudadanos con cuenta local. El dispatcher ya entra
por LDAP. El 15/11/2026 se dio de alta el primer oficial en el LDAP
(`jefe.gorgory`) con el grupo `poli`, que el backend y el front ya traducen
a `officer`.

Falta resolver **qué oficial del catálogo es esa persona**. Hoy:

- El catálogo es la tabla `officers` (V11): diez oficiales demo
  (`of-001`…`of-010`) con nombre, chapa, ubicación y estado
  (`disponible`/`ocupado`). El dispatcher asigna desde ahí y
  `emergencies.assigned_officer_id` guarda ese id.
- Las acciones del oficial (`en-route`, `on-scene`, `resolve`) exigen que
  el `sub` del token sea `assigned_officer_id`.
- El token del LDAP trae `sub`, `preferred_username`
  (`jefe.gorgory`) y `groups`. **No** trae nombre completo ni chapa.
- El `sub` es el `employeeNumber` del LDAP (formato `U000042`). Según
  el Grupo 2 (15/11/2026): lo genera el sistema, es único globalmente con
  unicidad forzada por el LDAP, y nunca se libera, porque las bajas son
  deshabilitaciones y no borrados. El Grupo 2 pide usar el `sub` como
  clave de la persona, **no** el usuario ni el mail.
- El puente actual es la variable `IDP_SUB_ALIASES` en Render
  (`{"U000xxx":"of-001"}`), que cambia el `sub` federado por el id del
  catálogo antes de autorizar.

Fuerzas en juego:

1. **Una sola fuente de verdad.** No queremos una lista de oficiales en el
   LDAP, otra en la base y una tercera en una variable de Render que haya
   que mantener a mano.
2. **Altas sin intervención.** Si alguien entra al LDAP con el grupo
   `poli`, tiene que quedar disponible como oficial sin tocar código ni
   configuración.
3. **Bajas efectivas.** Si alguien sale del LDAP (o del grupo `poli`),
   tiene que perder el acceso, y el dispatcher tiene que poder reasignar
   sus casos a otro oficial. Un caso no
   puede quedar trabado en manos de alguien que ya no es oficial.
4. **No hay ni habrá este año un canal de bajas.** Nos enteramos de quién
   es alguien solo cuando nos manda un token. Alguien que salió del LDAP
   simplemente deja de mandarlos, y eso no se distingue de un oficial que
   no está de turno. El access token dura 15 minutos y no hay revocación.
   El 15/11/2026 el Grupo 2 confirmó que **este año no va a publicar un
   evento de baja ni un endpoint que liste los usuarios de `poli`**.
   También confirmó cómo responde el refresh (rota el refresh token y
   relee a la persona en el LDAP con `reloadBySub`):
   - si la sacaron de un grupo, **renueva** y el token nuevo trae sus
     grupos actuales (`memberOf`), ya sin ese grupo;
   - **rechaza** si la cuenta fue borrada o deshabilitada
     (`pwdAccountLockedTime`), si cambió de módulo (OU) y el cliente ya no
     lo acepta, o si el refresh se reusó, venció o cambió la audience.
5. **La máquina de estados** solo permite volver a `VALIDATED` desde
   `DISPATCHED` (devolución del dispatcher). `EN_ROUTE` y `ON_SCENE` no
   tienen vuelta atrás.
6. **Seguridad.** Vincular por un dato que cualquiera puede repetir (el
   nombre) permitiría quedarse con la identidad y los casos de otro
   oficial.

## Decisión

Vamos a separar **identidad** y **operación**, con una fuente para cada
dato:

| Dato | Fuente de verdad |
|---|---|
| Quién es oficial (pertenencia a `poli`) | LDAP del Grupo 2 |
| Clave de la persona (`sub` = `employeeNumber`) | LDAP; nosotros solo guardamos el vínculo |
| Usuario (`preferred_username`) | LDAP; solo se muestra, nunca se usa para vincular |
| Chapa, estado operativo, ubicación, asignaciones e historial | Tabla `officers` (único catálogo) |

### 1. Vínculo en la tabla `officers`

Una migración agrega a `officers`:

- `idp_sub TEXT UNIQUE` (nulo para los oficiales demo locales): el `sub`
  del LDAP, única clave de vínculo;
- `active BOOLEAN NOT NULL DEFAULT true` y `deactivated_at TIMESTAMPTZ`;
- `last_seen_at TIMESTAMPTZ`;
- `badge_number` de los oficiales del LDAP es su `sub` (`U000042`): el
  token no trae chapa y el `employeeNumber` ya es el número único de la
  persona. Los oficiales demo locales conservan la suya.

`IDP_SUB_ALIASES` se elimina: el vínculo vive en la base.

### 2. Alta automática al primer request (just-in-time)

Cuando llega un token federado con grupo de oficial, `authenticate`
resuelve el oficial en este orden:

1. **Ya vinculado:** fila con `idp_sub` = `sub` del token.
2. **Nuevo:** se crea una fila con el siguiente id libre (`of-011`…),
   nombre = `preferred_username` (o el claim `name` si el Grupo 2 lo
   agrega), `badge_number` = `sub` y en `disponible`.

La resolución es un único `INSERT … ON CONFLICT (idp_sub)` dentro de una
transacción, así dos logins simultáneos no crean dos filas. El resultado
se cachea en memoria por proceso con un TTL corto. Desde ahí el request
opera con el id del catálogo (`of-0XX`), igual que hoy con el alias: el
chequeo de oficial asignado, el `actor` del historial, los eventos y el
rate limit no cambian. Se actualiza `last_seen_at`.

**El vínculo usa solo el `sub`.** No se vincula por nombre, usuario ni
mail. Para que una persona del LDAP sea un oficial que **ya existe** en
el catálogo, se le carga su `sub` en `idp_sub` antes de su primer login
(migración o acción del dispatcher) y su `badge_number` pasa a ser ese
`sub`; conserva su id, su nombre y su historial, y al entrar cae en el
paso 1. El `sub` se ve en el panel del Login Federado (es el
`employeeNumber`).

**Primer vínculo:** `jefe.gorgory` (`sub` `U000025`, grupo `poli`) se
vincula a `of-001` (Oficial Carlos Gómez), el oficial con más casos
asignados (9 de 182 emergencias al 15/11/2026, todas resueltas o
cerradas). Hasta implementar este ADR, el vínculo va por
`IDP_SUB_ALIASES={"U000025":"of-001"}` en Render; al implementarlo, la
migración precarga `idp_sub = 'U000025'` y `badge_number = 'U000025'` en
`of-001`.

### 3. Bajas: el acceso lo corta el LDAP, el catálogo lo marcamos nosotros

Sin señal del Grupo 2, separamos dos cosas:

- **El acceso se corta solo.** Quien sale del LDAP ya no consigue tokens
  nuevos; el que tiene vence en 15 minutos como máximo. No dependemos de
  enterarnos para eso.
- **El catálogo se entera por dos vías:**
  1. **Token sin `poli`:** como el refresh relee los grupos en el LDAP,
     a quien sacan de `poli` (y sigue en el LDAP) le llega, a los 15
     minutos como máximo, un token sin ese grupo. Si un `sub` vinculado a
     un oficial llega con un token sin grupo de oficial, se lo da de baja
     en ese momento. Por eso el chequeo corre en **todo** token federado
     cuyo `sub` esté vinculado, traiga o no grupos de oficial (sin `poli`,
     el front lo manda a las pantallas de ciudadano y esos requests son
     los que lo detectan). Si la persona no tiene la app abierta, se
     detecta la próxima vez que entre; mientras tanto ya no tiene acceso
     de oficial. No ve a quien fue borrado, deshabilitado o movido de
     módulo: a esa persona el refresh la rechaza y no nos vuelve a
     mandar un token.
  2. **Baja manual del dispatcher** (para cuentas borradas,
     deshabilitadas o movidas de módulo): en `/ops/oficiales`, el
     dispatcher marca al oficial como dado de baja
     (`PATCH /api/officers/:id/deactivate`, con motivo). Es la vía para
     quien fue borrado del LDAP o para cuando el dispatcher se entera
     por otro canal (el oficial no responde, aviso de la jefatura).

### 4. Efecto de la baja: la reasignación la decide el dispatcher

La baja (automática o manual) **no mueve los casos sola**. Marca al
oficial y deja que el dispatcher decida a quién reasignar:

1. El oficial queda `active = false` y sale del catálogo asignable de
   `GET /api/officers`.
2. `authenticate` lo rechaza (403) aunque su token siga vigente, porque
   consulta `active` en el catálogo. Así una baja manual corta también
   la ventana de 15 minutos del token.
3. Sus emergencias en curso **quedan como están**, con su
   `assigned_officer_id`. La API las devuelve marcadas
   (`assignedOfficerActive: false`) y el tablero del dispatcher las
   destaca como "oficial dado de baja, reasignar".
4. El dispatcher las **reasigna en un paso** a otro oficial activo con
   `PATCH /api/emergencies/:id/reassign`. Esa acción solo se permite si
   el oficial asignado está inactivo.

Por estado:

| Estado | Qué puede hacer el dispatcher |
|---|---|
| `DISPATCHED` | Reasignar (queda `DISPATCHED` con el oficial nuevo). También sirve el camino actual: devolver a `VALIDATED` y asignar. |
| `EN_ROUTE`, `ON_SCENE` | **Fuera de alcance de este ADR.** Quedan marcadas pero no se pueden reasignar todavía; se resuelve en un issue aparte (ver Pendientes). |
| `RESOLVED` | Nada que reasignar: el trabajo está hecho y el dispatcher la cierra. |

La reasignación es una transacción: cambia `assigned_officer_id`, deja la
fila en `emergency_status_history` (actor = dispatcher, motivo
`reasignada: oficial dado de baja en LDAP`, con el oficial anterior y el
nuevo), marca `ocupado` al oficial nuevo y emite `EmergenciaActualizada`
en el outbox, como cualquier transición.

Reactivación: si la baja fue **automática** (login sin `poli`) y la
persona vuelve al grupo, al entrar se reactiva la misma fila (mismo
`idp_sub`) y recupera los casos que todavía no se reasignaron. Si la baja
fue **manual**, entrar con `poli` no la deshace: la revierte el
dispatcher, para que una decisión explícita no se pise sola.

### 5. Criptografía

- **Se mantiene** la verificación RS256 del token contra el JWKS del
  Grupo 2: es lo que garantiza que el grupo `poli` es auténtico.
- **La baja manual** queda protegida como cualquier acción del
  dispatcher: token verificado, grupo `dispatcher` y registro de quién y
  por qué.
- **El `sub` se guarda en claro** (`idp_sub` y `badge_number`): ver
  alternativas descartadas.

## Alternativas consideradas

### Cómo vincular la persona del LDAP con el catálogo

| Alternativa | Decisión | Motivo |
|---|---|---|
| Vínculo en `officers` + alta just-in-time (elegida) | **Se hace** | Una sola tabla, sin pasos manuales, por identificador exacto. |
| Mantener `IDP_SUB_ALIASES` en Render | Descartada | Es una segunda lista que se mantiene a mano y exige un redeploy por cada oficial nuevo. |
| Vincular o reservar por usuario (`preferred_username`) o mail | Descartada | El Grupo 2 pide no usarlos como clave: pueden cambiar, y un usuario reasignado a otra persona heredaría una reserva ajena. Para vincular a un oficial existente se precarga su `sub`. |
| Vincular por nombre (fuzzy o exacto) | Descartada | Homónimos y variantes de acentos; un usuario creado con el nombre de otro oficial se quedaría con sus casos e historial. A lo sumo podría ser una sugerencia que confirma una persona; no se implementa ahora. |
| Usar el `sub` del LDAP como id del oficial (`assigned_officer_id = U000xxx`) | Descartada | Evita la columna de vínculo, pero no permite vincular a un oficial existente, mezcla dos formatos de id en el catálogo y en los eventos, y el espacio de `sub` es compartido con las cuentas locales (`U000042` también es un sub demo). |
| Id derivado del `sub` con un hash (`of-<hash(sub)>`) | Descartada | Sigue sin permitir vincular a uno existente y no ayuda a detectar bajas; solo cambia el formato del id. |
| Consultar el LDAP directamente desde el backend | Descartada | El LDAP es del Grupo 2 y no está expuesto; el contrato es el token y, a lo sumo, una API suya. |

### Cómo enterarnos de una baja

| Alternativa | Decisión | Motivo |
|---|---|---|
| Detectar el token sin `poli` | **Se hace** | Automático: como el refresh relee los grupos, una salida de `poli` llega en 15 minutos como máximo con la sesión abierta. Solo ve a quien sigue en el LDAP. |
| Baja manual del dispatcher | **Se hace** | Cubre a quien fue borrado, deshabilitado o movido de módulo: el refresh lo rechaza y nunca vuelve a mandarnos un token. |
| Evento del Grupo 2 en el bus | Descartada | El Grupo 2 no lo va a hacer este año (15/11/2026). Sería la vía preferida si cambia. |
| Reconciliación periódica contra una API del Grupo 2 | Descartada | El Grupo 2 no va a exponer el listado este año (15/11/2026). |
| Dar de baja por inactividad (N horas sin entrar) | Descartada | No distingue a alguien dado de baja de alguien que no está de turno: le cortaría el acceso a oficiales activos. `last_seen_at` queda solo como dato informativo. |
| Introspección del token en cada request | Descartada | Tampoco existe un endpoint del Grupo 2 para eso, y agregaría un llamado al IdP por request para achicar una ventana de 15 minutos. |
| Aceptar que el token venza solo, sin baja en el catálogo | Descartada | El oficial seguiría en el catálogo asignable y sus casos quedarían colgados. |

### Qué hacer con los casos del oficial dado de baja

| Alternativa | Decisión | Motivo |
|---|---|---|
| Marcar los casos y que el dispatcher reasigne en un paso los `DISPATCHED` (elegida) | **Se hace** | Una persona decide con contexto (puede haber alguien en el lugar); una baja errónea del LDAP no desarma la operación en curso; la reasignación queda auditada con el dispatcher como actor. |
| Devolver todo a `VALIDATED` automáticamente, con actor `system` | Descartada | Una baja errónea desasignaría todos los casos en curso de golpe, y un caso `ON_SCENE` volvería a la cola con alguien todavía en el lugar. |
| Devolver solo `DISPATCHED` automáticamente | Descartada | Los casos en curso quedarían con un oficial que ya no puede actualizarlos y sin forma de reasignarlos. |
| Reasignar también desde `EN_ROUTE`/`ON_SCENE` | Postergada | Cambia la máquina de estados y el contrato del bus; se trata en un issue aparte. |
| Reasignar en dos pasos (devolver a `VALIDATED` y asignar) | Se mantiene para `DISPATCHED` | Es el camino que ya existe; la reasignación en un paso lo complementa. |
| Cancelar los casos | Descartada | La emergencia sigue existiendo: hay que reasignarla, no cerrarla. |

### Criptografía

| Alternativa | Decisión | Motivo |
|---|---|---|
| Verificar RS256 contra el JWKS (ya existe) | **Se mantiene** | Garantiza la autenticidad del grupo. |
| Guardar `idp_sub` como HMAC con clave | Descartada | Se decidió guardar el `sub` en claro (15/11/2026): es un número opaco, además es la chapa visible, y el hash impediría leer el vínculo para depurar. |
| Cifrar el vínculo con AES | Descartada | Para buscar por el valor haría falta además un índice ciego (HMAC); mismo costo, sin beneficio hoy. |

## Consecuencias

### Esperadas

Se hace más fácil:

- Dar de alta un oficial: alcanza con el grupo `poli` en el LDAP. No hay
  que tocar Render, código ni la base.
- Saber quién es cada oficial: el catálogo es el único lugar, y el LDAP
  manda sobre quién pertenece.
- Cortar el acceso de quien sale del LDAP sin depender de otro grupo:
  lo corta el propio LDAP, y la baja manual lo corta en el acto.
- No mover casos por sorpresa.
- Reasignar en un paso un caso `DISPATCHED` de un oficial dado de baja.
- Auditar: el historial dice qué dispatcher reasignó, desde qué oficial
  y por qué.

Se hace más difícil o cuesta:

- El catálogo no se entera solo de quien fue borrado del LDAP: hasta que
  el dispatcher lo marca, ese oficial sigue figurando como asignable y
  sus casos no aparecen marcados.
- El dispatcher suma una tarea: dar de baja oficiales en `/ops/oficiales`.
- Un oficial aparece en el catálogo recién después de su primer login: el
  dispatcher no puede asignar a alguien que nunca entró.
- Sin el claim `name`, un oficial nuevo figura con su usuario
  (`jefe.gorgory`) en vez del nombre. La chapa muestra el `sub`, que no es
  un número de chapa policial real.
- La máquina de estados no cambia en este ADR: la reasignación cubre
  solo `DISPATCHED`. Los casos `EN_ROUTE`/`ON_SCENE` de un oficial dado
  de baja quedan marcados y trabados hasta que se resuelva el issue
  aparte.
- Los casos del oficial dado de baja quedan quietos hasta que el
  dispatcher actúa: la reasignación depende de que alguien mire el
  tablero.
- El front suma la marca en el tablero y la acción de reasignar.
- `authenticate` pasa a consultar la base para los oficiales federados
  (mitigado con caché).

Neutras:

- Los oficiales demo (`of-001`…`of-010`) siguen igual para las cuentas
  locales, en la misma tabla.
- El front sigue usando `GET /api/officers` y el id `of-0XX`; solo suma
  la marca, la baja manual y la reasignación.

### No esperadas (riesgos a vigilar)

- **Casos olvidados:** si nadie mira el tablero, un caso puede quedar
  con un oficial sin acceso durante mucho tiempo, sin que nadie avance.
  Por ahora no hay alerta (decisión del 16/11/2026): el caso solo queda
  marcado en el tablero.
- **Oficial fantasma:** alguien borrado, deshabilitado o movido de
  módulo en el LDAP sigue `disponible` en el catálogo hasta la baja
  manual (a quien solo sacan de `poli` lo detecta el token). El dispatcher puede asignarle un caso
  nuevo que nadie va a atender. Mitigación a evaluar: mostrar
  `last_seen_at` en el catálogo y en el modal de asignar, y avisar si el
  oficial no aparece hace más de un turno (solo aviso, sin baja
  automática).
- **Baja masiva por error del LDAP:** si alguien quita por error el grupo
  `poli`, en 15 minutos todos los oficiales con sesión abierta reciben un
  token sin el grupo y se van marcando. Los casos no se mueven, pero nadie puede
  actualizarlos hasta que el LDAP se corrija (y se reactivan solos al
  volver a entrar) o el dispatcher reasigne.
- **Baja manual equivocada:** el dispatcher marca al oficial equivocado.
  Queda auditado y se revierte a mano, pero mientras tanto esa persona
  recibe 403.
- **Casos en curso trabados:** un oficial `EN_ROUTE` u `ON_SCENE` dado de
  baja ya no puede avanzar el caso y el dispatcher todavía no puede
  reasignarlo. Hasta el issue aparte, se coordina por fuera del sistema.
- **Cuenta demo y persona real comparten oficial:** `of-001` también es
  una cuenta demo local (atajo de login). Con el vínculo, `jefe.gorgory` y
  ese atajo operan como el mismo oficial. Los logins demo quedan fuera de
  este ADR y se tratan en un issue aparte.
- **Reasignar a quien vuelve:** si el LDAP lo reactiva justo mientras el
  dispatcher reasigna, la acción tiene que fallar limpia (409: el oficial
  asignado ya está activo) en vez de pisar el caso.
- **`sub` precargado mal:** si al vincular un oficial existente se carga
  el `sub` de otra persona, esa persona entra como ese oficial, con sus
  casos e historial. Hay que copiarlo del panel del Login Federado y
  dejar auditado quién lo cargó. (La reutilización de `sub` no es un
  riesgo: el Grupo 2 garantiza que nunca se libera.)
- **Caché desactualizada con varias instancias:** con más de un proceso,
  uno puede seguir aceptando a un oficial ya dado de baja hasta que vence
  su caché. Hoy hay una sola instancia en Render; el TTL acota la ventana.
- **Catálogo que crece:** cada `poli` que entra una vez queda en la tabla.
  Los dados de baja no se borran (por el historial), solo se ocultan.
- **Ventana de 15 minutos** después de la baja en el LDAP: el access
  token viejo sigue sirviendo hasta que vence (el refresh ya trae los
  grupos nuevos o se rechaza), salvo que el dispatcher haga la baja
  manual antes.
- **Experiencia del oficial dado de baja:** pasa de golpe a recibir 403;
  el front tiene que mostrarlo y cerrarle la sesión en vez de reintentar.

### Pendientes para pasar a `aceptada`

Ninguno. Queda para la revisión del PR.

Postergado: pedir al Grupo 2 el claim `name` (mientras tanto, un oficial
nuevo figura con su usuario).

Fuera de este ADR, en issues aparte:

- reasignar casos `EN_ROUTE`/`ON_SCENE` de un oficial dado de baja;
- los logins demo (atajos locales como `of-001`) frente a los oficiales
  del LDAP.

## Relacionados

- [Usar identidad local con JWT y sacar la PII del navegador](usar-identidad-local-con-jwt.md)
- [Autorizar por JWT con ownership y endurecer el perímetro HTTP](autorizar-por-jwt-con-ownership-y-endurecer-el-perimetro-http.md)
- [Cifrar ubicación y PII sensible en reposo](cifrar-ubicacion-y-pii-sensible-en-reposo.md)

## Historial

- 2026-11-15: creación del ADR (estado: propuesta)
- 2026-11-15: la baja deja de desasignar los casos automáticamente; los
  marca y el dispatcher los reasigna en un paso (también desde
  `EN_ROUTE`/`ON_SCENE`).
- 2026-11-15: el Grupo 2 confirma que este año no publica evento de baja
  ni listado de `poli`. Se descartan esas dos vías y se suma la baja
  manual del dispatcher.
- 2026-11-15: el Grupo 2 confirma que el refresh relee los grupos en el
  LDAP y rechaza cuentas borradas, deshabilitadas o movidas de módulo. La
  salida de `poli` pasa a detectarse sola en 15 minutos; la baja manual
  queda para los otros casos.
- 2026-11-15: el Grupo 2 confirma que el `sub` es el `employeeNumber`,
  único y nunca reutilizado, y pide usarlo como clave. Se elimina la
  reserva por usuario (`ldap_username`): un oficial existente se vincula
  precargando su `sub`.
- 2026-11-15: decisiones: la chapa de los oficiales del LDAP es su `sub`;
  el `sub` se guarda en claro; la reasignación desde `EN_ROUTE`/`ON_SCENE`
  sale de este ADR y va a un issue aparte; `jefe.gorgory` se vincula a
  `of-001`, el oficial con más casos asignados.
- 2026-11-15: el claim `name` se posterga y los logins demo pasan a un
  issue aparte.
- 2026-11-16: el login de `jefe.gorgory` funciona (`sub` `U000025`); se
  vincula a `of-001` por `IDP_SUB_ALIASES` hasta implementar el ADR.
- 2026-11-16: no se agrega alerta por casos marcados sin reasignar; solo
  la marca en el tablero.
- 2026-11-17: cambio de estado: propuesta → pospuesta (Producto, @juanimoli. En producción corre el puente por alias (`IDP_SUB_ALIASES`), documentado en el ADR del Login Federado; la tabla de vínculo queda para después del Hito 2.)
