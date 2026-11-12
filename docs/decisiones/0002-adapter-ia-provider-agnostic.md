# 0002 — Adapter de IA provider-agnostic

**Estado:** aceptada · implementada (proveedor de producción en evaluación)

> **Actualización (noviembre de 2026).** La implementación quedó más simple que el diseño de abajo: la interfaz
> tiene un solo método (`GenerateAsync`) y hay tres adapters: `AnthropicProvider`, `OpenAiCompatibleProvider` (que
> cubre Gemini, NVIDIA NIM, OpenAI y Ollama con solo cambiar URL y modelo) y `MockAiProvider` para tests. El proveedor
> se elige por configuración para toda la instalación (fábrica `AiProviders.Create`), no por tenant, y el presupuesto
> de IA es un tope global con contador por tenant. La IA se usa en el asistente (registrar movimientos y responder
> consultas); lo que registra va a la cuarentena, donde cuenta y categoría se asignan por coincidencia de nombres.
> El principio central sigue: la IA sugiere y el usuario aprueba.

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
