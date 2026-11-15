# Vincular los oficiales del LDAP al catálogo de oficiales

- Estado: propuesta
- Fecha: 2026-11-15
- Decisores: (a completar: Producto, Security y el delegado del módulo)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Desde el 12/11/2026 el backend corre en `AUTH_MODE=hybrid`: el personal
entra con el token del Login Federado del Grupo 2 (RS256, verificado
contra su JWKS) y los ciudadanos con cuenta local. El dispatcher ya entra
por LDAP. El 15/11/2026 se dio de alta el primer oficial en el LDAP
(`of.gorgory`) con el grupo `poli`, que el backend y el front ya traducen
a `officer`.

Falta resolver **qué oficial del catálogo es esa persona**. Hoy:

- El catálogo es la tabla `officers` (V11): diez oficiales demo
  (`of-001`…`of-010`) con nombre, chapa, ubicación y estado
  (`disponible`/`ocupado`). El dispatcher asigna desde ahí y
  `emergencies.assigned_officer_id` guarda ese id.
- Las acciones del oficial (`en-route`, `on-scene`, `resolve`) exigen que
  el `sub` del token sea `assigned_officer_id`.
- El token del LDAP trae `sub` (`U000xxx`, opaco), `preferred_username`
  (`of.gorgory`) y `groups`. **No** trae nombre completo ni chapa.
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
   sus casos a otro oficial, en cualquier estado en curso. Un caso no
   puede quedar trabado en manos de alguien que ya no es oficial.
4. **No hay ni habrá este año un canal de bajas.** Nos enteramos de quién
   es alguien solo cuando nos manda un token. Alguien que salió del LDAP
   simplemente deja de mandarlos, y eso no se distingue de un oficial que
   no está de turno. El access token dura 15 minutos y no hay revocación.
   El 15/11/2026 el Grupo 2 confirmó que **este año no va a publicar un
   evento de baja ni un endpoint que liste los usuarios de `poli`**.
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
| Identificador estable (`sub`) y usuario (`preferred_username`) | LDAP; nosotros solo guardamos el vínculo |
| Chapa, estado operativo, ubicación, asignaciones e historial | Tabla `officers` (único catálogo) |

### 1. Vínculo en la tabla `officers`

Una migración agrega a `officers`:

- `idp_sub TEXT UNIQUE` (nulo para los oficiales demo locales);
- `ldap_username TEXT UNIQUE` (para reservar un oficial existente a un
  usuario del LDAP);
- `active BOOLEAN NOT NULL DEFAULT true` y `deactivated_at TIMESTAMPTZ`;
- `last_seen_at TIMESTAMPTZ`;
- `badge_number` pasa a ser opcional (el token no la trae).

`IDP_SUB_ALIASES` se elimina: el vínculo vive en la base.

### 2. Alta automática al primer request (just-in-time)

Cuando llega un token federado con grupo de oficial, `authenticate`
resuelve el oficial en este orden:

1. **Ya vinculado:** fila con `idp_sub` = `sub` del token.
2. **Reservado:** fila con `ldap_username` = `preferred_username` y
   `idp_sub` nulo → se vincula (se completa `idp_sub`).
3. **Nuevo:** se crea una fila con el siguiente id libre (`of-011`…),
   nombre = `preferred_username` (o el claim `name` si el Grupo 2 lo
   agrega), sin chapa y en `disponible`.

La resolución es un único `INSERT … ON CONFLICT (idp_sub)` dentro de una
transacción, así dos logins simultáneos no crean dos filas. El resultado
se cachea en memoria por proceso con un TTL corto. Desde ahí el request
opera con el id del catálogo (`of-0XX`), igual que hoy con el alias: el
chequeo de oficial asignado, el `actor` del historial, los eventos y el
rate limit no cambian. Se actualiza `last_seen_at`.

**El vínculo solo usa identificadores exactos** (`sub` o un
`ldap_username` reservado a propósito). No se vincula por nombre.

### 3. Bajas: el acceso lo corta el LDAP, el catálogo lo marcamos nosotros

Sin señal del Grupo 2, separamos dos cosas:

- **El acceso se corta solo.** Quien sale del LDAP ya no consigue tokens
  nuevos; el que tiene vence en 15 minutos como máximo. No dependemos de
  enterarnos para eso.
- **El catálogo se entera por dos vías:**
  1. **Login sin `poli`:** si un `sub` vinculado llega con un token que ya
     no trae el grupo de oficial (lo sacaron de `poli` pero sigue en el
     LDAP), se lo da de baja en ese momento. Es automático, pero no ve a
     quien fue borrado del LDAP, porque esa persona ya no puede entrar.
  2. **Baja manual del dispatcher:** en `/ops/oficiales`, el dispatcher
     marca al oficial como dado de baja
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
| `EN_ROUTE`, `ON_SCENE` | Reasignar: vuelve a `DISPATCHED` con el oficial nuevo, que arranca desde "en camino". Es una **transición nueva**, solo para el dispatcher y solo con el oficial asignado inactivo. |
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
- **No se hashea el vínculo** (`idp_sub`, `ldap_username`): ver
  alternativas descartadas.

## Alternativas consideradas

### Cómo vincular la persona del LDAP con el catálogo

| Alternativa | Decisión | Motivo |
|---|---|---|
| Vínculo en `officers` + alta just-in-time (elegida) | **Se hace** | Una sola tabla, sin pasos manuales, por identificador exacto. |
| Mantener `IDP_SUB_ALIASES` en Render | Descartada | Es una segunda lista que se mantiene a mano y exige un redeploy por cada oficial nuevo. |
| Vincular por nombre (fuzzy o exacto) | Descartada | Homónimos y variantes de acentos; un usuario creado con el nombre de otro oficial se quedaría con sus casos e historial. A lo sumo podría ser una sugerencia que confirma una persona; no se implementa ahora. |
| Usar el `sub` del LDAP como id del oficial (`assigned_officer_id = U000xxx`) | Descartada | Evita la columna de vínculo, pero no permite vincular a un oficial existente, mezcla dos formatos de id en el catálogo y en los eventos, y el espacio de `sub` es compartido con las cuentas locales (`U000042` también es un sub demo). |
| Id derivado del `sub` con un hash (`of-<hash(sub)>`) | Descartada | Sigue sin permitir vincular a uno existente y no ayuda a detectar bajas; solo cambia el formato del id. |
| Consultar el LDAP directamente desde el backend | Descartada | El LDAP es del Grupo 2 y no está expuesto; el contrato es el token y, a lo sumo, una API suya. |

### Cómo enterarnos de una baja

| Alternativa | Decisión | Motivo |
|---|---|---|
| Detectar el login sin `poli` | **Se hace** | Automático y sin depender de nadie, aunque solo ve a quien sigue en el LDAP. |
| Baja manual del dispatcher | **Se hace** | Cubre a quien fue borrado del LDAP, que nunca vuelve a mandarnos un token. |
| Evento del Grupo 2 en el bus | Descartada | El Grupo 2 no lo va a hacer este año (15/11/2026). Sería la vía preferida si cambia. |
| Reconciliación periódica contra una API del Grupo 2 | Descartada | El Grupo 2 no va a exponer el listado este año (15/11/2026). |
| Dar de baja por inactividad (N horas sin entrar) | Descartada | No distingue a alguien dado de baja de alguien que no está de turno: le cortaría el acceso a oficiales activos. `last_seen_at` queda solo como dato informativo. |
| Introspección del token en cada request | Descartada | Tampoco existe un endpoint del Grupo 2 para eso, y agregaría un llamado al IdP por request para achicar una ventana de 15 minutos. |
| Aceptar que el token venza solo, sin baja en el catálogo | Descartada | El oficial seguiría en el catálogo asignable y sus casos quedarían colgados. |

### Qué hacer con los casos del oficial dado de baja

| Alternativa | Decisión | Motivo |
|---|---|---|
| Marcar los casos y que el dispatcher los reasigne en un paso, en `DISPATCHED`, `EN_ROUTE` u `ON_SCENE` (elegida) | **Se hace** | Una persona decide con contexto (puede haber alguien en el lugar); una baja errónea del LDAP no desarma la operación en curso; la reasignación queda auditada con el dispatcher como actor. |
| Devolver todo a `VALIDATED` automáticamente, con actor `system` | Descartada | Una baja errónea desasignaría todos los casos en curso de golpe, y un caso `ON_SCENE` volvería a la cola con alguien todavía en el lugar. |
| Devolver solo `DISPATCHED` automáticamente | Descartada | Los casos en curso quedarían con un oficial que ya no puede actualizarlos y sin forma de reasignarlos. |
| Reasignar en dos pasos (devolver a `VALIDATED` y asignar) | Descartada para `EN_ROUTE`/`ON_SCENE` | Hoy la devolución solo existe desde `DISPATCHED`; abrirla desde otros estados para cualquier caso es más amplio que lo necesario. Para `DISPATCHED` sigue disponible. |
| Cancelar los casos | Descartada | La emergencia sigue existiendo: hay que reasignarla, no cerrarla. |

### Criptografía

| Alternativa | Decisión | Motivo |
|---|---|---|
| Verificar RS256 contra el JWKS (ya existe) | **Se mantiene** | Garantiza la autenticidad del grupo. |
| Guardar `idp_sub`/`ldap_username` como HMAC con clave | Descartada por ahora | El `sub` ya es opaco y el catálogo guarda el nombre en claro; el hash impide leer el vínculo para depurar. Se revisa si Security clasifica estos identificadores como PII (se haría con una clave gestionada como `LOCATION_ENCRYPTION_KEY`). |
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
- Reasignar en un paso un caso trabado, en cualquier estado en curso.
- Auditar: el historial dice qué dispatcher reasignó, desde qué oficial
  y por qué.

Se hace más difícil o cuesta:

- El catálogo no se entera solo de quien fue borrado del LDAP: hasta que
  el dispatcher lo marca, ese oficial sigue figurando como asignable y
  sus casos no aparecen marcados.
- El dispatcher suma una tarea: dar de baja oficiales en `/ops/oficiales`.
- Un oficial aparece en el catálogo recién después de su primer login: el
  dispatcher no puede asignar a alguien que nunca entró.
- Sin el claim `name`, el catálogo muestra el usuario (`of.gorgory`) en
  vez del nombre; sin chapa, la columna queda vacía.
- Cambia la máquina de estados (transición nueva `EN_ROUTE`/`ON_SCENE` →
  `DISPATCHED`, solo para el dispatcher y solo con el oficial asignado
  inactivo) y, con ella, lo que ven los consumidores de
  `EmergenciaActualizada`.
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
  Mitigación a evaluar: alerta si un caso marcado sigue sin reasignar
  pasados N minutos.
- **Oficial fantasma:** alguien borrado del LDAP sigue `disponible` en el
  catálogo hasta la baja manual. El dispatcher puede asignarle un caso
  nuevo que nadie va a atender. Mitigación a evaluar: mostrar
  `last_seen_at` en el catálogo y en el modal de asignar, y avisar si el
  oficial no aparece hace más de un turno (solo aviso, sin baja
  automática).
- **Baja masiva por error del LDAP:** si alguien quita por error el grupo
  `poli`, todos los oficiales pierden el acceso a la vez y, a medida que
  entran, se van marcando. Los casos no se mueven, pero nadie puede
  actualizarlos hasta que el LDAP se corrija (y se reactivan solos al
  volver a entrar) o el dispatcher reasigne.
- **Baja manual equivocada:** el dispatcher marca al oficial equivocado.
  Queda auditado y se revierte a mano, pero mientras tanto esa persona
  recibe 403.
- **Oficial en el lugar sin acceso:** un oficial `ON_SCENE` dado de baja
  sigue físicamente en el lugar pero ya no puede marcar "resuelta". El
  dispatcher tiene que coordinar por otra vía antes de reasignar, para no
  mandar a un segundo oficial sin saberlo.
- **Consumidores del bus que no esperan ir hacia atrás:** otros módulos
  pueden asumir que después de `EN_CAMINO` no viene `DESPACHADA`. Hay que
  avisarlo como cambio de contrato.
- **Reasignar a quien vuelve:** si el LDAP lo reactiva justo mientras el
  dispatcher reasigna, la acción tiene que fallar limpia (409: el oficial
  asignado ya está activo) en vez de pisar el caso.
- **Reutilización de usuarios en el LDAP:** si el Grupo 2 reasigna un
  `preferred_username` a otra persona, una reserva por `ldap_username`
  sin usar quedaría para la persona equivocada. Por eso, una vez
  vinculada, la fila se busca por `idp_sub` y la reserva no se reutiliza.
  Si el Grupo 2 reutilizara un `sub`, el vínculo sería incorrecto: hay que
  confirmar que no lo hace.
- **Caché desactualizada con varias instancias:** con más de un proceso,
  uno puede seguir aceptando a un oficial ya dado de baja hasta que vence
  su caché. Hoy hay una sola instancia en Render; el TTL acota la ventana.
- **Catálogo que crece:** cada `poli` que entra una vez queda en la tabla.
  Los dados de baja no se borran (por el historial), solo se ocultan.
- **Ventana de 15 minutos** después de la baja en el LDAP: el token viejo
  sigue sirviendo hasta que vence, salvo que el dispatcher haga la baja
  manual antes.
- **Experiencia del oficial dado de baja:** pasa de golpe a recibir 403;
  el front tiene que mostrarlo y cerrarle la sesión en vez de reintentar.

### Pendientes para pasar a `aceptada`

1. Confirmar con el Grupo 2 que el refresh rechaza a los usuarios dados
   de baja y que no reutilizan `sub`, y pedir el claim `name` (y chapa si
   la tienen). El evento de baja y el listado quedan fuera por este año.
2. Confirmar con Producto la reasignación desde `EN_ROUTE`/`ON_SCENE` y
   cuánto tiempo puede quedar un caso marcado antes de alertar.
3. Confirmar con Security que el vínculo puede guardarse en claro.
4. Definir si `of.gorgory` es un oficial nuevo o se reserva uno de los
   existentes con `ldap_username`.

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
