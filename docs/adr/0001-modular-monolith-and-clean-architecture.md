# ADR-0001: Monolito modular y Clean Architecture

Estado: **Propuesto**. Fecha: 2026-11-17.

## Contexto

VENUX combina CRM, catálogo, reservas, comercio e IA. Necesita reutilizar reglas desde varios frontends y mantener límites claros sin asumir el coste operativo de despliegues de negocio independientes. El repositorio todavía no tiene código de aplicación.

## Decisión propuesta

Un repositorio, una imagen y un ciclo de publicación, con procesos API y worker del mismo monolito. Módulos `identity`, `tenancy`, `crm`, `catalog`, `bookings`, `commerce` y `ai`; infraestructura común en `platform` y tipos/puertos puros mínimos en `kernel`.

Cada módulo separa `domain`, `application`, `infrastructure` y `presentation`. Dominio y aplicación no dependen de frameworks. FastAPI/Pydantic quedan en transporte; SQLAlchemy y SDK en adaptadores. Un punto de composición conecta dependencias.

Los módulos consumen contratos públicos de aplicación y no importan ORM, repositorios o entidades ajenos. Dependencias: tenancy -> identity; bookings -> CRM; commerce -> CRM/catalog; IA -> contratos de negocio. Identidad y tenancy se resuelven en la composición; los casos de uso reciben contexto/autorización por puertos. [Grafo y reglas completos](../architecture.md).

## Alternativas consideradas

- Organizar solo por carpetas globales `models/services/routes`: facilita comenzar, pero diluye propiedad y límites de negocio.
- Compartir modelos ORM y acceso libre a tablas: simplifica joins iniciales, pero acopla módulos y vuelve frágiles sus invariantes.
- Despliegues por dominio: fuera de lo solicitado y sin necesidad demostrada.

## Consecuencias

Permite transacciones locales y reutilización desde API/IA. Exige conversiones entre transporte, dominio y persistencia, contratos explícitos y revisión del grafo. Un fallo de proceso puede afectar al monolito y cada publicación incluye todos sus módulos.

Las FK entre módulos permanecen como contratos de datos de infraestructura; no habilitan acceso directo a tablas ajenas. No se crean módulos futuros de campañas, logística, notificaciones o búsqueda.

## Validación posterior a la revisión

Pruebas de dominio sin frameworks, checks de importación/ciclos y pruebas de integración de contratos cruzados. La estructura física se creará solo cuando se autorice implementación.
