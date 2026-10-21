# Cifrar ubicación y PII sensible en reposo con AES-256-GCM

- Estado: aceptada
- Fecha: 2026-09-24
- Decisores: @valenfiumana (Security), @juanimoli (Producto)
- Reemplaza a: —
- Reemplazada por: —

## Contexto

El módulo maneja información sensible:

- coordenadas y dirección de cada emergencia;
- teléfono, contacto de emergencia y condición médica del ciudadano;
- ubicaciones frecuentes del ciudadano.

Guardar estos datos en claro en Postgres o en el navegador aumenta el
impacto de una filtración de la base, de un backup o de un dispositivo.
También necesitamos poder rotar claves sin tener que recifrar todo el
histórico de una sola vez.

## Decisión

- Vamos a cifrar en la aplicación, antes de persistir, con
  **AES-256-GCM** (`src/security/location-crypto.js` del backend).
- Emergencias: latitud, longitud y dirección se guardan juntas en
  `location_encrypted`. Las columnas `latitude` y `longitude` quedan en
  `NULL` en los inserts nuevos (migración V3).
- Perfil ciudadano: teléfono, contacto de emergencia y condición médica
  van en `citizen_profiles.sensitive_encrypted` (V15).
- Ubicaciones guardadas: `citizen_saved_locations.location_encrypted`
  (V15).
- Las claves llegan por variables de entorno: `LOCATION_ENCRYPTION_KEY`
  (una sola clave) o el keyring `LOCATION_ENCRYPTION_KEYS`
  (`id:clave,id2:clave2`) con `LOCATION_ENCRYPTION_KEY_ID` como clave
  activa.
- El formato es versionado: cada valor lleva `keyId`, `iv`, `authTag` y
  `ciphertext`. Se cifra siempre con la clave activa y se descifra con la
  que indique cada fila, así se puede rotar sin recifrar el histórico.
  Las filas escritas antes del versionado se siguen leyendo.
- Las passwords del login local van con **bcrypt**, no con AES (ver
  [Usar identidad local con JWT](usar-identidad-local-con-jwt.md)).
- Minimizamos lo que queda guardado: un job anonimiza la ubicación
  cifrada de las emergencias después de `LOCATION_RETENTION_HOURS`
  (72 h por defecto) y otro limpia la PII de perfiles inactivos y las
  ubicaciones guardadas sin actualizar después de
  `CITIZEN_PII_RETENTION_DAYS` (90 días por defecto).
- El email de login queda en claro: es el identificador de la cuenta.
- Si se acepta guardar el barrio de la emergencia en claro (PR #26), es
  una excepción explícita a esta decisión: es una ubicación aproximada
  que hace falta para filtrar sin descifrar.

## Alternativas consideradas

| Alternativa | Motivo de descarte |
|---|---|
| Solo cifrado de disco / Postgres | No protege el contenido ante acceso a la base o a los backups. |
| `pgcrypto` | Lleva secretos y lógica criptográfica a la base y complica la rotación y la auditoría. |
| AWS KMS / Vault | Válido a futuro, pero fuera del alcance de Render + Neon para esta entrega. |
| AES-CBC | No autentica el ciphertext. |
| Cifrar el email de login | Complica la autenticación y las búsquedas. |

## Consecuencias

Se hace más fácil:

- Que la ubicación exacta y la PII no queden en claro en la base ni en
  los backups.
- Reusar el mismo mecanismo para cualquier PII nueva.
- Rotar claves con el keyring sin recifrar todo de una vez.
- Detectar alteraciones: GCM da confidencialidad e integridad, un
  ciphertext modificado no se descifra.

Se hace más difícil:

- Todo lo que necesita la ubicación tiene que descifrarla en la
  aplicación: no se puede filtrar, ordenar ni agrupar por ubicación en
  SQL.
- La gestión de claves: hoy viven como variables de entorno del servicio
  en Render (`LOCATION_ENCRYPTION_KEY`, `sync: false` en `render.yaml`).
  Las ve y las puede rotar quien administre el servicio en Render.
- Si se pierde una clave, los datos cifrados con ella quedan
  irrecuperables. Una clave vieja no se saca del keyring mientras haya
  filas que la usen.

## Historial

- 2026-09-24: propuesta en el issue #20 y aprobada ese día.
- 2026-10-21: se pasa a archivo como aceptada. Se suma dónde viven las
  claves y quién las rota (review del issue), la retención que ya está
  en `main` y la excepción del barrio en claro (PR #26).
