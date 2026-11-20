# Contratos REST iniciales

Estado: **propuestos para revisión**. No se publican endpoints ni se genera una implementación OpenAPI en esta etapa. Tras aprobación, los esquemas Pydantic v2 y el OpenAPI de FastAPI deberán cumplir estos contratos.

## 1. Convenciones comunes

- Base `/api/v1`, HTTPS y JSON UTF-8. Rutas/propiedades en inglés; mensajes de interfaz traducidos por cada frontend mediante códigos estables.
- Autenticación con `Authorization: Bearer <access_token>` OIDC. No se reciben ID tokens ni se crea `/login` local. `/me` y `/me/tenants` no seleccionan empresa.
- En las tablas siguientes, `T` representa `/api/v1/tenants/{tenant_id}`. Todos sus endpoints exigen membresía activa y tenant operativo salvo las excepciones técnicas documentadas.
- No hay `tenant_id` en bodies de escritura. La API establece tenant, creador, propietario propio y campos de auditoría; rechaza campos desconocidos y campos de solo lectura.
- UUID como string; timestamps RFC 3339 con offset; dinero decimal como string y moneda ISO 4217. Los ejemplos con `<...>` son ilustrativos.
- Recurso individual como objeto, sin envoltorio `data`. Colecciones con `items` y `next_cursor`; límite por defecto 25, máximo 100. Cursor opaco firmado, ligado a tenant, actor, filtros y orden; no concede permisos.
- Orden estable `(created_at DESC, id DESC)` salvo mensajes por `sequence`. No se aceptan columnas SQL arbitrarias como filtros/orden. Mensajes usan cursor de secuencia; disponibilidad devuelve intervalos acotados.
- `POST` de creación síncrona devuelve `201` y `Location`; trabajo externo aceptado devuelve `202` y recurso de seguimiento. Cancelaciones y decisiones síncronas devuelven `200`. Eliminación de ventana: `204`.
- Recursos mutables devuelven `version` y `ETag` fuerte, por ejemplo `"v3"`. PATCH, eliminación de ventana y comandos sobre un recurso existente requieren `If-Match`. Ausente: `428`; versión distinta: `412`. El caso de uso vuelve a comprobar la versión al escribir.
- Respuestas privadas con `Cache-Control: no-store`; `X-Request-ID` para correlación. Límites y tiempos son configuración del servidor, no valores controlados por el cliente.

### Idempotencia

`Idempotency-Key` es obligatorio en POST que creen recursos, pagos, mensajes/runs, cancelaciones o decisiones de herramientas. La clave se identifica por tenant, actor y método/ruta de operación. Un hash canónico del body y las precondiciones permite comparar su contenido; no forma parte de la unicidad. Retención propuesta: al menos 24 horas, pendiente de validar con pagos y clientes.

La autorización actual se revalida antes de devolver un replay. Misma clave y mismo contenido devuelve el status/body original con `Idempotency-Replayed: true`, incluso si cambió la versión del recurso después de la primera ejecución. Misma clave con contenido o precondición diferente devuelve `409 idempotency_conflict`. Una solicitud concurrente idéntica espera de forma acotada; si no terminó, devuelve `409 operation_in_progress` con `Retry-After`.

Las mutaciones locales y su registro de respuesta se confirman en la misma transacción. Una caída antes de commit no deja un recurso creado sin deduplicación. En pagos se añade la clave estable del intento enviada al proveedor. Expirada la retención no se promete replay: el cliente debe consultar el recurso antes de repetir una operación antigua. No se guardan tokens ni secretos en respuestas persistidas para replay.

### Errores

`Content-Type: application/problem+json`, conforme a [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html). Ejemplo ilustrativo:

```json
{
  "type": "urn:venux:problem:booking-overlap",
  "title": "El recurso ya está reservado",
  "status": 409,
  "detail": "Selecciona otro intervalo.",
  "instance": "/api/v1/tenants/<tenant_id>/bookings",
  "code": "booking_overlap",
  "request_id": "<request_id>"
}
```

| HTTP | Uso |
| --- | --- |
| `401` | Token ausente, inválido o expirado; identidad deshabilitada |
| `403` | Usuario miembro sin permiso para la acción; tenant suspendido para operaciones nuevas |
| `404` | Recurso inexistente, tenant sin membresía o recurso ajeno al tenant/propietario |
| `409` | Conflicto de dominio: stock, intervalo, estado, idempotencia, perfil requerido o conversación ocupada |
| `412 / 428` | Versión desactualizada / precondición requerida |
| `413 / 422` | Payload demasiado grande / validación de campos; `errors` contiene campo y código sin datos sensibles |
| `429` | Cuota o límite excedido; incluye `Retry-After` |
| `503` | Dependencia necesaria indisponible o falta de capacidad; no confirma un efecto desconocido |

No se exponen SQL, trazas, nombres de tablas, respuestas crudas del proveedor ni datos sobre recursos de otra empresa.

## 2. Permisos

| Permiso | Owner | Admin | Staff | Customer |
| --- | --- | --- | --- | --- |
| `tenancy:read` | Sí | Sí | Sí | Sí, metadatos mínimos |
| `tenancy:manage` | Sí | Configuración y membresías staff/customer | No | No |
| `crm:read`, `crm:write` | Sí | Sí | Sí | No |
| `crm:self` | Sí | Sí | Sí | Solo contacto vinculado |
| `catalog:read` | Sí | Sí | Sí | Solo productos activos |
| `catalog:write` | Sí | Sí | Sí | No |
| `bookings:read`, `bookings:write` | Sí | Sí | Sí | No |
| `bookings:self` | Sí | Sí | Sí | Solo reservas propias |
| `commerce:read`, `commerce:write` | Sí | Sí | Sí | No |
| `commerce:self` | Sí | Sí | Sí | Solo pedidos/pagos propios |
| `ai:use` | Sí | Sí | Sí | Sí, conversación propia |
| `ai:configure` | Sí | Sí | No | No |
| `audit:read` | Sí | Sí | No | No |

Los permisos `self` nunca aceptan un contacto arbitrario del body. El servidor usa el contacto vinculado a la identidad. Owner/admin/staff pueden operar sobre un contacto concreto con su permiso de gestión. No hay herencia de permisos entre tenants.

## 3. Identidad y empresa

| Método y ruta | Entrada | Salida / permiso |
| --- | --- | --- |
| `GET /api/v1/me` | Token | `200` identidad propia: `id, display_name, email, status` |
| `GET /api/v1/me/tenants` | Token; paginación | `200` empresas con membresía propia: `id, slug, name, status, role` |
| `GET T` | — | `200` `TenantView` y capacidades efectivas; `tenancy:read` |
| `PATCH T` | `TenantPatch`; If-Match | `200` tenant; `tenancy:manage` |
| `GET T/memberships` | Paginación | `200` membresías del tenant, sin perfiles globales; `tenancy:manage` |
| `POST T/memberships` | `user_id, role` de una identidad ya provisionada | `201` membresía activa; owner admite cualquier rol, admin solo staff/customer |
| `PATCH T/memberships/{id}` | `role?`, `status?`; If-Match | `200`; owner protege último owner; admin solo modifica membresías staff/customer y asigna esos roles |

`TenantView`: `id, slug, name, timezone, currency, status, role, capabilities`. Solo administradores reciben `order_hold_minutes` y versión editable. `TenantPatch`: `name?`, `timezone?`, `order_hold_minutes?`; moneda, slug, estado y orígenes se provisionan/controlan fuera de este contrato inicial. Cambiar una zona no cambia los instantes ya guardados.

Los orígenes exactos y registros OIDC de frontends se mantienen mediante control de plataforma auditado. No hay API de alta pública de empresas, invitaciones ni cambio de proveedor de identidad en esta propuesta. Esos flujos requieren revisión de onboarding.

## 4. CRM

| Método y ruta | Entrada | Salida / permiso |
| --- | --- | --- |
| `GET T/crm/contacts` | `status?`, `q?`, cursor | `200` contactos; `crm:read` |
| `POST T/crm/contacts` | `ContactCreate` | `201` contacto; `crm:write` |
| `GET T/crm/contacts/{id}` | — | `200` contacto; `crm:read` |
| `PATCH T/crm/contacts/{id}` | `ContactPatch`; If-Match | `200`; `crm:write` |
| `GET T/crm/contacts/{id}/activities` | Cursor | `200` actividades; `crm:read` |
| `POST T/crm/contacts/{id}/activities` | `kind, summary, occurred_at` | `201` actividad; `crm:write` |
| `GET T/me/contact` | — | `200` contacto propio; `crm:self` |
| `PATCH T/me/contact` | `display_name?`, `phone?`; If-Match | `200` contacto propio; `crm:self` |

`ContactCreate`: `kind`, `display_name`, `email?`, `phone?`. `ContactPatch`: los campos anteriores y `status?`. Vincular `user_id` se hace mediante operación de control confiable durante onboarding; no se acepta en estos bodies. `ContactView`: `id, kind, display_name, email, phone, status, version, created_at, updated_at`; el perfil propio omite campos internos y el vínculo de identidad.

## 5. Catálogo e inventario

| Método y ruta | Entrada | Salida / permiso |
| --- | --- | --- |
| `GET T/catalog/products` | `status?`, `q?`, cursor | `200`; `catalog:read`; cliente solo activos |
| `GET T/catalog/products/{id}` | — | `200` `ProductView`; mismo límite |
| `POST T/catalog/products` | `ProductCreate` | `201`; `catalog:write` |
| `PATCH T/catalog/products/{id}` | `ProductPatch`; If-Match | `200`; `catalog:write` |
| `GET T/commerce/inventory` | `product_id?`, cursor | `200` stock; `commerce:read` |
| `GET T/commerce/inventory/{product_id}` | — | `200` stock/ETag del producto, incluso cero inicial; `commerce:read` |
| `PATCH T/commerce/inventory/{product_id}` | `on_hand, reason`; If-Match | `200` stock y versión; `commerce:write` |

`ProductCreate`: `sku, name, description?, unit_price, currency`, estado inicial `draft`. `ProductPatch` permite esos campos y `status`; moneda debe seguir siendo la del tenant. `ProductView`: `id, sku, name, description, unit_price, currency, status, version`. Un producto sin fila de inventario tiene disponible cero. Comercio inicializa la fila mediante el contrato de ajuste descrito debajo; catálogo no depende de comercio.

`InventoryView`: `product_id, on_hand, reserved, available, version`. Para un producto existente sin fila de inventario, la lectura individual administrativa devuelve stock cero con `ETag: "v0"`; el primer ajuste requiere ese ETag y crea la fila de forma atómica con versión 1. Una creación concurrente falla con `412`. Los siguientes ajustes requieren la versión almacenada. El ajuste no puede dejar unidades por debajo de las reservadas y siempre se audita. Clientes no reciben cifras internas de inventario; la compra es validada por el servidor.

## 6. Reservas

| Método y ruta | Entrada | Salida / permiso |
| --- | --- | --- |
| `GET T/bookings/resources` | Cursor | `200` recursos activos para cliente; lectura de reservas o `bookings:self` |
| `POST T/bookings/resources` | `name, timezone` | `201` recurso; `bookings:write` |
| `PATCH T/bookings/resources/{id}` | `name?`, `timezone?`, `status?`; If-Match | `200`; `bookings:write` |
| `GET T/bookings/resources/{id}/windows` | `from, to` | `200` ventanas; `bookings:read` |
| `POST T/bookings/resources/{id}/windows` | `starts_at, ends_at` | `201` ventana; `bookings:write` |
| `DELETE T/bookings/resources/{id}/windows/{window_id}` | If-Match | `204`, rechaza reservas futuras afectadas; `bookings:write` |
| `GET T/bookings/availability` | `resource_id, from, to, duration_minutes` | `200` intervalos libres; lectura de reservas o `bookings:self` |
| `GET T/bookings` | `from?`, `to?`, `resource_id?`, `status?`, cursor | `200`; `bookings:read` o filtro propio con `bookings:self` |
| `POST T/bookings` | `BookingCreate` | `201` `BookingView`; `bookings:write` o `bookings:self` |
| `GET T/bookings/{id}` | — | `200`; misma política de lectura/propiedad |
| `POST T/bookings/{id}/cancellation` | `reason?`; If-Match | `200` reserva cancelada; gestión o propiedad y política de cancelación |
| `POST T/bookings/{id}/completion` | `outcome=completed/no_show`; If-Match | `200`; `bookings:write`, después del fin |

`BookingCreate`: `resource_id, starts_at, ends_at`, y `contact_id` obligatorio solo para personal que opera por otro contacto; clientes lo omiten. `BookingView`: `id, resource_id, contact_id, starts_at, ends_at, status, version, created_at`. La disponibilidad no devuelve IDs ni datos de otras reservas. Consulta máxima propuesta: 31 días; `duration_minutes` debe ser positivo y acotado por configuración.

## 7. Pedidos y pagos

| Método y ruta | Entrada | Salida / permiso |
| --- | --- | --- |
| `GET T/commerce/orders` | `status?`, cursor | `200`; `commerce:read` o filtro propio con `commerce:self` |
| `POST T/commerce/orders` | `OrderCreate` | `201` `OrderView`; `commerce:write` o `commerce:self` |
| `GET T/commerce/orders/{id}` | — | `200`; misma política |
| `POST T/commerce/orders/{id}/cancellation` | `reason?`; If-Match | `200`, solo pendiente; `commerce:write` o propiedad |
| `POST T/commerce/orders/{id}/payments` | Body vacío; If-Match del pedido | `202` `PaymentView` y Location de pago; gestión o propiedad |
| `GET T/commerce/payments/{payment_id}` | — | `200` `PaymentView`; acceso mediante propiedad del pedido |
| `POST /api/v1/webhooks/payments/{payment_account_id}` | Firma del proveedor y body original | `200` solo tras recepción durable; fuera de autenticación OIDC |

`OrderCreate`: `items` con entre 1 y 100 entradas `{product_id, quantity}`; `contact_id` solo para personal. Productos repetidos se rechazan; cantidades enteras positivas y límites de compra configurados. No acepta precio, total, estado, número, reserva de stock ni cuenta de pago. La cuenta se elige por configuración de tenant, no por un ID arbitrario del cliente.

`OrderView`: `id, number, contact_id, status, currency, total_amount, expires_at, lines, version, created_at`; cada línea contiene `product_id, sku, name, quantity, unit_price, line_total`. `reconciliation_required` es visible únicamente al personal autorizado. El frontend muestra el total congelado antes de iniciar pago.

`PaymentView`: `id, order_id, status, amount, currency, checkout_url?, failure_code?, version`. No incluye `secret_ref`, payloads ni claves internas del proveedor. `202` normalmente devuelve `pending` y `checkout_url=null`; el frontend consulta el recurso hasta que exista URL o resultado. Se distinguen «preparando checkout» y «esperando pago» por presencia de URL, sin inferir éxito de una redirección del navegador.

El webhook valida firma sobre bytes originales, ventana temporal y cuenta receptora. Un duplicado válido recibe `200` sin repetir efectos. Firma inválida devuelve `401`; cuenta no encontrada `404`; fallo de persistencia `503` para permitir reentrega. El body y algoritmo de firma se concretarán con la pasarela; VENUX no inventa un formato del proveedor.

## 8. Agentes de IA

| Método y ruta | Entrada | Salida / permiso |
| --- | --- | --- |
| `GET T/ai/agents` | Cursor | `200` vistas públicas `id, name, status`; `ai:use` |
| `GET T/ai/agents/{id}/configuration` | — | `200` configuración y versión; `ai:configure` |
| `POST T/ai/agents` | `AgentCreate` | `201`; `ai:configure` |
| `PATCH T/ai/agents/{id}` | `AgentPatch`; If-Match | `200`; `ai:configure` |
| `GET T/ai/conversations` | Cursor | `200` conversaciones propias; `ai:use` |
| `POST T/ai/conversations` | `agent_id` | `201` conversación propia; `ai:use` |
| `GET T/ai/conversations/{id}` | — | `200` conversación propia; `ai:use` |
| `GET T/ai/conversations/{id}/messages` | Cursor de secuencia | `200` mensajes visibles; propietario y `ai:use` |
| `POST T/ai/conversations/{id}/messages` | `content`; If-Match de conversación | `202` `RunView`; crea mensaje y run atómicamente; propietario y `ai:use` |
| `GET T/ai/runs/{id}` | — | `200` `RunView`; propietario y `ai:use` |
| `POST T/ai/tool-invocations/{id}/decision` | `decision=approve/reject, arguments_hash`; If-Match | `200` decisión persistida; propietario, `ai:use` y permiso de la acción |

`AgentCreate`: `name, instructions, provider, model, allowed_tools, max_steps, max_tokens`. `AgentPatch` admite esos campos y `status`. Proveedor/modelo y herramientas deben pertenecer a listas permitidas de plataforma. No se aceptan credenciales, direcciones de red arbitrarias ni código de herramientas.

`RunView`: `id, conversation_id, input_message_id, status, version, usage, error_code?, pending_tool_invocations`. Cada propuesta pendiente expone `id, tool_name, arguments` filtrados, `arguments_hash, approval_expires_at, version`. El contrato no expone instrucciones de sistema, secretos ni razonamientos internos del modelo.

Crear mensaje requiere texto no vacío con máximo inicial propuesto de 32 KiB. Un run en curso devuelve `409 conversation_busy`. Una propuesta rechazada o expirada termina el run sin ejecutar la acción. La aprobación es de un solo uso y el worker vuelve a autorizar antes de actuar. El estado `succeeded` de una herramienta de pago significa que solicitó el intento; el resultado del cobro se consulta en el recurso de pago.

## 9. Auditoría y salud

| Método y ruta | Entrada | Salida / permiso |
| --- | --- | --- |
| `GET T/audit/events` | `from?`, `to?`, `action?`, cursor | `200` eventos sanitizados del tenant; `audit:read` |
| `GET /health/live` | Sin token | `200` proceso vivo; sin configuración ni detalles internos |
| `GET /health/ready` | Red interna/proxy | `200` listo o `503`; comprueba base y revisión de migraciones |

La auditoría no incluye mensajes completos, datos de tarjeta, tokens ni resultados sensibles de herramientas. OpenAPI interactivo y métricas se restringen en producción según [seguridad y operación](security-and-operations.md).

## 10. Ejemplos de intercambio

Reserva de cliente; `contact_id` se deriva del usuario:

```http
POST /api/v1/tenants/<tenant_id>/bookings
Authorization: Bearer <access_token>
Idempotency-Key: <unique_key>
Content-Type: application/json

{"resource_id":"<resource_id>","starts_at":"2026-12-01T13:00:00Z","ends_at":"2026-12-01T14:00:00Z"}
```

```json
{
  "id": "<booking_id>",
  "resource_id": "<resource_id>",
  "contact_id": "<own_contact_id>",
  "starts_at": "2026-12-01T13:00:00Z",
  "ends_at": "2026-12-01T14:00:00Z",
  "status": "confirmed",
  "version": 1,
  "created_at": "2026-11-17T16:00:00Z"
}
```

Pedido de cliente:

```http
POST /api/v1/tenants/<tenant_id>/commerce/orders
Authorization: Bearer <access_token>
Idempotency-Key: <unique_key>
Content-Type: application/json

{"items":[{"product_id":"<product_id>","quantity":2}]}
```

La respuesta `201` incluye líneas congeladas, `total_amount`, `currency`, `expires_at`, `version` y ETag. El cliente usa ese ETag al solicitar el pago; no envía un total elegido por él.

## 11. Criterios de compatibilidad y validación

Cambios aditivos opcionales pueden convivir en v1. Eliminar/renombrar campos, cambiar semántica de dinero, ampliar visibilidad o modificar estados válidos exige revisión de compatibilidad y, cuando rompa clientes, otra versión. Los consumidores deben tolerar nuevos campos de respuesta; los requests siguen rechazando campos desconocidos.

Al implementar se probarán OpenAPI, validaciones, status, ETag, idempotencia, paginación, permisos propios/ajenos y los errores de negocio descritos. Los límites de body generales se proponen en 1 MiB; payloads de webhook y proveedor requieren límites concretos tras elegir integraciones.

