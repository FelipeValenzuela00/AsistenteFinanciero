# Arquitectura

[← Volver al README](../README.md)

## 1. Vista general

Dos aplicaciones desplegables y servicios externos gestionados:

| Pieza | Responsabilidad |
|---|---|
| **SPA React** | UI por componentes reutilizables (formularios, tablas, acciones y diálogos compartidos entre secciones), estado de servidor con React Query |
| **API ASP.NET Core 8** | Reglas de negocio, consistencia de saldos, contrato HTTP, multi-tenancy |
| **Worker** (planificado) | Cotización diaria del BCRA, exportaciones, enriquecimiento con IA |
| **PostgreSQL 16** | Única fuente de verdad, aislamiento por tenant con Row-Level Security |
| **Clerk** | Identidad, organizaciones (cada organización es un tenant), webhooks de sincronización |
| **Proveedor de IA** | Interpretación y categorización de movimientos, detrás de un adapter |

## 2. Capas del backend (Clean Architecture)

```mermaid
flowchart TB
    Api["<b>Api</b><br/>Controllers · Middleware · Program.cs<br/>Serialización del contrato"]
    Worker["<b>Worker</b><br/>Jobs en segundo plano"]
    Infra["<b>Infrastructure</b><br/>EF Core · Migraciones · Proveedores externos<br/>(cotizaciones, IA, bloqueo de filas)"]
    App["<b>Application</b><br/>Servicios · DTOs · Validadores<br/>Interfaces (IAiProvider, IExchangeRateProvider)"]
    Domain["<b>Domain</b><br/>Entidades · Enums · Reglas<br/><i>sin dependencias externas</i>"]

    Api --> Infra
    Api --> App
    Worker --> Infra
    Worker --> App
    Infra --> App
    App --> Domain
```

La regla de dependencias apunta siempre hacia el dominio. Las entidades protegen sus invariantes (por ejemplo,
`Account` decide si una cuenta puede quedar en negativo y `Transaction` valida montos y fechas), los servicios de
aplicación orquestan y la infraestructura implementa las interfaces.

## 3. Pipeline de una request

```mermaid
sequenceDiagram
    autonumber
    participant UI as React SPA
    participant JWT as JwtBearer
    participant C as Controller
    participant S as Servicio
    participant DB as PostgreSQL

    UI->>JWT: Authorization: Bearer <token de Clerk>
    JWT->>JWT: Valida firma (JWKS cacheado), issuer, audience, expiración
    JWT->>DB: Busca el tenant por la organización de Clerk (lo crea la primera vez; rol de sistema)
    JWT->>JWT: Agrega los claims tenant_id, user_id y rol
    JWT->>C: Usuario autenticado
    C->>S: DTO validado
    S->>DB: Al abrir la conexión, el interceptor fija app.current_tenant_id
    S->>DB: Consultas filtradas por TenantId (+ políticas RLS)
    S-->>C: Resultado
    C-->>UI: { success, data, meta? } o { success:false, error }
```

- **Errores 401/403** se escriben desde los eventos del autenticador con el mismo formato de error que el resto de
  la API.
- **El tenant y el creador salen siempre del token**: el cliente nunca puede mandar `tenantId` ni `createdById`.

## 4. Multi-tenancy

```mermaid
flowchart LR
    T1["Org A (token)"] --> MW
    T2["Org B (token)"] --> MW
    MW["Interceptor de EF Core<br/>set_config(app.current_tenant_id)"] --> Q["Consulta"]
    Q --> P{"Política RLS<br/>tenantId = current_setting(...)"}
    P -->|filas de A| RA["Datos de A"]
    P -->|filas de B| RB["Datos de B"]
```

Defensa en profundidad con dos capas:

1. **Aplicación**: cada servicio filtra por `TenantId`; un recurso de otra organización responde 404 (no 403, para
   no revelar que existe).
2. **Base de datos**: todas las tablas con datos de clientes tienen políticas RLS por `tenantId`.
   La API se conecta con un rol sin privilegios de dueño (`af_app`), las tablas tienen `FORCE ROW LEVEL SECURITY`
   y un interceptor de EF Core fija el tenant del token en cada conexión. Los flujos de sistema (login de Clerk,
   webhooks) usan un rol aparte con `BYPASSRLS`.

Ver la decisión [0001](decisiones/0001-multi-tenancy-con-rls.md).

## 5. Flujo de una transacción y consistencia de saldos

```mermaid
sequenceDiagram
    autonumber
    participant UI as React
    participant S as TransactionService
    participant L as RowLock
    participant FX as IExchangeRateProvider
    participant DB as PostgreSQL

    UI->>S: POST /transactions { cuenta, tipo, monto, fecha }
    S->>DB: BEGIN
    S->>L: SELECT ... FOR UPDATE (cuentas involucradas, ordenadas por Id)
    S->>FX: Cotización USD/ARS de la fecha (si la cuenta es USD)
    S->>S: amountBase = monto × cotización (moneda base ARS)
    S->>S: BalanceChangeSet: revertir efecto anterior / aplicar nuevo
    alt la cuenta no es tarjeta y quedaría en negativo
        S->>DB: ROLLBACK
        S-->>UI: 409 + mensaje explicativo + error.context
    else ok
        S->>DB: INSERT transacción + UPDATE saldo
        S->>DB: COMMIT
        S-->>UI: 201 { success, data }
    end
```

Reglas clave:

- `Saldo = Saldo inicial + Σ movimientos que afectan el saldo y no están eliminados`. Un test verifica este
  invariante tras 50 operaciones aleatorias.
- **Transferencias**: dos movimientos (`TRANSFER_OUT` + `TRANSFER_IN`) enlazados por `TransferId`, creados,
  editados y eliminados siempre juntos en una sola transacción de base. Entre monedas, el monto acreditado sale de
  la cotización o lo carga el usuario.
- **Ajustes manuales** de saldo quedan registrados como movimientos (`ADJUSTMENT_IN/OUT`): ningún cambio de saldo
  es invisible.
- **Archivar** solo ordena la vista; **eliminar** revierte el saldo. Cada movimiento tiene un checkbox «Afecta el
  saldo».
- **Concurrencia**: bloqueo pesimista de filas en orden determinístico (evita *lost updates* y deadlocks),
  verificado con 20 operaciones en paralelo.

Ver la decisión [0005](decisiones/0005-saldos-consistentes.md).

## 6. IA: adapter provider-agnostic y cuarentena

```mermaid
classDiagram
    class IAiProvider {
        <<interface>>
        +ProviderName string
        +GenerateAsync(request) AiCompletion
    }
    class AnthropicProvider
    class OpenAiCompatibleProvider
    class MockAiProvider
    class AiProviders {
        <<factory>>
        +Create(AiOptions) IAiProvider
    }
    IAiProvider <|.. AnthropicProvider
    IAiProvider <|.. OpenAiCompatibleProvider
    IAiProvider <|.. MockAiProvider
    AiProviders ..> IAiProvider
```

`OpenAiCompatibleProvider` cubre cualquier servidor con la API `/chat/completions` (Gemini, NVIDIA NIM, OpenAI,
Ollama): cambiar de modelo es cambiar `Ai:BaseUrl` y `Ai:Model`, sin código nuevo.

```mermaid
flowchart LR
    CH["Asistente (chat)"] --> IA["IAiProvider<br/>interpreta el mensaje"]
    IA --> Q["Bandeja de cuarentena"]
    CSV["CSV / extracto de<br/>Mercado Pago"] --> Q
    MK["WhatsApp/Telegram<br/>vía Make.com (en curso)"] --> W["POST webhook<br/>(idempotente por externalId)"] --> Q
    Q --> MT["Asigna cuenta y categoría<br/>por coincidencia de nombres"]
    MT --> R{"¿Necesita revisión?"}
    R -->|sí| U["Usuario revisa y corrige"]
    R -->|no| A
    U --> A["Aprobación individual o en lote<br/>(transacción ACID)"]
    A --> TX["Transacciones + saldos"]
```

Ver la decisión
[0002](decisiones/0002-adapter-ia-provider-agnostic.md).

## 7. Frontend

- **Arquitectura por componentes**: `TransactionFormModal`, `TransferModal`, `TransactionsTable`,
  `AffectsBalanceCheckbox` y el hook `useTransactionActions` se comparten entre la pantalla de Transacciones y el
  Dashboard, sin duplicar UI.
- **Estado de servidor** con React Query: query keys por tenant e invalidación centralizada (cualquier operación
  que mueva saldos refresca cuentas, transacciones y dashboard).
- **Cliente HTTP único** que valida el formato del contrato y transforma los errores en `ApiError` con detalles
  por campo y contexto estructurado.
- **Errores legibles**: los rechazos de negocio (por ejemplo, saldo insuficiente) se muestran completos dentro del
  formulario o en un diálogo, nunca en un toast que se corta.

## 8. CI/CD

```mermaid
flowchart LR
    PR["Pull request"] --> G["Guarda de ramas<br/>(a main solo develop o hotfix/*)"]
    PR --> CI["CI: build · dotnet format · analizadores (warnings as errors)<br/>migraciones EF contra PostgreSQL · tests unitarios e integración"]
    M["Merge a main"] --> CD["CD: imagen Docker → GHCR<br/>migraciones · health checks"]
```
