# Autorizar por JWT con ownership y endurecer el perímetro HTTP

- Estado: aceptada
- Fecha: 2026-09-24
- Decisores: @valenfiumana (Security), @juanimoli (Producto)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

Un JWT válido no implica acceso total al módulo. Los actores actuales
tienen permisos distintos: `citizen` y `dispatcher`. El ciudadano tiene
que acceder solo a sus propios recursos. El rol `officer` queda fuera del
alcance: hoy los oficiales existen como catálogo, no como identidad
autenticada.

Además necesitamos reducir el abuso sobre los endpoints sensibles y
evitar filtraciones por logs o por mensajes de error.

## Decisión

### Autorización

La identidad y los permisos salen del JWT (`sub` y `groups`). No
confiamos en `userId`, `actor` ni otros identificadores que lleguen en
el body o en el query string.

### Ownership

El ciudadano ve solo sus emergencias, cancela solo las propias y no
puede consultar recursos ajenos. Respuestas:

- `401`: no autenticado;
- `403`: sin permisos;
- `404`: recurso ajeno, en los flujos donde no queremos revelar que
  existe (detalle y cancelación).

### Auditoría

Las lecturas sensibles de emergencias quedan registradas en
`emergency_access_audit` (V3), con el `sub` del actor.

### Perímetro

- Rate limit por minuto: 20 en login y registro, 10 en el login de
  desarrollo, 5 en el botón de pánico y 30 en las mutaciones del perfil.
- CORS por allowlist (`CORS_ALLOWED_ORIGINS`; si no está definida, solo
  el front deployado).
- HSTS con `NODE_ENV=production` o `FORCE_HTTPS=true`, y headers de
  seguridad en todas las respuestas.
- `secureLog` con redacción de tokens, passwords, ubicación y PII.
- Errores 5xx saneados: el detalle va solo al log, el cliente recibe un
  mensaje genérico.

### Base de datos

La conexión a PostgreSQL usa TLS. La validación estricta del
certificado se activa con `DATABASE_SSL_VERIFY=true`; por defecto queda
apagada para no romper el pooler de Neon.

## Alternativas consideradas

| Alternativa | Motivo de descarte |
|---|---|
| Autorizar solo en el front | Se saltea llamando directo a la API. |
| Implementar `officer` ahora | Fuera del alcance actual. |
| RBAC fino en la base | Complejidad innecesaria para los roles que hay. |
| Sin rate limiting | Riesgo innecesario en un staging o demo público. |
| Forzar ya la validación estricta del certificado de Neon | Riesgo operativo durante el freeze de la demo. |

## Consecuencias

Se hace más fácil:

- Que la API sea la autoridad real de autorización.
- Limitar el impacto de un token de ciudadano comprometido.
- Rastrear quién leyó qué emergencia.
- Reducir el abuso y la exposición accidental de información.

Se hace más difícil:

- Front y back tienen que mantener una UX coherente, aunque la decisión
  final sea siempre del back.
- El rol `officer` queda pendiente a propósito.
- La política de TLS se tiene que endurecer después de la demo.
- El rate limit vive en memoria de cada instancia: con varias instancias
  en Render el límite no es exacto.
- Pendiente: hoy la auditoría se registra solo en
  `GET /api/panic-button/:id`. El front lee el detalle por
  `GET /api/emergencies/:id`, que todavía no registra; falta sumarlo
  ahí.
- Pendiente de configuración: el 2026-09-24 Render no mandaba el header
  HSTS, porque el servicio no tenía `NODE_ENV=production` ni
  `FORCE_HTTPS=true`.
- El consumidor de eventos en un servicio aparte (decisión del
  2026-10-08) va a llamar a comandos del back con un token de servicio,
  y eso entra en este perímetro. Hoy el back solo acepta tokens con
  `token_use` humano, así que habilitarlo necesita su propio permiso.

## Historial

- 2026-09-24: propuesta en el issue #22 y aprobada ese día.
- 2026-10-21: se pasa a archivo como aceptada, con título en imperativo.
  Se suman los pendientes del review (HSTS en Render, rate limit en
  memoria), la brecha de auditoría en el detalle de emergencias y el
  token de servicio del consumidor de eventos.
