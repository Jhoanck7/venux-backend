# ADR-0007 — Sitios públicos y habilitación de módulos

Estado propuesto para revisión.

Cada empresa puede publicar varios sitios y contratar módulos distintos. Tenancy administrará un identificador público único por sitio y una fila por combinación de empresa y módulo configurado. Una habilitación ausente se interpreta como desactivada. El catálogo inicial de claves se versiona en código.

El resolver público entrega solo la proyección de un sitio activo de empresa activa. La autorización privada exige identidad, membresía, permiso y capacidad habilitada. Separar estos conceptos permite cambiar dominios y frontends conservando el aislamiento y los contratos de negocio.

Guardar flags en cada frontend duplicaría la configuración y dejaría la API sin un control común. Un catálogo dinámico de plugins introduciría un ciclo de instalación innecesario para el monolito. La opción propuesta requiere desplegar código al añadir una capacidad nueva. La provisión queda bajo control de plataforma hasta acordar quién podrá administrarla.
