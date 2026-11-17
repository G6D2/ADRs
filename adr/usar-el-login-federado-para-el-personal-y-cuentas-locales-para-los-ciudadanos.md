# Usar el Login Federado para el personal y cuentas locales para los ciudadanos

- Estado: aceptada
- Fecha: 2026-11-17
- Decisores: Producto (@juanimoli), que lo aprobó el 2026-11-17. Lo implementaron en el backend @juanimoli (PR #89, modo híbrido) y @Salterm27 (PR #110, oficiales con el grupo `poli`, y PR #118, login de demo solo con ciudadanos); en el front, @SJTG (Frontend #56, login LDAP del personal) y @Salterm27 (Frontend #77).
- Reemplaza a: [Usar identidad local con JWT y sacar la PII del navegador](usar-identidad-local-con-jwt.md)
- Reemplazada por: —

Propuesto en el issue [#38](https://github.com/G6D2/ADRs/issues/38).

## Contexto

[Usar identidad local con JWT y sacar la PII del navegador](usar-identidad-local-con-jwt.md)
resolvió el login mientras el IdP de CityPass+ no estaba integrado: el
backend emitía sus propios tokens para todos y dejaba `AUTH_MODE=idp` para
pasar al IdP «cambiando configuración, no código de negocio». También
anticipaba que el ADR de la integración la iba a reemplazar.

La integración con el Login Federado del Grupo 2 no fue solo configuración:

- **El Login Federado tiene solo personal municipal.** Dispatchers,
  oficiales y delegados son usuarios de su LDAP, con grupos por módulo.
  Los ciudadanos no están, y el botón de pánico es del ciudadano.
- **El front ya loguea al personal por LDAP** desde Frontend #56 (06/11)
  y le manda ese token al backend. Con `AUTH_MODE=local`, el backend lo
  rechazaba.
- **El token real no es el que esperaba `idp`.** Confirmado el 12/11 con
  un token real: `iss` = `https://idp.citypass.local`, `aud` =
  `citypass-emergencias-api` como string y grupos con los nombres del LDAP
  (`dispatcher`, `delegados` y, desde el 15/11, `poli` para los oficiales).
  `AUTH_MODE=idp` exige `aud` como lista y usa los grupos tal como vienen,
  así que tampoco lo acepta.
- **Los dos emisores usaban el mismo `iss`.** Los tokens locales salían
  con `https://idp.citypass.local`, el mismo texto que el Login Federado.
  Con el mismo `iss` no hay forma de saber contra qué clave validar.
- **Comparten el espacio de `sub`.** El Login Federado usa subs como
  `U000042`, que también es un sub demo local.
- **El login de demo daba tokens de personal.** `POST /auth/dev/login`
  es público y no pide contraseña. Hasta el 17/11, su allowlist incluía
  `DISP-Demo001`, `of-001` y `of-002`: cualquiera con la URL de producción
  sacaba un token de dispatcher.

Producción corre con `AUTH_MODE=hybrid` desde el 12/11/2026, y el login
de demo da solo ciudadanos desde el 17/11 (backend #118). Este ADR
registra esa decisión.

## Decisión

Vamos a autenticar al personal con el Login Federado del Grupo 2 y a los
ciudadanos con cuentas locales, en el mismo backend (`AUTH_MODE=hybrid`).
El `iss` del token decide contra qué claves se valida.

| | Personal (dispatcher, oficial, delegado) | Ciudadano |
|---|---|---|
| Dónde entra | Login Federado (LDAP), desde el front | `POST /auth/register` y `POST /auth/login` del backend |
| Quién firma | El Grupo 2 | El backend |
| Firma | RS256, contra el JWKS del Grupo 2 (`IDP_JWKS_URI`) | RS256, con la clave local (`LOCAL_JWT_PRIVATE_KEY_PEM`, o una generada al arrancar) |
| `iss` | `IDP_ISSUER` = `https://idp.citypass.local` | `LOCAL_JWT_ISSUER` = `urn:citypass:emergencias:local` (default) |
| `aud` | Incluye `citypass-emergencias-api` (string o lista) | Lista con `citypass-emergencias-api` |
| Grupos | Los del LDAP, traducidos con un mapa | Los de `auth_users`: el registro siempre da `citizen` |
| Sesión | 15 minutos; el BFF la renueva con el refresh token del Grupo 2 | 15 minutos (`LOCAL_TOKEN_TTL_SECONDS`), sin renovación |

### 1. Un modo, dos emisores

`AUTH_MODE` admite tres valores (`src/auth/config.js` y `src/auth/setup.js`):

| Modo | Qué tokens acepta | Rutas `/auth/*` |
|---|---|---|
| `local` (default) | Solo locales. Con `NODE_ENV=production` no arranca sin `ALLOW_LOCAL_AUTH=true`. | Sí |
| `idp` | Solo del IdP, con el contrato estricto (`aud` lista, `token_use` y `ver` obligatorios, grupos sin traducir). | No |
| `hybrid` (producción) | Los dos. | Sí, con la misma política que en `local` |

En `hybrid` (`src/auth/hybrid-verifier.js`):

- `decodeJwt` lee el `iss` solo para elegir el verificador. Después corre
  la verificación completa: firma, `iss` exacto, `aud` y `exp`. Un `iss`
  falso no sirve: un token que dice ser federado se valida contra el JWKS
  del Grupo 2, y uno que dice ser local, contra la clave local.
- Cualquier otro `iss` da `401 Unknown token issuer`.
- Si `LOCAL_JWT_ISSUER` es igual a `IDP_ISSUER`, el servidor no arranca.
- No depende de `NODE_ENV` ni de `ALLOW_LOCAL_AUTH`: elegir `hybrid` ya
  declara que se aceptan tokens locales.
- Volver atrás es `AUTH_MODE=local`.

### 2. Token del Login Federado

`src/auth/federated-verifier.js` siempre verifica la firma y exige:

- RS256, con las claves de
  `https://api.citypass-login.pp.ua/.well-known/jwks.json` (`IDP_JWKS_URI`
  solo admite `https`, salvo contra `localhost`);
- `iss` exacto, sin barra final;
- `aud` que incluya `citypass-emergencias-api`;
- `exp` presente y vigente, con la tolerancia de
  `IDP_CLOCK_TOLERANCE_SECONDS` (60 s por default);
- `token_use` y `ver`, si vienen, iguales a `human` y `1`: un token de
  servicio da 401. Son obligatorios solo con `IDP_REQUIRE_TOKEN_USE` e
  `IDP_REQUIRE_TOKEN_VER`, que `render.yaml` no prende;
- `sub` no vacío.

El claim `module` no se valida: lo que ata el token a Emergencias es `aud`.

### 3. Grupos del LDAP a grupos del módulo

`src/auth/federated-claims.js` traduce los grupos con un mapa. Producción
usa el mapa por default: `render.yaml` no define `IDP_GROUP_MAP`.

| Grupo del LDAP | Grupo interno |
|---|---|
| `dispatcher`, `dispatchers`, `operador` | `dispatcher` |
| `officer`, `poli` | `officer` |
| `delegado`, `delegados` | `delegado`: no habilita ninguna ruta. `GET /api/user` lo informa y el front lo manda al panel del Login Federado. |

- Las mayúsculas no importan, y un grupo que no está en el mapa no da nada.
- Si se define `IDP_GROUP_MAP`, reemplaza el mapa entero. Un JSON roto o
  un grupo interno que no existe frenan el arranque.
- **`citizen` no está en el mapa, a propósito.** Como los dos emisores
  comparten el espacio de `sub`, un federado con `citizen` podría ver las
  emergencias de un ciudadano local con el mismo `sub`.

### 4. Oficiales

Las acciones del oficial exigen que el `sub` sea el
`assigned_officer_id` del catálogo (`of-001`…), y el `sub` del LDAP
(`U000025`) no lo es. `IDP_SUB_ALIASES` lo traduce en `authenticate`,
antes de autorizar y solo en tokens federados; el `sub` original queda en
`idpSub`. Producción tiene un alias en `render.yaml`:
`{"U000025":"of-001"}` (`jefe.gorgory`). Cómo se reemplaza ese puente
está en
[Vincular los oficiales del LDAP al catálogo de oficiales](vincular-los-oficiales-del-ldap-al-catalogo-de-oficiales.md).

### 5. Ciudadanos

- Registro y login propios, sobre la tabla `auth_users` (V16) con bcrypt.
  `POST /auth/register` crea siempre un `citizen`. `POST /auth/login`
  responde igual si el correo no existe.
- Sigue igual que en el ADR anterior: el perfil y las ubicaciones
  frecuentes se guardan cifrados en el backend (`/api/citizen/*`); el BFF
  guarda el access token en una cookie `httpOnly`, y el `middleware.js`
  del front es un guard de UX, sin validar la firma. La autoridad de
  seguridad sigue siendo la API.

### 6. Renovación de la sesión del personal

La hace el BFF del front (`/api/auth/refresh`), no el backend. Guarda el
refresh token del Grupo 2 en otra cookie `httpOnly` y lo canjea cada 13
minutos, antes de que venza el access token de 15. Solo las sesiones del
Login Federado tienen refresh token. El Grupo 2 rota el refresh token, relee
los grupos en el LDAP y rechaza las cuentas borradas, deshabilitadas o
movidas de módulo (confirmado el 15/11, ver el ADR de oficiales). El
backend solo ve access tokens.

### 7. Login de demo en producción: solo ciudadanos

`POST /auth/dev/login` emite un token local sin contraseña. Sigue en
producción como **concesión para la demo**: el front lo usa en los atajos
"Ciudadano con SOS" (`CIT-JuanPanico`) y "Ciudadano anónimo" (`U000042`).

- Con `DEV_LOGIN_STRICT=true` (en `render.yaml`, y también el default con
  `NODE_ENV=production`), solo acepta tres `sub`, con grupos fijos que el
  cliente no elige: `U000042`, `CIT-JuanPanico` y `CIT-LuciaTranquila`,
  los tres `citizen` (`src/auth/dev-login-policy.js`). Cualquier otro da
  400.
- Desde el 17/11 (backend #118) no hay dispatcher ni oficiales en la
  lista: el personal entra solo por el Login Federado.
- Rate limit: 10 por minuto por `sub` pedido y 100 por minuto por IP.
- Swagger, público en `/api-docs`, documenta el endpoint, con `U000042`
  como ejemplo. Queda así porque es la documentación con la que integran
  los otros grupos.

El riesgo de esta concesión está en Consecuencias.

## Alternativas consideradas

| Alternativa | Motivo de descarte |
|---|---|
| Solo cuentas locales (`AUTH_MODE=local`, como hasta el 12/11) | El personal ya entra por el LDAP del Grupo 2, que es la identidad del personal en CityPass+, y `local` rechaza ese token. Mantener cuentas propias de dispatcher duplica las altas y bajas del personal, y el atajo sin contraseña daba un token de dispatcher a cualquiera. |
| Solo Login Federado (`AUTH_MODE=idp`) | Deja afuera a los ciudadanos: el Login Federado no los tiene, y sin ciudadanos no hay botón de pánico. Además, `idp` tal como está no acepta el token real (`aud` string, grupo `poli`). |
| Login Federado para todos, con los ciudadanos dados de alta en el IdP | Depende de que el Grupo 2 incorpore a los vecinos, que hoy no tiene. Darlos de alta a mano desde el panel del delegado no escala, pone datos de vecinos en el LDAP de otro grupo y nos saca el registro desde la app. |
| Mantener dispatcher y oficiales en el login de demo (allowlist hasta el 17/11) | Un endpoint público sin contraseña que da tokens de dispatcher anula el Login Federado. |
| Sacar el login de demo de producción ya | Es lo que corresponde después de la demo, pero hoy la demo usa los atajos de ciudadano. Queda como pendiente. |

## Consecuencias

### Esperadas

Se hace más fácil:

- Que el personal tenga una sola cuenta en CityPass+: las altas, las
  bajas y los grupos se manejan en el LDAP del Grupo 2.
- Que los ciudadanos se registren desde la app, sin depender de otro
  grupo.
- Proteger al personal: en producción, un token de dispatcher u oficial
  exige la contraseña del LDAP.
- Saber de dónde viene cada identidad: `GET /api/user` devuelve
  `authSource` (`local` o `federated`) y, en los tokens federados,
  `idpSub` e `idpGroups`.
- Mantener la autorización: sigue mirando `sub` y `groups`, venga el
  token de donde venga.

Se hace más difícil o cuesta:

- Hay dos emisores con reglas distintas para mantener (`aud` lista o
  string, `token_use` obligatorio u opcional).
- Dependemos del JWKS del Grupo 2. Si no responde cuando el backend tiene
  que recargarlo, los tokens del personal dan 401. A los ciudadanos no
  les cambia nada.
- Cada oficial del LDAP necesita un alias en Render hasta que se
  implemente el vínculo en la tabla `officers`.
- Las sesiones locales duran 15 minutos y no se renuevan.
- `render.yaml` no declara `LOCAL_JWT_PRIVATE_KEY_PEM`. Si tampoco está
  en el dashboard, cada reinicio genera una clave nueva y corta las
  sesiones locales vigentes.
- `AUTH_MODE=idp` quedó como estaba y no sirve con el token real. Si un
  día se quiere solo federado, primero hay que alinearlo con `hybrid`.

Neutras:

- Al pasar de `local` a `hybrid` cambia el `iss` de los tokens locales:
  los que estaban vigentes dejan de servir y hay que volver a entrar.
- El rol `officer`, que
  [Autorizar por JWT con ownership y endurecer el perímetro HTTP](autorizar-por-jwt-con-ownership-y-endurecer-el-perimetro-http.md)
  dejaba fuera de alcance, ya existe: el oficial entra por el Login
  Federado y solo avanza la emergencia que tiene asignada.

### No esperadas (riesgos a vigilar)

- **SOS de demo sin autenticación real.** Cualquiera que lea Swagger o el
  front puede sacar un token de ciudadano demo y disparar un SOS. La
  emergencia entra a la cola del dispatcher y sale como `EmergenciaCreada`
  al bus, donde reaccionan otros módulos. El backend no mira
  `has_antipanic_button` en el alta, así que también sirven `U000042` y
  `CIT-LuciaTranquila`. Lo acotan el rate limit del SOS (5 por minuto por
  `sub`) y el del login de demo. Es un riesgo aceptado para la demo.
- **Identidades demo compartidas.** Todos los que entran con el mismo
  ciudadano demo son la misma persona para el backend: ven y cancelan las
  emergencias de los demás, con su ubicación, y comparten el perfil.
- **El registro también está abierto.** `POST /auth/register` no
  verifica el correo, así que crear un ciudadano y disparar un SOS no
  necesita el login de demo. Lo que suma el login de demo es la falta
  total de fricción y las identidades compartidas.
- **No hay interruptor.** En `hybrid`, `/auth/dev/login` se monta
  siempre. Sacarlo después de la demo pide un cambio de código, por
  ejemplo no montarlo con `NODE_ENV=production`.
- **`DEV_LOGIN_STRICT=false` en producción** vuelve al modo laxo:
  cualquier `sub`, con `dispatcher` u `officer`. El Login Federado dejaría
  de proteger al personal.
- **Un alias mal cargado** en `IDP_SUB_ALIASES` hace que una persona del
  LDAP opere como otro oficial, con sus casos y su historial.
- **`IDP_GROUP_MAP` reemplaza el default entero.** Si se define para
  sumar un grupo y se olvida uno de los del default, ese personal pierde
  el acceso.

### Pendientes

- Sacar o restringir el login de demo en producción después de la demo
  del 26/11, y decidir si Swagger lo sigue documentando.
- Decidir si se prenden `IDP_REQUIRE_TOKEN_USE` e `IDP_REQUIRE_TOKEN_VER`:
  el token real trae los dos claims.
- Implementar el vínculo de los oficiales en la tabla `officers` (ADR de
  oficiales).

## Relacionados

- [Vincular los oficiales del LDAP al catálogo de oficiales](vincular-los-oficiales-del-ldap-al-catalogo-de-oficiales.md)
- [Autorizar por JWT con ownership y endurecer el perímetro HTTP](autorizar-por-jwt-con-ownership-y-endurecer-el-perimetro-http.md)
- [Cifrar ubicación y PII sensible en reposo con AES-256-GCM](cifrar-ubicacion-y-pii-sensible-en-reposo.md)
- [Adoptar Next.js con App Router y un BFF](adoptar-next-js-con-app-router-y-un-bff.md)

## Historial

- 2026-11-17: creación del ADR (estado: propuesta). Registra el modo
  híbrido que corre en producción desde el 12/11 (backend #89) y el login
  de demo solo con ciudadanos (backend #118, 17/11).
- 2026-11-17: cambio de estado: propuesta → aceptada
