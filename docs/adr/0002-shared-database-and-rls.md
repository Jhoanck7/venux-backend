# ADR-0002: Base compartida con tenant explícito y RLS

Estado: **Propuesto**. Fecha: 2026-11-17.

## Contexto

Varias empresas comparten backend. Un filtro omitido, una referencia manipulada o una conexión reutilizada pueden revelar datos de otra empresa. Se busca aislamiento comprobable sin administrar una instalación por cliente.

## Decisión propuesta

Una base PostgreSQL con esquemas por módulo y filas empresariales con `tenant_id NOT NULL`. Filtros explícitos en todos los repositorios, claves únicas por tenant y FK `(tenant_id, id)` entre entidades empresariales. RLS con `USING/WITH CHECK`, habilitada y forzada; rol runtime sin propiedad ni bypass.

El tenant se selecciona en la URL y se valida contra identidad y membresía actuales. Contexto SQL local a cada transacción, nunca estado global ni configuración persistente del pool. El contexto ausente no concede acceso.

Identidades y registro de empresas son globales con políticas especiales; lectura de membresías propias habilita `/me/tenants`. Control de plataforma, claim de jobs y resolución de webhooks tienen privilegios separados y acotados. [Políticas, excepciones y límite de confianza](../multi-tenancy.md).

## Alternativas consideradas

- Base por tenant: mayor aislamiento físico y recuperación individual, pero más conexiones, migraciones y operación.
- Esquema por tenant: multiplica objetos/migraciones y complica resolución/pooling.
- Solo filtros ORM: un error de repositorio deja sin segunda barrera el aislamiento.

## Consecuencias

Simplifica migraciones y operación inicial. La base compartida tiene recursos y recuperación compartidos: un tenant ruidoso afecta capacidad y restaurar uno requiere un proceso específico.

RLS no sustituye permisos por acción/propietario ni contiene una credencial SQL o proceso comprometido capaz de fijar otro contexto. Las excepciones privilegiadas requieren revisión de grants y funciones, no un rol global de lectura para API o worker.

## Validación posterior a la revisión

PostgreSQL real con roles runtime: acceso cruzado, contexto ausente, INSERT/UPDATE con FK ajena, pool tras rollback, concurrencia, rutas propias y las tres excepciones técnicas. Verificar catálogo RLS/FORCE y privilegios en cada migración.
