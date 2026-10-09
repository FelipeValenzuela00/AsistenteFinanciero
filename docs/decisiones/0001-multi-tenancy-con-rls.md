# 0001 — Multi-tenancy en una sola base con Row-Level Security

**Estado:** aceptada · aislamiento por servicio en producción; enforcement de RLS en curso

## Contexto

Cada organización (empresa o profesional) es un *tenant* y sus datos financieros no pueden mezclarse jamás con
los de otro. Las opciones habituales:

| Opción | Aislamiento | Costo operativo | Migraciones |
|---|---|---|---|
| Una base por tenant | Máximo | Alto (N bases, N conexiones, N backups) | N veces |
| Un schema por tenant | Alto | Medio | N veces |
| **Base y schema compartidos + `tenantId` + RLS** | Alto si RLS está forzada | Bajo | Una vez |

## Decisión

Base y schema compartidos: todas las tablas con datos de clientes tienen `TenantId` (UUID) y una política de
PostgreSQL Row-Level Security:

```sql
ALTER TABLE "Transactions" ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON "Transactions"
  USING ("TenantId" = current_setting('app.current_tenant_id')::uuid);
```

Un middleware (`TenantResolver`) resuelve el tenant desde la organización del JWT (lo crea la primera vez que
aparece) y fija `app.current_tenant_id` antes de cualquier acceso a datos.

## Por qué

- **Defensa en profundidad**: aunque un servicio olvide un `WHERE TenantId = ...`, la base no devuelve filas de
  otro tenant. El error humano más común en SaaS multi-tenant queda cubierto a nivel de motor.
- **Operación simple**: una base, un pool de conexiones, un backup y una migración por despliegue.
- **Escala suficiente** para el volumen esperado (miles de tenants con volúmenes moderados).
- **UUIDs** en todas las claves: sin colisiones entre tenants y sin IDs enumerables.

## Consecuencias

- Para que RLS sea efectiva, la aplicación debe conectarse con un rol **sin privilegios de dueño de las tablas**,
  las tablas deben usar `FORCE ROW LEVEL SECURITY` y el tenant debe fijarse con `SET LOCAL` dentro de cada
  transacción (con *connection pooling*, un `SET` de sesión puede filtrarse a otra request). Ese endurecimiento es
  parte de la fase de seguridad en curso; mientras tanto, el aislamiento está garantizado por la capa de
  aplicación y cubierto por tests (un recurso de otro tenant responde 404).
- Los índices únicos incluyen `TenantId` (por ejemplo, el nombre de una cuenta es único por tenant).
