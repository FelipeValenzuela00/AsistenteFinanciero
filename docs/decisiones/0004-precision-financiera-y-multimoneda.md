# 0004 — Precisión financiera y multimoneda

**Estado:** aceptada · implementada (la cotización automática desde el BCRA está planificada)

## Contexto

Las cuentas pueden estar en ARS o USD, la cotización cambia a diario y los reportes tienen que consolidar todo en
una sola moneda sin que el pasado "se mueva" cuando sube el dólar.

## Decisión

- Todos los montos son `decimal` en .NET y `numeric(18,2)` en PostgreSQL; **nunca** `float`/`double`.
- El monto se guarda siempre positivo; el signo lo da el tipo de movimiento.
- Cada transacción guarda su **moneda**, la **cotización usada** y el **monto en moneda base (ARS)**, calculados
  por el backend en el momento de registrarla con la cotización de la fecha.
- Las cotizaciones vienen de una abstracción (`IExchangeRateProvider`); hoy desde la base, a futuro desde un job
  diario contra el BCRA.
- La moneda de una cuenta se puede cambiar: el saldo se convierte con la cotización y los movimientos previos
  quedan bloqueados para no mezclar monedas.
- Las fechas sin hora elegidas en la UI viajan a las 12:00 UTC para que ninguna zona horaria las corra de día.

## Por qué

- `decimal` evita errores de redondeo binario (0.1 + 0.2 ≠ 0.3 en coma flotante).
- Guardar la cotización histórica hace que los reportes de meses pasados sean reproducibles.
- Que el backend calcule el monto en moneda base evita que cada cliente lo haga distinto.

## Consecuencias

- Si no hay cotización para una operación en USD, se rechaza (409) en lugar de inventar un valor.
- En transferencias entre monedas con monto manual, la equivalencia en ARS usa la cotización implícita de la
  operación, no la del sistema.
