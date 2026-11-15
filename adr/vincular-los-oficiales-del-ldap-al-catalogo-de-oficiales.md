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
   tiene que perder el acceso y sus casos tienen que volver a quedar sin
   asignar, para que el dispatcher los reasigne. Un caso no puede quedar
   en manos de alguien que ya no es oficial.
4. **No hay canal de bajas hoy.** Nos enteramos de quién es alguien solo
   cuando nos manda un token. Alguien que salió del LDAP simplemente deja
   de mandarlos, y eso no se distingue de un oficial que no está de turno.
   El access token dura 15 minutos y no hay revocación.
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

### 3. Detección de bajas

Se combinan tres señales, de la más a la menos rápida:

1. **Evento del Grupo 2 en el bus** (usuario dado de baja o sacado de
   `poli`), consumido con el mismo esquema de listeners que ya usamos
   para Espacios y Reclamos. Es la vía principal. **Depende del Grupo 2:**
   hoy no publica ese evento y hay que pedirlo.
2. **Reconciliación periódica:** si el Grupo 2 expone un endpoint que
   lista los usuarios de `poli`, un job compara esa lista con los
   oficiales vinculados y da de baja a los que faltan. Es el respaldo si
   no hay evento.
3. **Login sin `poli`:** si un `sub` vinculado llega con un token que ya
   no trae el grupo de oficial, se lo da de baja en ese momento. No cuesta
   nada, pero solo actúa si la persona vuelve a entrar.

### 4. Efecto de la baja

En una sola transacción:

1. El oficial queda `active = false` (sale del catálogo asignable de
   `GET /api/officers`).
2. Sus emergencias abiertas vuelven a `VALIDATED` sin oficial, con actor
   `system` y motivo `oficial dado de baja en LDAP`. Cada cambio deja su
   fila en `emergency_status_history` y su `EmergenciaActualizada` en el
   outbox, como cualquier transición.
3. A partir de ahí `authenticate` rechaza (403) a ese oficial aunque su
   token siga vigente, porque consulta `active` en el catálogo.

Por estado:

| Estado | Qué pasa |
|---|---|
| `DISPATCHED` | Vuelve a `VALIDATED` con la transición que ya existe. |
| `EN_ROUTE`, `ON_SCENE` | Vuelve a `VALIDATED` con una **transición nueva, solo para `system`**, y queda marcada para el dispatcher (puede haber alguien en el lugar). |
| `RESOLVED` | No cambia: el trabajo está hecho y lo cierra el dispatcher. |

Si la persona vuelve al grupo `poli`, al entrar se reactiva la misma fila
(mismo `idp_sub`): conserva su historial, pero no recupera los casos.

### 5. Criptografía

- **Se mantiene** la verificación RS256 del token contra el JWKS del
  Grupo 2: es lo que garantiza que el grupo `poli` es auténtico.
- **Se exige autenticación del evento de baja.** El listener solo acepta
  bajas que lleguen por el canal autenticado del gateway del Grupo 1 o
  firmadas por el IdP. Si no, cualquiera con acceso al bus podría
  desasignar a todos los oficiales.
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
| Evento del Grupo 2 en el bus | **Se hace** (depende del Grupo 2) | Llega en segundos y reutiliza la infraestructura de listeners. |
| Reconciliación periódica contra una API del Grupo 2 | **Se hace como respaldo** (depende del Grupo 2) | Cubre eventos perdidos o la ausencia de eventos. |
| Detectar el login sin `poli` | **Se hace** | Gratis, aunque llega tarde. |
| Dar de baja por inactividad (N horas sin entrar) | Descartada | No distingue a alguien dado de baja de alguien que no está de turno: desasignaría oficiales activos. `last_seen_at` queda solo como dato informativo. |
| Introspección del token en cada request | Descartada | Agrega un llamado al IdP por request (latencia y dependencia en línea) para achicar una ventana de 15 minutos que la baja en el catálogo ya cierra apenas nos enteramos. |
| Aceptar que el token venza solo, sin baja en el catálogo | Descartada | El oficial seguiría en el catálogo asignable y sus casos quedarían colgados. |

### Qué hacer con los casos del oficial dado de baja

| Alternativa | Decisión | Motivo |
|---|---|---|
| Devolver `DISPATCHED`, `EN_ROUTE` y `ON_SCENE` a `VALIDATED` (elegida) | **Se hace** | Ningún caso queda en manos de alguien sin acceso. |
| Devolver solo `DISPATCHED` | Descartada | Los casos en curso quedarían con un oficial que ya no puede actualizarlos. |
| Cancelar los casos | Descartada | La emergencia sigue existiendo: hay que reasignarla, no cerrarla. |
| Dejarlos asignados y solo avisar al dispatcher | Descartada | Depende de que alguien actúe; es justamente lo que queremos evitar. |

### Criptografía

| Alternativa | Decisión | Motivo |
|---|---|---|
| Verificar RS256 contra el JWKS (ya existe) | **Se mantiene** | Garantiza la autenticidad del grupo. |
| Exigir autenticación del evento de baja | **Se hace** | Evita desasignaciones forjadas. |
| Guardar `idp_sub`/`ldap_username` como HMAC con clave | Descartada por ahora | El `sub` ya es opaco y el catálogo guarda el nombre en claro; el hash impide leer el vínculo para depurar. Se revisa si Security clasifica estos identificadores como PII (se haría con una clave gestionada como `LOCATION_ENCRYPTION_KEY`). |
| Cifrar el vínculo con AES | Descartada | Para buscar por el valor haría falta además un índice ciego (HMAC); mismo costo, sin beneficio hoy. |

## Consecuencias

### Esperadas

Se hace más fácil:

- Dar de alta un oficial: alcanza con el grupo `poli` en el LDAP. No hay
  que tocar Render, código ni la base.
- Saber quién es cada oficial: el catálogo es el único lugar, y el LDAP
  manda sobre quién pertenece.
- Asegurar que ningún caso quede asignado a alguien sin acceso.
- Auditar: el historial y los eventos dicen `system` y el motivo de cada
  desasignación.

Se hace más difícil o cuesta:

- Dependemos del Grupo 2 para el evento de baja o el endpoint de listado.
  Hasta que exista alguno, la única señal automática es el login sin
  `poli`, y una baja puede tardar en reflejarse.
- Un oficial aparece en el catálogo recién después de su primer login: el
  dispatcher no puede asignar a alguien que nunca entró.
- Sin el claim `name`, el catálogo muestra el usuario (`of.gorgory`) en
  vez del nombre; sin chapa, la columna queda vacía.
- Cambia la máquina de estados (transición nueva `EN_ROUTE`/`ON_SCENE` →
  `VALIDATED` solo para `system`) y, con ella, lo que ven los
  consumidores de `EmergenciaActualizada`.
- `authenticate` pasa a consultar la base para los oficiales federados
  (mitigado con caché).
- Hay que mantener un listener nuevo y, si existe la API, un job.

Neutras:

- Los oficiales demo (`of-001`…`of-010`) siguen igual para las cuentas
  locales, en la misma tabla.
- El front no cambia: sigue usando `GET /api/officers` y el id `of-0XX`.

### No esperadas (riesgos a vigilar)

- **Baja masiva por error del LDAP:** si alguien quita por error el grupo
  `poli` (o el evento llega mal), se desasignan todos los casos en curso
  de golpe. Mitigación a evaluar: un tope de bajas por ventana que frene
  y alerte en vez de aplicar.
- **Oficial en el lugar desasignado:** un caso `ON_SCENE` vuelve a
  `VALIDATED` mientras la persona sigue físicamente ahí. El dispatcher
  podría mandar a un segundo oficial sin saberlo; por eso la marca en el
  tablero.
- **Consumidores del bus que no esperan ir hacia atrás:** otros módulos
  pueden asumir que después de `EN_CAMINO` no viene `VALIDADA`. Hay que
  avisarlo como cambio de contrato.
- **Reutilización de usuarios en el LDAP:** si el Grupo 2 reasigna un
  `preferred_username` a otra persona, una reserva por `ldap_username`
  sin usar quedaría para la persona equivocada. Por eso, una vez
  vinculada, la fila se busca por `idp_sub` y la reserva no se reutiliza.
  Si el Grupo 2 reutilizara un `sub`, el vínculo sería incorrecto: hay que
  confirmar que no lo hace.
- **Caché desactualizada con varias instancias:** con más de un proceso,
  uno puede seguir aceptando a un oficial ya dado de baja hasta que vence
  su caché. Hoy hay una sola instancia en Render; el TTL acota la ventana.
- **Eventos duplicados o fuera de orden:** una baja que llega dos veces o
  después de una realta no puede desasignar casos nuevos. El listener
  tiene que ser idempotente (como los actuales, con
  `eventos_procesados`) y comparar fechas.
- **Catálogo que crece:** cada `poli` que entra una vez queda en la tabla.
  Los dados de baja no se borran (por el historial), solo se ocultan.
- **Ventana de 15 minutos** entre la baja en el LDAP y que nos enteremos,
  si no hay evento: el token viejo sigue sirviendo hasta que llega alguna
  señal.
- **Experiencia del oficial dado de baja:** pasa de golpe a recibir 403;
  el front tiene que mostrarlo y cerrarle la sesión en vez de reintentar.

### Pendientes para pasar a `aceptada`

1. Pedir al Grupo 2: evento de baja (o cambio de grupo), endpoint que
   liste los usuarios de `poli`, claim `name` (y chapa si la tienen), y
   confirmar que el refresh rechaza usuarios dados de baja y que no
   reutilizan `sub`.
2. Confirmar con Producto la transición nueva desde `EN_ROUTE`/`ON_SCENE`.
3. Confirmar con Security que el vínculo puede guardarse en claro.
4. Definir si `of.gorgory` es un oficial nuevo o se reserva uno de los
   existentes con `ldap_username`.

## Relacionados

- [Usar identidad local con JWT y sacar la PII del navegador](usar-identidad-local-con-jwt.md)
- [Autorizar por JWT con ownership y endurecer el perímetro HTTP](autorizar-por-jwt-con-ownership-y-endurecer-el-perimetro-http.md)
- [Consumir eventos externos con un listener y reglas por evento](consumir-eventos-externos-con-un-listener-y-reglas-por-evento.md)
- [Cifrar ubicación y PII sensible en reposo](cifrar-ubicacion-y-pii-sensible-en-reposo.md)

## Historial

- 2026-11-15: creación del ADR (estado: propuesta)
