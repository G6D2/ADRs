# Usar identidad local con JWT y sacar la PII del navegador

- Estado: aceptada
- Fecha: 2026-09-24
- Decisores: @valenfiumana (Security), @juanimoli (Producto)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

El IdP de CityPass+ todavía no está integrado. Necesitamos:

- login de punta a punta para la demo;
- roles y ownership;
- perfil ciudadano;
- no guardar PII sensible en `localStorage`.

La solución local tiene que respetar un contrato compatible con el IdP,
para no reescribir la autorización cuando llegue. Para el Hito 2 se está
integrando el login federado del Grupo 2: este ADR describe la decisión
vigente hasta entonces, y el ADR de esa integración lo va a reemplazar
con `tools/adr supersede`.

## Decisión

### Modo local

Con `AUTH_MODE=local` (el valor por defecto), el backend emite y
verifica JWT **RS256** con la misma validación que usaría con el IdP;
solo cambia quién firma (`src/auth/local-keys.js`). Los claims siguen la
estructura de CityPass+ (`sub`, `groups`, `token_use`, `ver`). En
producción el modo local solo arranca con `ALLOW_LOCAL_AUTH=true`.

### Usuarios

Tabla `auth_users` (V16): `email`, `password_hash` con bcrypt,
`display_name` y `groups`. Endpoints `POST /auth/register` y
`POST /auth/login`, con rate limit.

### Perfil sensible

El perfil y las ubicaciones frecuentes se manejan por `/api/citizen/*` y
se guardan cifrados en el backend (ver
[Cifrar ubicación y PII sensible en reposo](cifrar-ubicacion-y-pii-sensible-en-reposo.md)).
El navegador no es fuente de verdad de PII: el front solo borra restos
viejos de `localStorage`.

### Login de desarrollo

`POST /auth/dev/login` queda solo para tests y demo. Se restringe con
`ALLOW_LOCAL_AUTH`, `DEV_LOGIN_STRICT` y una allowlist de usuarios demo
(`src/auth/dev-login-policy.js`). El modo estricto, que acepta solo la
allowlist, se activa con `DEV_LOGIN_STRICT=true` o con
`NODE_ENV=production`.

### Frontend

El BFF de Next.js guarda el access token en una cookie `httpOnly`. El
`middleware.js` del front es un guard de UX (presencia y vigencia de la
cookie, rol); no valida la firma. La autoridad de seguridad sigue siendo
la API.

### Migración al IdP

`AUTH_MODE=idp` con `IDP_ISSUER`, `IDP_AUDIENCE` e `IDP_JWKS_URI`: el
backend verifica contra el JWKS del IdP. La lógica de negocio y la
autorización no cambian.

## Alternativas consideradas

| Alternativa | Motivo de descarte |
|---|---|
| Esperar al IdP | Bloquea la demo y las pruebas de autorización. |
| Google OAuth | Fuera del alcance y del contrato de CityPass+. |
| Whitelist de emails solo en el front | No es autenticación real. |
| Guardar el perfil en `localStorage` | Queda expuesto ante XSS. |
| Autorizar solo desde el middleware del front | El front no puede ser la autoridad de seguridad. |

## Consecuencias

Se hace más fácil:

- Hacer la demo completa sin depender de otro grupo.
- Pasar al IdP cambiando configuración, no código de negocio.
- Tener la PII sensible en un solo lugar y cifrada.
- Mantener la autorización del dominio separada del proveedor de
  identidad.

Se hace más difícil:

- Hay dos modos de autenticación para mantener mientras dure la
  transición.
- `ALLOW_LOCAL_AUTH` se tiene que controlar con cuidado en producción.
- Las cuentas demo no representan el esquema de autenticación final.
- Pendiente de configuración: el 2026-09-24 el login de desarrollo en
  Render aceptaba cualquier `sub` (probado con `QA-API-*`). El código lo
  restringe, pero el servicio no tenía `NODE_ENV=production` ni
  `DEV_LOGIN_STRICT=true`. Hay que verificarlo cuando el servicio vuelva
  a estar arriba.

## Historial

- 2026-09-24: propuesta en el issue #21 y aprobada ese día.
- 2026-10-21: se pasa a archivo como aceptada, con título en imperativo.
  Se aclara que rige hasta el login federado y se registra el pendiente
  de configuración del login de desarrollo (review del issue).
