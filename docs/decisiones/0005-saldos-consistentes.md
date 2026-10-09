# 0005 — Saldos incrementales, atómicos y con bloqueo de filas

**Estado:** aceptada · implementada

## Contexto

El saldo de una cuenta es el dato más sensible del producto. Tiene que reflejar exactamente los movimientos, aun
cuando dos operaciones sobre la misma cuenta lleguen al mismo tiempo, y ningún cambio de saldo puede quedar sin
rastro.

## Opciones

| Opción | Lectura | Escritura | Riesgo |
|---|---|---|---|
| Calcular el saldo al leer (Σ movimientos) | Cara con muchos movimientos | Simple | Dashboards lentos |
| **Saldo materializado, actualizado en la misma transacción** | Barata | Requiere control de concurrencia | *Lost updates* si no se bloquea |

## Decisión

- `Account.Balance` se actualiza **de forma incremental dentro de la misma transacción de base** que el
  movimiento: alta (aplica), edición (revierte lo anterior y aplica lo nuevo), eliminación (revierte).
- Antes de modificar, se bloquean las filas involucradas con `SELECT ... FOR UPDATE`, **siempre en orden de Id**,
  para evitar deadlocks entre transferencias cruzadas.
- Un `BalanceChangeSet` acumula los efectos y valida al final: si una cuenta que no es tarjeta de crédito quedaría
  en negativo, se hace rollback completo y se responde **409 con un mensaje explicativo** y datos estructurados
  (`error.context`: cuenta, saldo actual, movimiento, saldo resultante).
- Los ajustes manuales se registran como movimientos `ADJUSTMENT_IN/OUT` y la cuenta guarda su saldo inicial
  aparte.
- Las transferencias son dos movimientos enlazados que se crean, editan y eliminan siempre juntos.

## Verificación

- **Invariante**: `Saldo = Saldo inicial + Σ movimientos que afectan el saldo y no están eliminados`, comprobado
  tras 50 operaciones aleatorias con semilla fija.
- **Concurrencia**: 20 gastos en paralelo sobre la misma cuenta, con y sin fondos suficientes; el resultado final
  es exacto.

## Consecuencias

- Lecturas baratas (dashboard y listados no recalculan nada).
- Cada escritura que toca saldos debe pasar por el `BalanceChangeSet`; tocar `Balance` directamente está
  prohibido por convención y lo cubren los tests.
- Cuando cambió una regla (archivar dejó de revertir saldos), una migración recalculó todos los saldos a partir
  del invariante.
