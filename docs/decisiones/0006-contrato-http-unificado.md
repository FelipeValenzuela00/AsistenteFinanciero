# 0006 — Contrato HTTP unificado

**Estado:** aceptada · implementada

## Contexto

Durante el desarrollo, cada endpoint respondía distinto: algunos devolvían el objeto suelto, otros un envoltorio;
`DELETE` devolvía 204 en unos y 200 en otros; los enums viajaban como números y los errores de validación no
tenían forma fija. El frontend terminaba con casos especiales por endpoint y los tests no detectaban las
diferencias.

## Decisión

Un único contrato, documentado y verificado con tests de contrato que inspeccionan el JSON crudo:

| Caso | HTTP | Cuerpo |
|---|---|---|
| Recurso | 200 / 201 | `{ "success": true, "data": { ... } }` |
| Listado | 200 | `{ "success": true, "data": [ ... ], "meta": { page, pageSize, total, totalPages, hasNext, hasPrev } }` |
| Sin resultado | 200 | `{ "success": true }` |
| Error | 400–500 | `{ "success": false, "error": { code, message, details?, context? } }` |

- Códigos de error cerrados: `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`,
  `INTERNAL_ERROR`.
- `details`: errores por campo `[{ field, errors[] }]`. `context`: datos estructurados para que la UI explique el
  error (por ejemplo, el detalle de un saldo insuficiente).
- Enums como texto `UPPER_SNAKE` (`CREDIT_CARD`, `TRANSFER_OUT`); los números se rechazan.
- Ediciones con `PATCH` de semántica estándar: campo ausente = sin cambios, `null` = limpiar.
- Paginación uniforme con `page` y `pageSize` (1..100).

## Por qué

- El frontend tiene **un solo cliente HTTP** que valida el formato y convierte cualquier error en un `ApiError`
  tipado.
- Los controllers no arman respuestas a mano: usan helpers de una clase base, así el contrato no puede divergir.
- Los tests de contrato fallan si alguien cambia la forma de una respuesta.

## Consecuencias

- Todo cambio de ruta, DTO o forma de respuesta se registra en el contrato y en un registro cronológico de cambios
  estructurales.
