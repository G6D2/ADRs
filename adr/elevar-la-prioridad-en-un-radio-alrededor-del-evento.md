# Elevar la prioridad en un radio alrededor del evento

- Estado: propuesta
- Fecha: 2026-11-12
- Decisores: @Salterm27 (lo implementó en el backend). Lo confirma Producto (@juanimoli), que todavía no lo revisó.
- Reemplaza a: [Elevar temporalmente la prioridad con contextos de zona](elevar-temporalmente-la-prioridad-con-contextos-de-zona.md)
- Reemplazada por: —

## Contexto

El ADR anterior definió que, mientras dura un evento público, las emergencias de la zona suben un nivel de prioridad de despacho, y que **la zona es el barrio**. Evaluó la zona por radio y la descartó (alternativa F), porque obligaba a descifrar ubicaciones en cada lectura.

Con el mecanismo por barrio andando aparecieron dos problemas:

- **El barrio es demasiado grueso.** Un evento en una plaza de Palermo subía la prioridad de todo Palermo, incluidas emergencias a veinte cuadras.
- **Y a la vez deja afuera lo que está cerca.** La vereda de enfrente no sube si queda del otro lado del límite del barrio.

El obstáculo para medir distancias sigue en pie: la ubicación de la emergencia se guarda cifrada ([Cifrar ubicación y PII sensible en reposo](cifrar-ubicacion-y-pii-sensible-en-reposo.md)) y el listado ordena y pagina en SQL, así que no se puede calcular una distancia al leer sin descifrar.

Este ADR registra el cambio tal como quedó en el backend (PR #96 de `emergencias-backend`, en producción desde el 12/11/2026). El cambio entró sin pasar por este registro ni por la revisión de Producto: el ADR queda en `propuesta` hasta que Producto lo confirme.

## Decisión

Vamos a definir la zona de un evento como **un punto y un radio**, y a medir la distancia **una sola vez, al crear la emergencia**.

**Zona.** `contexto_zona` deja de tener `barrio` y pasa a tener `latitud`, `longitud` y `radio_metros`. El punto es la ubicación del evento, que no es un dato personal y va en claro. Un evento sin coordenadas numéricas no genera contexto: el texto de barrio o zona del evento ya no se usa.

**Radio.** Es un valor fijo del backend, configurable con `CONTEXTO_ZONA_RADIO_DEFAULT_METROS` (500 m por defecto). Espacios no informa un radio. Cada contexto guarda el radio con el que se registró.

**Medición al crear.** Cuando se crea una emergencia, la aplicación tiene la coordenada en memoria, antes de cifrarla. En ese momento busca los contextos en curso, mide la distancia y, si la emergencia cae dentro del radio, guarda el vínculo en la tabla `emergencia_contexto_zona`: solo los dos identificadores. No se guarda ninguna coordenada en claro ni la distancia, y no se descifra nada al leer.

**Cálculo al leer.** La prioridad de despacho sigue calculándose en SQL: un nivel más si la emergencia tiene un vínculo con un contexto que sigue en curso. Por eso baja sola cuando el evento termina, se suspende o se cancela, y el orden entre páginas sigue siendo correcto.

**Motivo.** `priorityContext` informa `latitud`, `longitud` y `radioMetros` del evento, en lugar del barrio.

**Retención.** El vínculo se borra cuando la retención anonimiza la ubicación de la emergencia.

Lo que no cambia respecto del ADR anterior: el ajuste es de un nivel con tope en la prioridad máxima, varios eventos sobre la misma emergencia no se acumulan, las coberturas preventivas no suben, y lo que llega viejo no pisa lo nuevo (una cancelación que llega antes que la publicación gana).

### Reglas que cambian

1. **Solo suben las emergencias creadas con el evento en curso.** Una emergencia que ya existía cuando el evento empieza, o cuando llega su publicación, no sube aunque siga abierta. Antes subía cualquier emergencia abierta del barrio.
2. **La zona es un radio, no un barrio.** Una emergencia del mismo barrio a más de 500 m no sube; una de otro barrio a menos de 500 m sí.
3. **Los vínculos no se recalculan.** Si un evento cambia de lugar, las emergencias ya vinculadas quedan como estaban.
4. **`emergencies.barrio` y `data.barrio` en el bus no cambian.** El barrio sigue existiendo y lo siguen usando los otros módulos; solo deja de intervenir en esta regla.

## Alternativas consideradas

| Alternativa | A favor | En contra |
| --- | --- | --- |
| Seguir por barrio (ADR anterior) | Sube también las emergencias que ya estaban abiertas. No necesita coordenadas del evento. | Zona demasiado gruesa y con bordes arbitrarios. |
| Radio medido al leer (alternativa F del ADR anterior) | Alcanza a las emergencias previas al evento. | Descifra ubicaciones en cada lectura del listado. |
| **Radio medido al crear, con el vínculo guardado** | Preciso. No descifra nada al leer ni guarda coordenadas en claro. Ordena en SQL. | Las emergencias previas al evento no suben. |

## Consecuencias

Se hace más fácil:

- Que la suba alcance a lo que está realmente cerca del evento, sin importar el límite del barrio.
- Mantener el listado ordenado en SQL sin tocar la ubicación cifrada.

Se hace más difícil, y hay que decidir si lo aceptamos:

- **Las emergencias que ya estaban abiertas no suben.** Es el cambio de comportamiento más visible. En una demo, el orden importa: primero se publica el evento y después se crea la emergencia.
- **Los contextos creados por barrio antes del cambio no alcanzan a ninguna emergencia** hasta que Espacios republique el evento, porque no tienen punto.
- **Los módulos que recalculan la suba tienen que usar la regla nueva.** La guía `docs/prioridad-por-contexto-para-consumidores.md` del backend ya la describe; falta avisarles.
- **Una alerta climática no tiene un radio natural.** El ADR anterior la resolvía por barrio. Con este mecanismo queda sin definir cómo entra; hoy no hay listener de alertas climáticas.
- **Un radio único para todo tipo de evento.** Un recital y una feria chica pesan igual.

Sin implementar todavía, igual que con el ADR anterior: el alta y la baja de contextos a mano desde el tablero, el aviso por alerta climática y la prioridad manual.

### Preguntas abiertas para Producto

- [ ] ¿Aceptamos que las emergencias anteriores al evento no suban? Si no, hay que medir también al registrar el contexto, lo que obliga a descifrar las emergencias abiertas en ese momento.
- [ ] ¿500 m es el radio correcto? ¿Hace falta un radio por tipo de evento o por capacidad?
- [ ] ¿Cómo entra una alerta climática: por barrio, como antes, o con otra forma de zona?

## Historial

- 2026-11-12: creación del ADR (estado: propuesta). Documenta el cambio que ya está en producción (PR #96 de `emergencias-backend`) y reemplaza al ADR por barrio.
