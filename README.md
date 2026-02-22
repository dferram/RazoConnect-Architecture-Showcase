# RazoConnect — Architecture Showcase

RazoConnect es una plataforma SaaS B2B multi-tenant en produccion que permite a negocios mayoristas operar con su propia marca, catalogo de productos, clientes y configuracion, compartiendo infraestructura con otros negocios de forma completamente aislada. El codigo fuente es privado al ser un producto comercial, pero toda la arquitectura, patrones de diseno y decisiones tecnicas estan documentadas en este repositorio.

---

## Tabla de Contenidos

- [Arquitectura General](#arquitectura-general)
- [Stack Tecnologico](#stack-tecnologico)
- [Puntos Fuertes del Proyecto](#puntos-fuertes-del-proyecto)
- [Documentacion Tecnica](#documentacion-tecnica)

---

## Arquitectura General

```mermaid
flowchart TD
    Cliente["Navegador / App"] --> Dominio["Dominio del Tenant\nrazo.com.mx"]
    Dominio --> TG["tenantGuard\nDeteccion por hostname"]
    TG --> Auth["authMiddleware\nJWT + Session"]
    Auth --> TSG["tenantSessionGuard\nVerifica tenant_id en token"]
    TSG --> Rutas["Router Express\n20+ modulos"]
    Rutas --> Controllers["Controllers"]
    Controllers --> Services["Services"]
    Services --> PG["PostgreSQL\n20+ tablas, 20+ funciones PL/pgSQL"]
    Services --> Cloudinary["Cloudinary\nImagenes optimizadas"]
    Services --> Email["Nodemailer\nPlantillas Handlebars"]
    Services --> MP["MercadoPago\nPagos en linea"]
```

El flujo comienza en el dominio del tenant. Cada hostname se resuelve a un registro de tenant en la base de datos antes de que cualquier logica de negocio se ejecute. La sesion y el token JWT se validan independientemente y se comparan contra el tenant detectado, de modo que un token valido para un negocio no puede operar en otro. Las rutas de Express estan organizadas en mas de 20 modulos funcionales que delegan a una capa de servicios, y esos servicios se comunican exclusivamente con PostgreSQL, Cloudinary, Nodemailer y MercadoPago.

---

## Stack Tecnologico

| Categoria | Tecnologia |
|---|---|
| Runtime | Node.js + Express |
| Base de datos | PostgreSQL con 20+ funciones PL/pgSQL y pg_cron |
| Autenticacion | JWT + express-session + Passport.js + Google OAuth 2.0 |
| Pagos | MercadoPago SDK con manejo de webhooks |
| Almacenamiento de imagenes | Cloudinary + Multer + Sharp (procesamiento antes de subir) |
| Email | Nodemailer con plantillas Handlebars |
| Generacion de documentos | PDFKit (facturas PDF) + ExcelJS (reportes Excel) |
| Tareas programadas | node-cron + pg_cron |
| Deployment | Azure App Service (trust proxy configurado) |
| Seguridad | Implementacion manual de OWASP: CSP, HSTS, rate limiting, input sanitization, secrets audit |
| Arquitectura | Multi-tenant Row-Level con 4 capas de aislamiento en middleware |

---

## Puntos Fuertes del Proyecto

**Seguridad sin dependencias de terceros.** Las cabeceras de seguridad (CSP, HSTS, X-Frame-Options), el rate limiter y el validador de inputs estan escritos a mano siguiendo OWASP Top 10. No se usa helmet ni ningun paquete de seguridad de terceros. Esto reduce la superficie de ataque y garantiza comprension completa de cada medida. Ver [SECURITY_LAYERS.md](SECURITY_LAYERS.md).

**Multi-tenancy con aislamiento real.** La deteccion de tenant por dominio, la validacion cruzada de JWT, la destruccion activa de sesion ante mismatch y el filtrado por `tenant_id` en cada query de base de datos forman cuatro capas independientes de aislamiento. Ver [MULTI_TENANCY.md](MULTI_TENANCY.md).

**Base de datos que se valida a si misma.** Mas de 20 funciones PL/pgSQL y 10+ triggers garantizan consistencia ACID sin delegar esa responsabilidad al codigo de aplicacion. pg_cron ejecuta mantenimiento diario directamente en la base de datos. Ver [DATABASE_DESIGN.md](DATABASE_DESIGN.md).

**Inventario inteligente con FIFO y reordenamiento automatico.** El sistema de asignacion de stock respeta orden de llegada pero permite prioridad para clientes VIP con efecto cascada documentado. El reordenamiento normaliza cantidades a multiplos del empaque del proveedor. Ver [SMART_INVENTORY.md](SMART_INVENTORY.md).

**Sistema de credito con scoring de riesgo.** El analisis de riesgo crediticio evalua antiguedad, historial de compras, maximo historico y pagos vencidos para generar una recomendacion antes de que el admin tome la decision final. Ver [CREDIT_SYSTEM.md](CREDIT_SYSTEM.md).

**Auditoria forense inmutable.** El Kardex registra cada movimiento de inventario con stock previo y posterior. El auditLogger genera diffs de cambios en formato JSONB. Ninguna de las dos tablas permite UPDATE ni DELETE. Ver [AUDIT_LOGGING.md](AUDIT_LOGGING.md).

**Automatizacion de extremo a extremo.** node-cron, pg_cron y triggers de base de datos trabajan en conjunto para actualizar deudas vencidas, notificar restock a favoritos y generar ordenes de compra automaticas ante backorders. Ver [AUTOMATION.md](AUTOMATION.md).

---

## Documentacion Tecnica

| Documento | Descripcion |
|---|---|
| [MULTI_TENANCY.md](MULTI_TENANCY.md) | Como funciona el aislamiento multi-tenant: los tres niveles de segregacion y por que RazoConnect elige Row-Level con proteccion adicional en middleware |
| [SECURITY_LAYERS.md](SECURITY_LAYERS.md) | Las 10 capas de seguridad implementadas manualmente siguiendo OWASP: desde securityHeaders hasta Row-Level Security en base de datos |
| [DATABASE_DESIGN.md](DATABASE_DESIGN.md) | Diseno del schema: 6 dominios, diagrama ER, 20+ funciones PL/pgSQL, triggers de sincronizacion y tareas pg_cron |
| [SMART_INVENTORY.md](SMART_INVENTORY.md) | Algoritmo FIFO con Priority Override, Smart Reordering por empaque de proveedor y notificaciones de restock a favoritos |
| [CREDIT_SYSTEM.md](CREDIT_SYSTEM.md) | Flujo de solicitud de credito, scoring de riesgo automatico, flujo RMA de devoluciones y estados del ciclo de vida del credito |
| [AUDIT_LOGGING.md](AUDIT_LOGGING.md) | Kardex inmutable append-only, diff tracking de cambios en JSONB y tipos de eventos auditados |
| [AUTOMATION.md](AUTOMATION.md) | Todas las automatizaciones del sistema: tareas cron, triggers de base de datos, generacion automatica de ordenes de compra y notificaciones |
| [MODULES.md](MODULES.md) | Inventario completo de los 20+ modulos del sistema organizados por actor: cliente, admin, agente y sistema |
| [DEVELOPER.md](DEVELOPER.md) | Perfil tecnico del desarrollador Diego Ferram y descripcion de xCore |

---

Desarrollado por Fernando Ramírez | xCore — 2025
