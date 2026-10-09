# Calidad y testing

[← Volver al README](../README.md)

## Números

| Suite | Tests | Herramientas |
|---|---|---|
| Backend: unitarios | 162 | xUnit, FluentAssertions, Moq |
| Backend: integración + contrato | 324 | xUnit, `WebApplicationFactory`, **Testcontainers (PostgreSQL 16 real)** |
| Frontend | 354 | Vitest, Testing Library, mock del adaptador HTTP |

Todo en verde, sin tests omitidos. CI ejecuta además `dotnet format`, analizadores con *warnings as errors*,
migraciones de EF Core contra PostgreSQL, lint y typecheck.

## Método

1. **Reglas de negocio numeradas** (D1–D24): por ejemplo, "D2: solo las tarjetas de crédito pueden quedar con
   saldo negativo" o "D21: una transferencia es un par atómico de movimientos".
2. **Catálogo de casos** con prefijos por módulo (`ACC` cuentas, `CAT` categorías, `TRX` transacciones, `BAL`
   saldos, `TRF` transferencias, `ADJ` ajustes, `DSH` dashboard, `ENV` contrato) y su resultado esperado.
3. Cada test lleva *traits* `Modulo` y `Caso`, así se puede correr un módulo o un caso puntual y rastrear qué regla
   cubre.
4. **Desarrollo por fases** (infraestructura de pruebas → cuentas → categorías → transacciones → saldos y
   transferencias → dashboard → seguridad → E2E). Cada fase cierra con tests en verde, **prueba manual guiada con
   una tabla de casos** y revisión de logs.

## Tests destacados

- **Invariante de saldos**: 50 operaciones aleatorias (altas, ediciones, archivados, eliminaciones,
  transferencias, ajustes) con semilla fija; al final, cada saldo es exactamente el saldo inicial más la suma de
  sus movimientos.
- **Concurrencia real**: 20 gastos lanzados en paralelo contra la misma cuenta sobre PostgreSQL real; sin *lost
  updates* y con rechazo correcto de los que dejarían la cuenta en negativo.
- **Atomicidad**: si una transferencia falla por fondos insuficientes, no queda persistida ninguna de sus dos
  patas.
- **Aislamiento entre tenants**: cualquier recurso de otra organización responde 404.
- **Contrato**: los tests leen el JSON crudo y verifican el envoltorio, los códigos de error, que los enums sean
  texto y que no aparezcan campos inesperados.

## Bugs que encontró el proceso

La revisión de código y la prueba manual detectaron, entre otros:

- Transacciones que **no actualizaban el saldo** de la cuenta (resuelto con el modelo de saldos de la decisión
  0005).
- Un servicio **sin registrar en el contenedor de dependencias** que hacía fallar todo un módulo con un error
  engañoso.
- **Diferencias de contrato** entre frontend y backend (enums numéricos, `DELETE` con 204, `PUT` vs `PATCH`) que
  dejaban el dashboard en blanco.
- En transferencias entre monedas con monto manual, una **equivalencia en ARS calculada con la cotización del
  sistema** en lugar de la de la operación.
