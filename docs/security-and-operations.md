# Seguridad, despliegue y escalabilidad

Estado: **propuesto para revisión**. Relacionado: [arquitectura](architecture.md), [aislamiento](multi-tenancy.md) y [contratos REST](rest-api.md). Los controles descritos son requisitos del diseño, no capacidades ya implementadas.

## 1. Límites de confianza

| Frontera | Riesgo | Control previsto |
| --- | --- | --- |
| Navegador -> API | Suplantación, datos manipulados y abuso | TLS, access token validado, esquemas estrictos, permisos y límites de petición |
| Usuario -> otra empresa | Lectura o escritura cruzada | Contexto validado, filtros, RLS y FK compuestas |
| Cliente -> otro cliente del mismo tenant | Acceso indebido a contactos, pedidos o reservas | Autorización por contacto/propietario en cada caso de uso |
| API/worker -> PostgreSQL | SQL injection, fuga por pool, exceso de privilegios | Consultas parametrizadas, transacciones locales y roles mínimos |
| Proveedor -> webhook | Eventos falsos, repetidos o desordenados | Firma, ventana temporal, cuenta validada, inbox durable y deduplicación |
| Modelo IA -> herramientas | Instrucciones maliciosas y ampliación de privilegios | Registro cerrado, validación, autorización delegada y aprobación humana |
| Runtime -> host/secretos | Credenciales o datos expuestos | Contenedores sin root, secretos por archivo, acceso operativo restringido |

RLS es una barrera contra errores de acceso entre empresas; no protege frente a un proceso backend completamente comprometido. El runtime sigue siendo una parte confiable que puede establecer contexto de sesión.

## 2. Autenticación y autorización

Se propone un proveedor OIDC externo. Frontends de navegador usan Authorization Code con PKCE; refresh y logout se delegan al proveedor. El uso de PKCE y la exclusión de flujos inseguros siguen las recomendaciones de [RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.html). La elección del proveedor y su configuración exacta requieren revisión.

- La API acepta access tokens destinados a su audience, con issuer y algoritmos permitidos, firma, expiración y tolerancia de reloj acotada. Nunca confía en un JWT solo decodificado ni recibe un ID token en lugar del access token.
- JWKS se obtiene de un issuer configurado, con caché acotada y soporte de rotación. Un `kid` desconocido permite una actualización limitada, no llamadas ilimitadas dirigidas por el atacante.
- Tokens de navegador se mantienen en memoria; no se prescriben refresh tokens en `localStorage`. Si se añade un BFF o autenticación por cookie, se deberán revisar CSRF, `HttpOnly`, `Secure` y `SameSite`; no se introduce ese componente ahora.
- Estado de usuario, membresía y tenant se comprueba en cada unidad de trabajo. Los roles de un token antiguo no sustituyen la membresía actual.
- Permisos por acción y propiedad del registro, con denegación por defecto. Un admin empresarial no puede administrar otra empresa ni obtener credenciales SQL de plataforma.
- MFA para owners, administradores y operadores de plataforma se exige en el proveedor elegido. Acceso de soporte con privilegios extraordinarios no se añade a la API comercial.

CORS permite orígenes exactos previamente registrados. En rutas empresariales se comprueba su asociación al tenant; `/me` usa la lista de frontends registrados. Incluir `Vary: Origin`; no usar comodines con credenciales. Las llamadas servidor a servidor pueden no tener `Origin` y siguen sujetas al mismo token y permisos. CORS no es una barrera de autorización.

## 3. Base de datos y secretos

| Rol/conexión | Facultades |
| --- | --- |
| `app_runtime` | DML necesario bajo RLS; sin propiedad, DDL, `BYPASSRLS`, `TRUNCATE` ni eliminación de auditoría |
| `job_dispatcher` | Solo ejecutar claim/renovación restringida de leases; sin SELECT libre de payloads |
| `webhook_resolver` | Solo resolver una cuenta concreta para verificación; sin acceso a pedidos |
| Migración | DDL y gestión de políticas; credencial disponible solo al job de migración |
| Control de plataforma | Provisionamiento/estado de tenant e identidades por operaciones auditadas; sin endpoint comercial genérico |
| Backup | Permisos operativos necesarios, acceso restringido y alcance separado del runtime |

Los propietarios dedicados de funciones privilegiadas no se usan para iniciar sesión. Los grants se crean explícitamente; se revocan privilegios innecesarios de `PUBLIC` y escritura/creación en esquemas públicos. No se usan las credenciales de superusuario inicial de PostgreSQL en la aplicación.

Credenciales de base, OIDC confidenciales cuando correspondan, pagos e IA se suministran mediante archivos secretos con permisos del host. Las tablas guardan referencias, no claves en claro. Las contraseñas no se incluyen en imagen, Compose versionado, URLs registradas ni archivos de ejemplo con valores reales. Rotación documentada y entornos con secretos distintos.

Compose permite montar secretos por servicio; el host sigue debiendo proteger los archivos de origen. Es el mecanismo descrito en la [documentación de secretos de Docker Compose](https://docs.docker.com/compose/how-tos/use-secrets/), no una garantía de cifrado del disco del host.

## 4. Protección de API e integraciones

- Esquemas de escritura separados de lectura; lista explícita de campos, tamaños y tipos. La normalización de SKU, slug y email no se deja a cada frontend.
- Límites iniciales: body JSON general de 1 MiB, mensajes IA de 32 KiB, páginas de 100 elementos y disponibilidad hasta 31 días. Límites específicos de webhook se fijan con el proveedor.
- Proxy único aplica límites por IP y tamaño; aplicación limita trabajos/runs activos por tenant y por usuario. Contar/reservar capacidad en PostgreSQL con bloqueo del tenant impide que réplicas excedan cuotas concurrentes. No hay límites críticos basados solo en memoria de cada proceso.
- Consultas con timeout, pool y concurrencia acotados; claves y filtros no se convierten en SQL dinámico.
- URL de checkout y endpoints de proveedor provienen de configuración permitida. La entrada de usuario o del modelo no determina hosts de red; no hay proxy HTTP arbitrario.
- Webhooks verifican firma sobre el body original antes de persistir un evento aceptado. Las redirecciones del navegador nunca confirman pago.
- Idempotencia de API, intentos externos y eventos como capas separadas. Reintentar ciegamente después de un timeout de pago está prohibido; se consulta el intento existente.
- Fijar dependencias e imágenes a versiones verificadas al implementar, revisar vulnerabilidades y actualizar parches mediante el ciclo normal de publicación. No se seleccionan números de versión desde este diseño como si fueran permanentes.

## 5. Agentes de IA

Las respuestas y documentos del modelo se consideran datos no confiables. Las defensas de separación de instrucciones, validación de herramientas y revisión humana se apoyan en la [guía de prevención de prompt injection de OWASP](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html).

1. Registro de herramientas permitido en código, con esquemas estrictos y mapeo a contratos de aplicación. El agente no crea herramientas ni ejecuta código, shell o SQL.
2. Contexto de tenant y usuario fijado fuera del modelo. Los argumentos no pueden cambiarlo, elevar rol ni elegir credenciales.
3. Lecturas respetan los permisos del solicitante, incluida propiedad de registros. No se entrega al modelo la base completa ni conversaciones de otras personas.
4. Toda herramienta que crea, cancela, cobra o modifica exige una propuesta visible y aprobación humana de sus argumentos exactos. Quien aprueba debe ser propietario de la conversación y tener permiso de la acción; la aprobación no amplía el permiso del solicitante.
5. Hash de argumentos, versión, caducidad y consumo único impiden reutilizar o cambiar una aprobación. Un cambio de argumentos genera otra propuesta.
6. Revalidar permisos, membresía, tenant, agente activo y estado del recurso al ejecutar. La aprobación no garantiza que siga habiendo stock o disponibilidad.
7. Limitar pasos, tokens, duración, concurrencia y tamaño de resultados. `max_tokens` es presupuesto total del run; antes de cada llamada se calcula el saldo y se limita la solicitud al proveedor. Al agotarlo, detener el run y registrar un código estable.
8. Enviar únicamente datos necesarios al proveedor; redactar secretos y PII ajena al propósito. Retención, uso para entrenamiento, residencia y términos del proveedor son decisiones pendientes antes de usar datos reales.
9. Auditar herramientas y resultados sanitizados; no registrar razonamientos internos del modelo ni prompts completos en logs generales. El frontend escapa texto y sanitiza Markdown antes de renderizarlo.

La mitigación reduce riesgos de prompt injection pero no presume eliminarlos. Una operación financiera sigue necesitando validación comercial, idempotencia y confirmación del proveedor, aunque la haya iniciado una herramienta aprobada.

## 6. Docker Compose previsto

No se crea `compose.yaml` ni Dockerfile hasta aprobar esta arquitectura. La topología propuesta es:

| Servicio Compose | Función | Red/persistencia |
| --- | --- | --- |
| `proxy` | TLS, límite de body/IP y enrutamiento a API | Único componente con puertos públicos 80/443 |
| `api` | FastAPI sin estado de sesión local | Red del proxy y red privada de base; salida controlada a OIDC |
| `worker` | Jobs PostgreSQL del mismo monolito | Red privada de base y salida a pagos/IA; sin puerto público |
| `db` | PostgreSQL primario | Solo red privada interna; volumen persistente; sin puerto público |
| `migrate` | Una ejecución Alembic por publicación | Misma imagen; acceso privado a base y secreto exclusivo de migración |

API, worker y migración usan la **misma imagen**. No hay despliegues independientes de CRM, reservas, comercio o IA. Una red adicional de salida de API/worker permite proveedores externos; pertenecer a la red interna de base no publica PostgreSQL.

Desarrollo puede montar código y publicar PostgreSQL únicamente en localhost con credenciales locales. Producción usa imagen inmutable, sin autoreload ni montajes del código fuente; contenedores de aplicación sin root, límites CPU/memoria, filesystem de solo lectura donde sea posible, capacidades reducidas y sin socket Docker.

Secuencia: base saludable -> migración única finalizada -> API/worker. `depends_on` necesita condiciones de salud/finalización; levantar un contenedor no significa que esté listo, como explica [Docker Compose](https://docs.docker.com/compose/how-tos/startup-order/). La aplicación también reconecta con espera acotada tras caídas de PostgreSQL; el orden de arranque no resuelve fallos posteriores.

Un bloqueo de publicación evita migraciones simultáneas. No ejecutar migraciones desde cada réplica de API. Cambios incompatibles requieren ventana de mantenimiento; cambios compatibles siguen expandir/migrar/contraer. Un rollback de imagen solo es posible si el esquema sigue siendo compatible; no se asume que un downgrade destructivo sea seguro.

## 7. Worker y entrega durable

La fila de job funciona como intención transaccional de entrega, sin broker adicional. Claim de lotes pequeños con `FOR UPDATE SKIP LOCKED`, cambio de estado y lease en una transacción corta; PostgreSQL contempla `SKIP LOCKED` para evitar contención entre consumidores de colas en [SELECT](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE).

- Una vez reclamado, el worker abre contexto runtime del tenant y carga el payload. Cada handler acepta solo su esquema y capacidades.
- Renueva el lease durante tareas largas. Cada transacción de escritura bloquea la fila del job y verifica propietario, vigencia y generación del lease antes de cambiar negocio o confirmar resultados. La generación creciente impide que un worker cuyo lease expiró confirme resultados tardíos.
- Una caída deja el job recuperable al vencer el lease. Reintentos con backoff, jitter y máximo de intentos; al agotarse pasa a `dead` y genera alerta.
- Mutaciones locales y finalización de su paso se confirman juntas. Las llamadas remotas se realizan fuera de transacción con claves estables. Dos entregas pueden invocar un proveedor dos veces; su idempotencia evita duplicar efectos. En IA puede repetirse coste de generación tras un timeout, aunque no se repitan mutaciones locales.
- Vencimiento de pedidos se agenda al crearlos usando `available_at=expires_at`. No hace falta otro servicio de scheduler para ese caso.
- Ordenar por antigüedad y limitar trabajos activos por tenant/tipo reduce monopolización. La política de claim consulta capacidad disponible y reserva el cupo con bloqueo del tenant; protege que IA no bloquee vencimientos o pagos.
- Jobs muertos se inspeccionan y reencolan operativamente con la misma clave; no se borran para forzar un segundo cobro. Retención/purga se fija según operación y conciliación.

## 8. Observabilidad y datos personales

Logs estructurados con `request_id`, tenant conocido, actor, operación, duración y código de resultado. Acceso solo operativo; los tenants consultan su auditoría filtrada. Denegaciones y fallos anteriores a commit van a logs de seguridad; las mutaciones aceptadas guardan auditoría en su misma transacción.

Métricas: tasa/latencia/error HTTP, ocupación del pool, conexiones, locks, consultas lentas, antigüedad de jobs, leases vencidos, jobs muertos, pagos pendientes/conciliación y tokens/coste IA. Evitar etiquetas de alta cardinalidad como tenant, usuario o request. IDs detallados se investigan en logs restringidos.

Liveness solo comprueba proceso. Readiness comprueba conexión y revisión compatible de esquema, sin leer negocio sin contexto. Un fallo del proveedor de IA no derriba las rutas de reservas. Worker publica heartbeat operativo; no se deduce su salud de que API responda.

Prompts, conversaciones, emails y payloads de proveedor son datos sensibles. Cifrado de volúmenes y backups, TLS fuera del host, acceso mínimo y política de retención por categoría. Se deben acordar jurisdicción, plazos, exportación y anonimización antes de producción; este documento no afirma cumplimiento legal específico.

## 9. Recuperación y disponibilidad

Docker Compose en un servidor es la base inicial de despliegue, coherente con su [documentación de uso en producción](https://docs.docker.com/compose/how-tos/production/). Reiniciar contenedores no sustituye backup ni tolerancia a caída del host.

- Backups cifrados fuera del servidor. Para un RPO de minutos se requieren archivado WAL y recuperación a punto en el tiempo, además de backups base; un dump diario no lo cumple.
- Objetivos iniciales para discusión: **RPO <= 15 minutos y RTO <= 4 horas**. No son garantías hasta validar almacenamiento, alertas, personal y restauraciones reales.
- Ensayar restauración en entorno aislado, aplicar esquema compatible, verificar integridad/RLS y conciliar pagos/jobs con proveedores antes de reanudar cobros. Conservar el historial necesario de claves para no repetir efectos después de restaurar.
- Hacer una restauración completa antes de producción y periódicamente después. Medir tiempo real y pérdida máxima de datos, con responsable y procedimiento operativo.
- Recuperar un único tenant requiere restaurar una copia aislada y extraer datos respetando FK; la base compartida no permite restaurarlo directamente con un simple comando de Compose. Exportación/importación por tenant no se implementa en esta etapa.

## 10. Escalabilidad dentro del monolito

1. Comenzar con una API, un worker y PostgreSQL; medir carga por tipo de operación. Reducir N+1, cargar relaciones explícitamente, paginar y revisar planes antes de añadir infraestructura.
2. Dimensionar pools y memoria. Presupuesto de conexiones: `replicas_api * procesos_api * (pool_size + max_overflow) + conexiones_worker + conexiones_operativas < max_connections - reserva`. Incluir pools pequeños de dispatcher/resolver y migración.
3. Aumentar concurrencia limitada de API y worker según CPU, conexiones, locks y latencia de proveedores. Async no elimina límites de base ni permite compartir `AsyncSession`.
4. Replicar procesos de la misma imagen detrás del proxy si el host tiene capacidad. La API es stateless, los jobs se coordinan en PostgreSQL y no se usan locks de proceso para consistencia.
5. Escalar verticalmente o ubicar PostgreSQL en un servidor/servicio dedicado sin cambiar los módulos. Réplicas de lectura o pooler se evalúan con mediciones y con pruebas nuevas de consistencia/contexto; no forman parte del despliegue inicial.

No se añaden Redis, broker, particionado, motor vectorial ni orquestador. Los límites previsibles son I/O y conexiones de PostgreSQL, contención de recursos calientes, cuota del proveedor y el host único. La pérdida completa del servidor requiere recuperación, no failover automático.

Antes de producción se debe probar carga mixta de varios tenants, un tenant ruidoso, reservas concurrentes, compra del último stock, reentrega de webhooks, muerte de worker y restauración. Los umbrales de latencia/carga se fijarán con el volumen esperado; esta entrega no presenta benchmarks.
