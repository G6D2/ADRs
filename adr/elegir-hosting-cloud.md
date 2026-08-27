# Elegir hosting cloud

- Estado: propuesta
- Fecha: 2026-08-14
- Decisores: @valenfiumana, @juanimoli, @Salterm27
- Reemplaza a: —
- Reemplazada por: —

Propuesto en el issue [#3](https://github.com/G6D2/ADRs/issues/3).

## Contexto

Para disponibilizar nuestra aplicación necesitamos infraestructura en internet.
No podemos usar hosting privado porque el costo es prohibitivo: necesitamos algo
económico o incluso gratuito hasta que hagamos el Go Live.

## Decisión

- Vamos a usar **Render** para la primera entrega.
- Vamos a revisar la decisión cuando tengamos que integrarnos con el bus de EDA.
- Vamos a activarlo y dejarlo prendido desde el inicio de la clase del día de la
  entrega, para evitar el arranque en frío durante la demo.

## Alternativas consideradas

| Servicio | Pros | Contras |
|---|---|---|
| **Render** | Muy simple de usar, ideal para demos. CI/CD casi gratis. | El servicio se duerme. |
| Railway | Buena experiencia, ideal para demos. | Crédito mensual de prueba de USD 5. |
| Vercel | All-in-one para frontend y backend. Integraciones con distintos proveedores de bases de datos. | No es performante para escuchar eventos. |
| Vanilla AWS | Es el estándar de la industria. Gratis por 12 meses. | Curva de aprendizaje compleja; hay que desarrollar todas las cosas asociadas. |

## Consecuencias

Se hace más fácil:

- El manejo del free tier es sencillo.
- El pipeline de deployment es fácil.
- No requiere adopción: ya conocemos la herramienta.

Se hace más difícil:

- Cada 15 minutos se apaga la máquina y tarda entre 30 y 60 segundos en
  prenderse, lo que obliga a encenderla con anticipación antes de cada demo.

## Historial

- 2026-08-14: creación del ADR (estado: propuesta)
