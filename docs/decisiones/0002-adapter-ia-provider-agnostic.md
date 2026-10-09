# 0002 — Adapter de IA provider-agnostic

**Estado:** aceptada · interfaz y proveedor mock implementados; Gemini planificado

## Contexto

La IA interpreta mensajes libres ("pagué 15 lucas de nafta con la visa") y los convierte en movimientos
categorizados. El mercado de modelos cambia cada pocos meses en precio, calidad y disponibilidad, y algunos
clientes pueden exigir un modelo local por privacidad.

## Decisión

La lógica de negocio depende solo de una interfaz en la capa de aplicación:

```csharp
public interface IAiProvider
{
    string ProviderName { get; }
    Task<AiResponse> CompleteAsync(AiRequest request, CancellationToken ct);
    Task<decimal> EstimateCostAsync(AiRequest request);
    Task<bool> HealthCheckAsync(CancellationToken ct);
}
```

Cada proveedor (Gemini, OpenAI, Anthropic, Ollama local) es un adapter en infraestructura. Un registro
(`AiProviderRegistry`, patrón Factory/Strategy) elige el proveedor según la configuración de cada tenant. Los
prompts viven como plantillas versionadas (JSON), fuera del código.

## Por qué

- **Sin lock-in**: cambiar de proveedor es registrar otro adapter, sin tocar servicios ni tests de negocio.
- **Costo controlado por tenant**: `EstimateCostAsync` permite presupuestos y elegir el modelo más barato que
  cumpla.
- **Testeable**: `MockAiProvider` responde de forma determinística, así los tests no dependen de la red ni de un
  modelo no determinístico.
- **Resiliencia**: `HealthCheckAsync` habilita *fallback* a otro proveedor si uno está caído.
- **Gemini como default** por su salida estructurada nativa (JSON con schema) y su relación costo/rendimiento.

## Consecuencias

- La respuesta de la IA **nunca** crea transacciones directamente: pasa por una bandeja de cuarentena con un flag
  `needsReview` y el usuario aprueba. La IA sugiere; el usuario decide.
- Hay que normalizar las diferencias entre proveedores (formatos de salida estructurada, límites de tokens) dentro
  de cada adapter.
