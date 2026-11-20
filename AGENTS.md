
# VENUX Backend

Plataforma empresarial modular y multi-tenant.

## Stack
- Python, FastAPI y Pydantic v2
- PostgreSQL, SQLAlchemy 2 y Alembic
- Pytest

## Arquitectura
- Monolito modular
- Capas: domain, application, infrastructure, presentation
- El dominio no depende de frameworks
- Casos de uso para lógica de negocio
- Un backend para múltiples empresas
- Aislamiento estricto mediante tenant_id

## Reglas de trabajo
- Implementar únicamente lo solicitado
- No crear módulos futuros innecesariamente
- No refactorizar código ajeno a la tarea
- Leer solo archivos relevantes
- Reutilizar patrones existentes
- Ejecutar pruebas relacionadas con los cambios
- Resumir resultados y archivos modificados
- No implementar funcionalidades adicionales sin autorización

## Documentación
Consultar docs/architecture.md cuando una tarea
afecte la arquitectura o los límites entre módulos.
