# ADR-0005: Agentes de IA con permisos delegados

Estado: **Propuesto**. Fecha: 2026-11-17.

## Contexto

Los agentes necesitan consultar y operar reservas, productos, ventas y CRM. Un modelo puede producir argumentos incorrectos o seguir instrucciones maliciosas, y su respuesta no constituye una autorización.

## Decisión propuesta

Módulo `ai` para agentes, conversaciones, mensajes, runs y herramientas. Un puerto abstrae proveedor/modelo. No se crea infraestructura de entrenamiento, RAG o almacenamiento vectorial.

Herramientas de un registro cerrado llaman los mismos contratos de aplicación que la API. Nunca acceden directamente a ORM, SQL, shell, red arbitraria ni credenciales. Contexto de tenant/actor queda fuera de los argumentos del modelo.

Permisos efectivos = permisos actuales del solicitante intersectados con herramientas del agente y capacidades iniciales del run. Las conversaciones son privadas de su propietario, también para personal. Revalidar membresía, identidad, tenant, agente y estado del recurso antes de cada herramienta.

Toda herramienta con efecto de escritura requiere aprobación humana visible de argumentos concretos, hash, versión, vencimiento y consumo único. Aprobar no amplía permisos. La clave de invocación deduplica el efecto local; pagos conservan además idempotencia de proveedor.

Runs asíncronos, un run activo por conversación y límites de pasos/tokens/duración/concurrencia. Persistir mensajes visibles y resultados, no razonamientos internos. [Modelo](../data-model.md), [REST](../rest-api.md) y [controles](../security-and-operations.md).

## Alternativas consideradas

- Conectar el LLM a PostgreSQL: rompe permisos de aplicación y aumenta superficie de fuga/escritura.
- Crear API interna privilegiada de IA: separa reglas y facilita escalamiento de permisos.
- Acciones autónomas sin aprobación: no hay políticas comerciales ni evidencia de riesgo aceptado para autorizarlo.

## Consecuencias

Se reutilizan invariantes y auditoría. Las aprobaciones añaden pasos a la experiencia y la ejecución puede fallar por cambios de estado tras aprobar. El proveedor recibe datos seleccionados: sus términos de retención/residencia requieren revisión antes de producción.

Prompt injection sigue siendo un riesgo residual; el sistema confía en permisos y validaciones externas al modelo, no en que el prompt pueda evitar todo abuso.

## Validación posterior a la revisión

Ataques de prompt injection, argumentos con tenant ajeno, ID de otro cliente, modificación/replay de aprobación, revocación antes de ejecución, agotamiento de presupuesto y reintento de herramienta. Confirmar proveedor y alcance de datos enviado.
