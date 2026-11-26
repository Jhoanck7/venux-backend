# Registro de decisiones de arquitectura

Todas las decisiones están **propuestas** con fecha 2026-11-17. Documentar una opción no significa aceptarla ni autorizar su implementación. La aceptación deberá registrar la revisión del usuario; si cambia una decisión aceptada, se conserva su historia y se añade un ADR que la sustituya.

| ADR | Decisión propuesta | Estado |
| --- | --- | --- |
| [0001](0001-modular-monolith-and-clean-architecture.md) | Monolito modular y Clean Architecture | Propuesto |
| [0002](0002-shared-database-and-rls.md) | Base compartida, tenant explícito, RLS y FK compuestas | Propuesto |
| [0003](0003-common-rest-api-and-identity.md) | API REST común, OIDC y autorización por empresa | Propuesto |
| [0004](0004-transactions-and-postgres-jobs.md) | Unidad de trabajo y jobs durables en PostgreSQL | Propuesto |
| [0005](0005-delegated-ai-agents.md) | Agentes de IA con permisos delegados y aprobación humana | Propuesto |
| [0006](0006-compose-deployment-and-scaling.md) | Docker Compose y escalado del mismo monolito | Propuesto |
| [0007](0007-sites-and-tenant-features.md) | Sitios públicos y habilitación de módulos por empresa | Propuesto |

Punto de entrada y decisiones comerciales pendientes: [arquitectura](../architecture.md).
