# Modulos del Sistema / System Modules

<details open>
<summary>🇲🇽 Español</summary>

RazoConnect esta organizado en mas de 20 modulos funcionales, cada uno con sus propias rutas, controladores y servicios. Los modulos se agrupan por el actor principal que los utiliza: clientes, administradores, agentes de venta y el sistema automatizado.

---

## Tabla de Contenidos

- [Mapa de Modulos por Actor](#mapa-de-modulos-por-actor)
- [Inventario Completo de Modulos](#inventario-completo-de-modulos)
- [Arquitectura de un Modulo](#arquitectura-de-un-modulo)

---

## Mapa de Modulos por Actor

```mermaid
flowchart TD
    subgraph Cliente
        C1[Catalogo de productos]
        C2[Carrito y pedidos]
        C3[Credito y pagos]
        C4[Devoluciones RMA]
        C5[Favoritos y alertas]
        C6[Notificaciones]
    end

    subgraph Admin
        A1[Panel de gestion]
        A2[Inventario y Kardex]
        A3[Ordenes de compra]
        A4[Clientes y creditos]
        A5[Reportes Excel y PDF]
        A6[Cupones y descuentos]
        A7[Landing configurable]
    end

    subgraph Agente
        AG1[Portal de agentes]
        AG2[Mis clientes]
        AG3[Comisiones]
        AG4[Crear pedidos para clientes]
    end

    subgraph Sistema
        S1[Cron jobs diarios]
        S2[Triggers de BD]
        S3[Emails automaticos]
        S4[Notificaciones de restock]
        S5[Auditoria y bitacora]
    end
```

---

## Inventario Completo de Modulos

| Modulo | Actor Principal | Descripcion |
|---|---|---|
| auth | Todos | Login con email/password, registro, Google OAuth 2.0 y logout. Maneja sesion con express-session y JWT simultaneamente |
| productos | Admin / Cliente | Catalogo de productos con variantes (talla, color, etc.), gestion de imagenes con procesamiento Sharp antes de subir a Cloudinary |
| carrito | Cliente | Agregar, quitar y calcular el carrito de compras con validacion de stock en tiempo real |
| pedidos | Cliente / Admin | Creacion de pedidos, actualizacion de estatus, generacion de PDF de remision con PDFKit |
| direcciones | Cliente | Gestion de multiples direcciones de envio por cliente |
| admin | Admin | Panel administrativo central con acceso a todos los recursos del tenant |
| reportes | Admin | Exportacion de reportes a Excel con ExcelJS: cuentas por cobrar, movimientos, ventas por periodo |
| public | Todos | Landing page del tenant configurable desde el panel admin; soporta temas visuales por temporada |
| notificaciones | Todos | Sistema de notificaciones in-app con lectura, archivo y conteo de no leidas |
| clientes | Admin / Agente | Gestion completa de clientes: alta, edicion, historial de compras, estado de credito |
| staff | Admin | Gestion de usuarios internos del tenant (admins, agentes, viewers) con control de roles |
| inventario | Admin / Agente | Kardex de movimientos, sesiones de inventario con asignacion de agentes, ajustes y auditorias |
| devoluciones | Admin / Cliente | Sistema RMA: solicitudes, evidencias fotograficas, reintegro de stock y ajuste de cuentas por cobrar |
| creditos | Admin / Cliente | Solicitud de credito, scoring de riesgo automatico, aprobacion manual, suspension por vencimiento |
| comisiones | Admin / Agente | Calculo automatico de comisiones al entregar pedidos, esquemas configurables por agente, reportes |
| agentes | Admin | Gestion de agentes de venta: alta, cartera de clientes asignada, metas y metricas |
| ordenes-compra | Admin | Ordenes de compra a proveedores con Smart Reordering, recepcion y validacion de empaque |
| favoritos | Cliente | Lista de productos favoritos con alertas de restock automaticas cuando el stock se repone |
| cupones | Admin / Cliente | Creacion y aplicacion de cupones de descuento: porcentaje, monto fijo, con fecha de vencimiento |
| pagos | Cliente | Integracion con MercadoPago SDK: checkout, webhooks de confirmacion, reconciliacion de estado |
| developer | Super Admin | Panel global para crear y gestionar tenants, configurar dominios y monitorear salud del sistema |

---

## Arquitectura de un Modulo

Todos los modulos siguen la misma estructura de capas, lo que facilita el onboarding y el mantenimiento.

```mermaid
flowchart LR
    Route["routes/modulo.js"] --> Controller["controllers/moduloController.js"]
    Controller --> Service["services/moduloService.js"]
    Service --> DB["PostgreSQL\nWHERE tenant_id = $1"]
    Controller --> Middleware["middlewares/\nauth, tenantGuard, rateLimiter"]
```

Los middlewares de seguridad (tenantGuard, authMiddleware, tenantSessionGuard) se aplican en el router antes de que la peticion llegue al controlador. Los controladores validan el request y coordinan los servicios, pero no contienen logica de negocio. Los servicios son los unicos que hablan con la base de datos o con servicios externos.

---

Desarrollado por Fernando Ramírez | <a href="https://xcore-byg8fkdve4eyatbz.mexicocentral-01.azurewebsites.net/">xCore</a>

</details>

<details>
<summary>🇺🇸 English</summary>

RazoConnect is organized into more than 20 functional modules, each with its own routes, controllers, and services. Modules are grouped by the primary actor that uses them: clients, administrators, sales agents, and the automated system.

---

## Table of Contents

- [Module Map by Actor](#module-map-by-actor)
- [Complete Module Inventory](#complete-module-inventory)
- [Module Architecture](#module-architecture)

---

## Module Map by Actor

```mermaid
flowchart TD
    subgraph Cliente
        C1[Catalogo de productos]
        C2[Carrito y pedidos]
        C3[Credito y pagos]
        C4[Devoluciones RMA]
        C5[Favoritos y alertas]
        C6[Notificaciones]
    end

    subgraph Admin
        A1[Panel de gestion]
        A2[Inventario y Kardex]
        A3[Ordenes de compra]
        A4[Clientes y creditos]
        A5[Reportes Excel y PDF]
        A6[Cupones y descuentos]
        A7[Landing configurable]
    end

    subgraph Agente
        AG1[Portal de agentes]
        AG2[Mis clientes]
        AG3[Comisiones]
        AG4[Crear pedidos para clientes]
    end

    subgraph Sistema
        S1[Cron jobs diarios]
        S2[Triggers de BD]
        S3[Emails automaticos]
        S4[Notificaciones de restock]
        S5[Auditoria y bitacora]
    end
```

---

## Complete Module Inventory

| Module | Primary Actor | Description |
|---|---|---|
| auth | All | Login with email/password, registration, Google OAuth 2.0, and logout. Handles session with express-session and JWT simultaneously |
| productos | Admin / Client | Product catalog with variants (size, color, etc.), image management with Sharp processing before uploading to Cloudinary |
| carrito | Client | Add, remove, and calculate the shopping cart with real-time stock validation |
| pedidos | Client / Admin | Order creation, status updates, PDF generation of delivery receipts with PDFKit |
| direcciones | Client | Management of multiple shipping addresses per client |
| admin | Admin | Central administrative panel with access to all tenant resources |
| reportes | Admin | Export reports to Excel with ExcelJS: accounts receivable, movements, sales by period |
| public | All | Configurable tenant landing page from the admin panel; supports seasonal visual themes |
| notificaciones | All | In-app notification system with read, archive, and unread count |
| clientes | Admin / Agent | Complete client management: registration, editing, purchase history, credit status |
| staff | Admin | Management of internal tenant users (admins, agents, viewers) with role control |
| inventario | Admin / Agent | Movement Kardex, inventory sessions with agent assignment, adjustments, and audits |
| devoluciones | Admin / Client | RMA system: requests, photographic evidence, stock reintegration, and accounts receivable adjustment |
| creditos | Admin / Client | Credit request, automatic risk scoring, manual approval, suspension for overdue |
| comisiones | Admin / Agent | Automatic commission calculation on order delivery, configurable schemes per agent, reports |
| agentes | Admin | Sales agent management: registration, assigned client portfolio, goals, and metrics |
| ordenes-compra | Admin | Purchase orders to suppliers with Smart Reordering, reception, and packaging validation |
| favoritos | Client | Product favorites list with automatic restock alerts when stock is replenished |
| cupones | Admin / Client | Creation and application of discount coupons: percentage, fixed amount, with expiration date |
| pagos | Client | Integration with MercadoPago SDK: checkout, confirmation webhooks, status reconciliation |
| developer | Super Admin | Global panel to create and manage tenants, configure domains, and monitor system health |

---

## Module Architecture

All modules follow the same layered structure, which facilitates onboarding and maintenance.

```mermaid
flowchart LR
    Route["routes/modulo.js"] --> Controller["controllers/moduloController.js"]
    Controller --> Service["services/moduloService.js"]
    Service --> DB["PostgreSQL\nWHERE tenant_id = $1"]
    Controller --> Middleware["middlewares/\nauth, tenantGuard, rateLimiter"]
```

The security middlewares (tenantGuard, authMiddleware, tenantSessionGuard) are applied at the router level before the request reaches the controller. Controllers validate the request and coordinate services, but do not contain business logic. Services are the only ones that communicate with the database or external services.

---

Developed by Fernando Ramírez | <a href="https://xcore-byg8fkdve4eyatbz.mexicocentral-01.azurewebsites.net/">xCore</a>

</details>
