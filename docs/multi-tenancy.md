# Aislamiento multi-tenant

Estado: **propuesto para revisión**. Relacionado: [arquitectura](architecture.md), [datos](data-model.md) y [ADR-0002](adr/0002-shared-database-and-rls.md).

## 1. Estrategia

Se elige una base PostgreSQL y tablas compartidas con `tenant_id`, no una base ni un esquema por empresa. Los esquemas pertenecen a módulos. Cada operación empresarial actúa sobre exactamente un tenant.

El aislamiento combina autorización en aplicación, filtros explícitos en repositorios, FK compuestas y Row-Level Security (RLS). RLS protege ante consultas que olviden un filtro; los permisos de aplicación protegen funciones y propiedad de registros dentro de la misma empresa.

Las políticas comparan el tenant de cada fila con el contexto transaccional. Se habilita `ENABLE ROW LEVEL SECURITY` y `FORCE ROW LEVEL SECURITY`; el rol runtime no es propietario, superusuario ni tiene `BYPASSRLS`. Estas diferencias de roles y la denegación sin políticas están descritas por [PostgreSQL](https://www.postgresql.org/docs/current/ddl-rowsecurity.html).

## 2. Contexto y resolución de una petición

`TenantContext` contiene `tenant_id`, `user_id`, `membership_id`, rol/capacidades efectivos y `request_id`. Es inmutable, explícito y se pasa al caso de uso; no se guarda en variables globales ni en una sesión compartida.

1. Validar el access token OIDC: firma, algoritmo permitido, issuer, audience de la API, expiración y tipo de token.
2. En una transacción corta de bootstrap, establecer con parámetros seguros `app.auth_issuer` y `app.auth_subject` desde el token validado. Resolver únicamente la identidad correspondiente y comprobar que está activa.
3. Establecer `app.user_id` a partir de esa identidad interna. Consultar membresías propias para `/me/tenants` o para validar el tenant seleccionado en la URL.
4. Para una operación empresarial, abrir la unidad de trabajo y establecer `app.tenant_id` y `app.user_id` como parámetros **locales a la transacción**, mediante `set_config(..., true)` o equivalente `SET LOCAL` seguro. Releer membresía activa y estado del tenant dentro de esa unidad de trabajo.
5. Autorizar la acción, resolver propiedad de registros cuando corresponda y ejecutar el caso de uso. Commit o rollback eliminan el contexto local antes de devolver la conexión al pool.

No hay un tenant predeterminado. Un contexto ausente o vacío no coincide con filas; un contexto inválido aborta la operación. No se emiten consultas empresariales antes de establecerlo. Si el caso de uso abre otra transacción, debe reestablecer y revalidar todo el contexto.

No se acepta `X-Tenant-ID`, body, query string, `Origin`, hostname ni texto del modelo de IA como fuente alternativa de autorización. La URL solo selecciona una empresa candidata. Si se usaran dominios personalizados en otro alcance, necesitarían un registro verificado y seguirían sujetos a membresía.

Una revocación impide nuevas unidades de trabajo. Una operación ya autorizada y en curso puede terminar; no se promete revocación retroactiva de transacciones. Antes de ejecutar un job o herramienta se abre y autoriza una nueva unidad de trabajo.

## 3. Políticas de filas y excepciones enumeradas

| Datos | Lectura | Escritura |
| --- | --- | --- |
| Tablas empresariales ordinarias | `tenant_id` coincide con `app.tenant_id` | `USING` y `WITH CHECK` exigen la misma coincidencia; el tenant es inmutable |
| `identity.users` | Identidad propia por issuer/subject verificados; después de bootstrap, por `app.user_id` confiable | Perfil propio con campos permitidos; issuer/subject/status se gestionan por control de identidad |
| `tenancy.memberships` | Tenant activo del contexto; adicionalmente, solo para SELECT, membresías del propio `app.user_id` sin contexto tenant | Únicamente tenant del contexto; rol y propiedad se validan en aplicación |
| `tenancy.tenants` | Tenant del contexto validado, o metadatos de empresas con membresía propia activa para bootstrap; permite consultar su estado suspendido | Configuración del tenant del contexto mediante caso de uso autorizado; alta/estado global solo control de plataforma |

La excepción de lectura de membresías permite `/me/tenants`; no aplica a contactos, reservas, pedidos ni conversaciones. Esa consulta comienza sin `app.tenant_id` y filtra además por `user_id`. La política de membresías no consulta `tenants`, evitando recursión con la política del registro de empresas.

Los roles de empresa se validan en la aplicación; un owner no obtiene permisos SQL distintos ni `BYPASSRLS`. RLS no sustituye la política de acceso propio de un cliente. El runtime no dispone de `TRUNCATE`, DDL, creación de funciones ni privilegios para alterar políticas.

Los parámetros de sesión son confiables porque los establece exclusivamente el backend. Esta estrategia **no** contiene a un atacante que controle el proceso, su credencial SQL o SQL arbitrario: podría cambiar esos parámetros. Consultas parametrizadas, privilegios mínimos, ausencia de SQL generado por IA y control de secretos son parte del límite de confianza.

### Excepciones técnicas restringidas

| Entrada | Necesidad | Límite obligatorio |
| --- | --- | --- |
| Alta y mantenimiento de plataforma | Crear tenant, identidad y primer owner; suspender empresa | Credencial operativa separada; sin endpoint comercial genérico ni credencial en API/worker; auditoría |
| Claim de jobs | Encontrar trabajo pendiente de cualquier empresa | Rol `job_dispatcher` solo ejecuta una función de claim; devuelve ID, tenant, tipo y generación de lease, nunca payload ni datos de negocio |
| Webhook de pagos | Resolver a qué tenant pertenece la cuenta antes de tener un usuario | Rol `webhook_resolver` solo ejecuta función que resuelve un ID de cuenta opaco a tenant/proveedor/referencia de secreto; no lista cuentas ni lee ventas |

Las funciones privilegiadas usan un propietario dedicado con políticas explícitas solo sobre las tablas necesarias, no superusuario ni `BYPASSRLS`. Deben fijar un `search_path` seguro, calificar objetos, evitar SQL dinámico y revocar `EXECUTE` a `PUBLIC`; esas precauciones corresponden a las [reglas de funciones SECURITY DEFINER de PostgreSQL](https://www.postgresql.org/docs/current/sql-createfunction.html#SQL-CREATEFUNCTION-SECURITY).

El dispatcher puede reclamar leases por esa función, pero no ejecutar consultas libres con privilegios sobre todas las tablas. El worker dispone de una conexión runtime distinta para cargar y procesar **un tenant por transacción**. El receptor de webhook verifica firma y cuenta antes de abrir una transacción empresarial; nunca devuelve los metadatos de resolución al remitente.

## 4. Integridad y consultas

- Todas las referencias empresariales usan `(tenant_id, foreign_id) -> (tenant_id, id)`, además del tenant obligatorio en la tabla hija.
- Todas las búsquedas por ID se realizan por `(tenant_id, id)`. No se carga globalmente para comprobar después el tenant.
- Listados, conteos, exportaciones y agregaciones aplican los mismos límites. No se permite una API de reportes que omita tenant.
- Unicidad de SKU, número de pedido y claves comerciales se define por tenant. La unicidad global de issuer/subject, slug y cuenta de pagos es una excepción operativa, sin endpoints de enumeración pública.
- Las comprobaciones de integridad referencial pueden revelar existencia a través de errores; las excepciones de base se traducen a errores de dominio sin nombres de tabla ni detalles de otras empresas. ID ajeno se presenta igual que ID inexistente.
- No se agregan vistas con privilegios del propietario que eludan RLS. Cualquier vista futura debe preservar las políticas del invocante y someterse a pruebas específicas.

## 5. Autorización dentro de la empresa

| Rol | Alcance inicial |
| --- | --- |
| `owner` | Todo el negocio, configuración de empresa y gestión de propietarios; no puede quitar al último owner |
| `admin` | Gestión de negocio, agentes, auditoría y membresías staff/customer; no modifica owners ni se promueve a owner |
| `staff` | Operación de CRM, catálogo, reservas y ventas; uso de IA; sin administrar membresías ni configurar agentes |
| `customer` | Catálogo activo, contacto propio, reservas/pedidos propios y conversaciones propias |

El mapa de permisos concretos está en [REST](rest-api.md). Las escrituras aceptan una lista explícita de campos; un cliente no puede asignarse un rol, contacto, creador ni estado pagado.

Un cliente necesita una membresía activa y un contacto vinculado de manera confiable. Si falta el vínculo, las operaciones propias responden `409 customer_profile_required`; no se asume identidad por email. El alta/vinculación inicial se revisa antes de implementar onboarding.

Por defecto una conversación solo es visible a su propietario, también entre empleados. Configurar un agente o leer auditoría no concede acceso al contenido de conversaciones ajenas. No se añade supervisión global de conversaciones en este alcance.

## 6. Jobs y agentes

- El claim obtiene el tenant de la fila durable. El payload y los argumentos del modelo no pueden sobrescribirlo.
- Un job delegado identifica la membresía solicitante; dentro del tenant se resuelve su usuario y estado actual. Solo entonces se establece su identidad confiable para autorizar la tarea. Usuarios deshabilitados y membresías revocadas no continúan herramientas delegadas.
- Vencimientos de pedido y procesamiento de pagos usan un principal de servicio limitado al handler correspondiente. Un tenant suspendido bloquea operaciones nuevas de usuarios e IA; sí admite conciliación de pagos ya recibidos y liberación de stock por el principal de servicio, con auditoría.
- Cada invocación de herramienta se autoriza como el solicitante original, con sus permisos actuales y límites de datos propios. La aprobación humana no amplía esos permisos.
- El estado de un run, sus mensajes, las claves de idempotencia y la deduplicación de jobs incluyen tenant. No se reutiliza memoria conversacional entre empresas.

## 7. Otros canales de fuga

En el alcance inicial no hay caché distribuida ni almacenamiento de adjuntos. Si se introduce caché local para datos empresariales, su clave debe incluir tenant y, cuando corresponda, usuario/permisos; una entrada sin ese contexto no se puede compartir. Las respuestas privadas usan `Cache-Control: no-store`.

Logs, métricas, errores y auditoría tampoco pueden mezclar datos visibles a empresas distintas. `tenant_id` puede figurar en logs restringidos para diagnóstico, pero no como etiqueta ilimitada en métricas. No se registran access tokens, secretos, payloads de pago completos ni prompts en logs generales.

Una futura búsqueda o almacenamiento de archivos necesitará autorización antes de leer y claves con tenant. No se crean ahora esos componentes.

## 8. Verificación obligatoria al implementar

Las pruebas de aislamiento se ejecutarán sobre PostgreSQL real con roles runtime, no solo mocks o SQLite:

1. Dos tenants, usuarios compartidos y exclusivos, y datos con IDs conocidos. Probar GET/listado/conteo/POST/PATCH/cancelación con referencias ajenas.
2. Omitir deliberadamente el filtro de repositorio y comprobar RLS; intentar insertar/modificar una FK de otro tenant y cambiar `tenant_id`.
3. Consultar sin contexto y con contexto inválido; comprobar rollback y reutilización de la misma conexión de pool alternando tenants.
4. Ejecutar peticiones concurrentes y jobs de empresas distintas; comprobar que ninguna sesión/contexto se comparte.
5. Probar las excepciones de `/me/tenants`, claim y webhook: no permiten leer payloads empresariales ni ampliar permisos.
6. Revocar membresía o suspender tenant antes de ejecutar un job/herramienta; comprobar denegación y los handlers de conciliación permitidos.
7. Probar acceso propio de clientes dentro del mismo tenant, conversaciones privadas y aprobación con argumentos o tenant manipulados.
8. Comprobar catálogo de tablas: RLS, FORCE, grants, FK e índices; asegurar que el usuario de tests no oculta fallos al usar privilegios de propietario/superusuario.

Estas son condiciones de aceptación para implementación posterior; esta entrega no implementa ni ejecuta esas pruebas.
