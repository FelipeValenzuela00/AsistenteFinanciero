# 0003 — Autenticación gestionada (Clerk) + validación JWT estándar

**Estado:** aceptada · implementada

## Contexto

El producto necesita login seguro, organizaciones con miembros y roles, invitaciones y MFA. Construir eso desde
cero es caro y riesgoso.

## Decisión

- **Clerk** gestiona identidad, sesiones y organizaciones. Cada organización de Clerk es un tenant.
- El backend **solo valida tokens** con el middleware estándar `AddJwtBearer`: descubre las claves públicas (JWKS)
  por OIDC, las cachea y valida firma, emisor, audiencia y expiración.
- Al validar, mapea los claims propios de Clerk (organización y rol) a claims estándar (`org_id`, `Role`) que usan
  las políticas de autorización.
- Los cambios en Clerk (usuarios, organizaciones, membresías) llegan por **webhooks firmados** (Svix, HMAC-SHA256)
  con protección contra *replay* e idempotencia por id de evento.
- Si un usuario autenticado todavía no existe en la base, se **auto-provisiona** desde el token en su primera
  operación.

## Por qué

- Seguridad delegada a un proveedor especializado; el backend no guarda contraseñas.
- Un único camino de validación, el estándar de ASP.NET Core, en lugar de validaciones manuales dispersas.
- Los webhooks mantienen la base sincronizada sin *polling*.

## Consecuencias

- El tenant y el usuario creador se toman **siempre del token**: el cliente no puede enviarlos.
- Los errores 401/403 se emiten desde los eventos del autenticador con el mismo formato de error que el resto de la
  API.
