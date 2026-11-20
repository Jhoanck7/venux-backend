# ADR-0006: Docker Compose y escalado del monolito

Estado: **Propuesto**. Fecha: 2026-11-17.

## Contexto

Se solicita Docker Compose y una arquitectura operable sin Kafka, Kubernetes ni microservicios. Todavía no existen métricas de tráfico que justifiquen infraestructura adicional.

## Decisión propuesta

Un servidor inicial con proxy HTTPS, API, worker, PostgreSQL y tarea de migración de una sola ejecución. API/worker/migración usan una imagen y versión comunes. Solo proxy publica puertos; base en red privada y volumen persistente.

Secretos por archivo con acceso mínimo, roles separados y contenedores de aplicación sin root. Healthchecks y revisión de migraciones condicionan arranque; reintentos acotados gestionan desconexiones posteriores. Backups cifrados fuera del host y restauraciones ensayadas son requisitos de producción.

Escalado por medición: consultas/índices, pools y concurrencia limitada; después réplicas de los mismos procesos y mayor capacidad de PostgreSQL. No hay locks en memoria para invariantes compartidas. [Topología y límites operativos](../security-and-operations.md).

## Alternativas consideradas

- Un proceso sin worker durable: bloquea solicitudes largas o pierde tareas.
- Añadir caché, broker y orquestador desde el inicio: sin necesidad comprobada y fuera del alcance solicitado.
- Base gestionada/dedicada: opción operativa posterior compatible con los módulos; no se exige para el primer entorno.

## Consecuencias

Desarrollo y publicación comparten topología sencilla. El host y el PostgreSQL primario son puntos de fallo; Compose no aporta failover entre servidores. Más réplicas en el mismo host mejoran capacidad, no resuelven esa disponibilidad.

Presupuesto de conexiones, jobs y recursos por tenant limita el escalado. RPO de 15 minutos y RTO de 4 horas son objetivos propuestos, pendientes de acuerdo y ensayo; no garantías de esta documentación.

## Validación posterior a la revisión

Arranque limpio, migración fallida, reinicio de base/API/worker, carga de varios tenants, saturación del pool y restauración externa. Fijar parches/dependencias y objetivos de carga antes de producción. No se generan archivos de despliegue hasta aprobar el diseño.
