# ADR-0004: Unidad de trabajo y jobs durables en PostgreSQL

Estado: **Propuesto**. Fecha: 2026-11-17.

## Contexto

Reservar el mismo recurso, vender el último stock o reintentar un pago exige consistencia. IA y pagos son lentos y pueden fallar después de producir un efecto externo. La solución inicial no necesita otro sistema de mensajería.

## Decisión propuesta

SQLAlchemy 2 asíncrono con una sesión por tarea/petición y una unidad de trabajo por comando. El caso de uso de entrada hace commit/rollback; los contratos internos participan sin confirmar por separado.

PostgreSQL protege solapamientos con exclusión de rangos, stock con bloqueos y límites, e idempotencia con unicidad. Cambios locales, auditoría y jobs se guardan en la misma transacción. No se llama a proveedores manteniendo transacciones abiertas.

Worker de la misma imagen reclama jobs mediante función restringida y `SKIP LOCKED`, con leases, generación, reintentos acotados y estado `dead`. Procesa una empresa por contexto. Los jobs son la intención durable; no se crea además un bus ni otra tabla de outbox con idéntica función.

Entrega al menos una vez: claves estables para operaciones locales/proveedor, recepción durable y deduplicación de webhooks. Timeouts de pago se concilian antes de crear otro intento. Pago tardío de pedido cancelado no recupera stock ni reactiva la venta. [Invariantes](../data-model.md) y [operación del worker](../security-and-operations.md).

## Alternativas consideradas

- Tareas en memoria o `BackgroundTasks` como garantía de entrega: se pierden al caer el proceso.
- Mantener la transacción durante IA/pago: ocupa conexiones y bloqueos mientras espera una red externa.
- Añadir broker y consumidores independientes: carga operativa no justificada en este alcance.

## Consecuencias

La entrega sobrevive al reinicio y mantiene atomicidad de intención local. Jobs, leases y deduplicación requieren handlers explícitos y operación de errores. La cola consume capacidad de PostgreSQL y debe vigilarse.

No hay garantía de exactamente una vez sobre redes externas. La integración elegida debe soportar idempotencia o consulta/reconciliación fiable; la generación IA puede repetir coste tras fallos sin repetir una mutación autorizada.

## Validación posterior a la revisión

Reservas simultáneas, última unidad, rollback entre pasos, POST duplicado, webhook repetido/desordenado, caída tras efecto remoto y expiración de lease. Probar que un worker obsoleto no confirma y que jobs muertos son visibles y recuperables.
