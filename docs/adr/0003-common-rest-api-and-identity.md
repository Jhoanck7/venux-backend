# ADR-0003: API REST común e identidad OIDC

Estado: **Propuesto**. Fecha: 2026-11-17.

## Contexto

Los frontends serán personalizados, pero los datos, reglas y permisos deben tener una interpretación común. Un usuario puede trabajar con varias empresas y también ser cliente de alguna de ellas.

## Decisión propuesta

REST JSON bajo `/api/v1`, con recursos empresariales en `/tenants/{tenant_id}`. Paginación por cursor, errores Problem Details, ETag/If-Match e idempotencia en comandos. El contrato se mantendrá en OpenAPI mediante FastAPI/Pydantic v2 cuando se autorice código. [Contratos y permisos](../rest-api.md).

Identidad externa OIDC, Authorization Code con PKCE para navegador y access token dirigido a la API. La identidad local se vincula por `(issuer, subject)`; roles pertenecen a membresías de tenant y se revalidan en aplicación. Proveedor concreto y onboarding aún requieren revisión.

Cuatro roles iniciales: owner, admin, staff y customer, traducidos a permisos por acción. Clientes solo ven datos propios; frontend, email, hostname y contenido de prompts no acreditan propiedad. Frontends personalizan experiencia; el backend no contiene bifurcaciones comerciales por nombre de empresa.

El alcance inicial exige autenticación. No se define todavía un storefront anónimo ni una API pública de alta de empresas. Orígenes y clientes OIDC se registran por control de plataforma.

## Alternativas consideradas

- API distinta por frontend/empresa: duplica semántica y aumenta divergencia.
- Confiar en tenant/rol enviados por el navegador: convierte datos manipulables en autorización.
- Contraseñas y refresh locales: amplía responsabilidades de identidad sin ser necesario para el diseño inicial.

## Consecuencias

Los frontends evolucionan visualmente con un contrato común. La identidad depende de un proveedor externo y su configuración; emitir tokens, recuperar cuentas y MFA no son capacidades locales de VENUX.

Cada cambio incompatible necesita revisión de clientes y posible versión nueva. Los campos de respuesta pueden crecer; los cuerpos de escritura siguen siendo estrictos para evitar asignación masiva.

## Validación posterior a la revisión

Pruebas de token inválido/expirado, issuer/audience incorrectos, roles revocados, acceso propio, ETag e idempotencia; snapshots o checks de compatibilidad OpenAPI. Confirmar proveedor y flujo de alta antes de implementarlos.
