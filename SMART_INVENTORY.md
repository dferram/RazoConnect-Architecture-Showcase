# Sistema de Inventario Inteligente / Smart Inventory System

<details open>
<summary>🇲🇽 Español</summary>

RazoConnect gestiona el inventario con tres mecanismos que trabajan en conjunto: asignación FIFO con Priority Override para distribuir el stock disponible entre ordenes pendientes, Smart Reordering para normalizar cantidades de compra al empaque del proveedor, y notificaciones de restock a clientes con alertas activas en favoritos.

---

## Tabla de Contenidos

- [FIFO con Priority Override](#fifo-con-priority-override)
- [Flujo de Asignación de Inventario](#flujo-de-asignación-de-inventario)
- [Smart Reordering](#smart-reordering)
- [Notificaciones de Restock a Favoritos](#notificaciones-de-restock-a-favoritos)
- [OptimizationService](#optimizationservice)

---

## FIFO con Priority Override

El algoritmo base es FIFO (First In, First Out): las órdenes se surten en el orden en que llegaron. Sin embargo, las órdenes de clientes VIP pueden adelantarse en la fila. Cuando una orden VIP toma stock que estaba asignado a una orden normal, esa orden normal es degradada automáticamente y el cliente recibe una notificación.

Este comportamiento es deliberado. FIFO puro es justo pero ignorante del valor comercial de cada cliente. Priority Override permite que el negocio honre sus compromisos con clientes estratégicos sin abandonar la transparencia: cada degradación queda registrada y notificada.

```mermaid
flowchart TD
    Stock["Stock disponible: 1000 unidades"]
    OrdenA["Orden A — Normal — 500u — hace 5 dias"]
    OrdenB["Orden B — Normal — 300u — hace 3 dias"]
    OrdenC["Orden C — VIP — 250u — hace 1 dia"]

    Stock --> OrdenA
    OrdenA --> Surtida_A["Orden A Surtida\nStock restante: 500"]
    Surtida_A --> OrdenB
    OrdenB --> Surtida_B["Orden B Surtida\nStock restante: 200"]
    Surtida_B --> OrdenC
    OrdenC --> VIP_Check{"VIP con stock insuficiente\nToma stock de normales"}
    VIP_Check --> Degradación["Orden B degradada a PARCIAL\nOrden C Surtida"]
    Degradación --> Notificación["Notificación automática a cliente de Orden B"]
```

---

## Flujo de Asignación de Inventario

```mermaid
sequenceDiagram
    participant Admin as Admin
    participant SS as SmartStockService
    participant DB as PostgreSQL
    participant Cliente as Cliente

    Admin->>SS: Nueva entrada de stock (1000 unidades)
    SS->>DB: SELECT ordenes_pendientes ORDER BY es_prioritario DESC, fecha_creacion ASC
    DB-->>SS: [OrdenA, OrdenB, OrdenC_VIP]
    SS->>SS: Ejecutar algoritmo FIFO + Priority Override
    SS->>DB: UPDATE pedidos SET estatus, cantidad_surtida, cantidad_backorder
    SS->>DB: INSERT INTO movimientos_inventario (Kardex)
    SS->>DB: INSERT INTO notificaciones (clientes afectados)
    DB-->>Cliente: Notificación: "Tu pedido fue actualizado"
```

---

## Smart Reordering

Los proveedores venden en múltiplos de empaque: cajas de 12, packs de 24, paquetes de 6. Cuando el sistema genera una orden de compra automática, normaliza la cantidad solicitada al siguiente múltiplo del empaque del proveedor.

```mermaid
flowchart LR
    Solicitud["Cantidad solicitada: 7"] --> Regla["Regla de empaque: 5"]
    Regla --> Cálculo["ceil(7 / 5) * 5 = 10"]
    Cálculo --> Resultado["Cantidad normalizada: 10\nSobrante: 3"]
```

El sobrante no se desperdicia. Queda disponible en el inventario para cubrir otras órdenes pendientes o futuras. Esto reduce el número de órdenes de compra necesarias y optimiza el costo logístico por unidad.

---

## Notificaciones de Restock a Favoritos

Cuando un cliente agrega una variante de producto a favoritos con la alerta de restock activa, el sistema la registra. Cuando esa variante recibe nueva entrada de stock, el servicio de inventario notifica automáticamente a todos los clientes con alerta activa y desactiva la alerta para evitar notificaciones duplicadas.

```mermaid
sequenceDiagram
    participant Admin as Admin
    participant Inv as inventoryService
    participant DB as PostgreSQL
    participant Cliente as Cliente

    Admin->>Inv: Registrar entrada de stock (variante_id: 42)
    Inv->>DB: UPDATE stock_admin SET cantidad = cantidad + X
    Inv->>DB: SELECT clientes_favoritos WHERE variante_id = 42 AND alerta_restock_activa = TRUE
    DB-->>Inv: [cliente_1, cliente_2, cliente_3]
    Inv->>DB: INSERT INTO notificaciones (clienteid, tipo='restock', ...)
    Inv->>DB: UPDATE clientes_favoritos SET alerta_restock_activa = FALSE
    DB-->>Cliente: Notificación: "Tu producto favorito está disponible"
```

---

## OptimizationService

El OptimizationService analiza las órdenes de compra pendientes del sistema y detecta oportunidades de consolidación. Cuando hay múltiples órdenes pendientes para el mismo proveedor y el mismo producto, el servicio calcula el ahorro potencial de consolidarlas en una sola orden y la cantidad óptima considerando el empaque del proveedor.

El resultado es una sugerencia que el administrador puede aprobar o rechazar. Si la aprueba, el sistema crea un grupo de órdenes consolidadas manteniendo el tracking individual de cada orden para efectos de facturación y recepción.

---

Desarrollado por Fernando Ramírez | <a href="https://xcore-byg8fkdve4eyatbz.mexicocentral-01.azurewebsites.net/">xCore</a>

</details>

<details>
<summary>🇺🇸 English</summary>

RazoConnect manages inventory with three mechanisms working together: FIFO allocation with Priority Override to distribute available stock among pending orders, Smart Reordering to normalize purchase quantities to the supplier's packaging, and restock notifications to clients with active alerts in favorites.

---

## Table of Contents

- [FIFO with Priority Override](#fifo-with-priority-override)
- [Inventory Allocation Flow](#inventory-allocation-flow)
- [Smart Reordering](#smart-reordering-1)
- [Restock Notifications to Favorites](#restock-notifications-to-favorites)
- [OptimizationService](#optimizationservice-1)

---

## FIFO with Priority Override

The base algorithm is FIFO (First In, First Out): orders are fulfilled in the order they arrived. However, VIP client orders can move ahead in the queue. When a VIP order takes stock that was assigned to a normal order, that normal order is automatically downgraded and the client receives a notification.

This behavior is deliberate. Pure FIFO is fair but ignorant of the commercial value of each client. Priority Override allows the business to honor its commitments to strategic clients without abandoning transparency: each downgrade is recorded and notified.

```mermaid
flowchart TD
    Stock["Stock disponible: 1000 unidades"]
    OrdenA["Orden A — Normal — 500u — hace 5 dias"]
    OrdenB["Orden B — Normal — 300u — hace 3 dias"]
    OrdenC["Orden C — VIP — 250u — hace 1 dia"]

    Stock --> OrdenA
    OrdenA --> Surtida_A["Orden A Surtida\nStock restante: 500"]
    Surtida_A --> OrdenB
    OrdenB --> Surtida_B["Orden B Surtida\nStock restante: 200"]
    Surtida_B --> OrdenC
    OrdenC --> VIP_Check{"VIP con stock insuficiente\nToma stock de normales"}
    VIP_Check --> Degradacion["Orden B degradada a PARCIAL\nOrden C Surtida"]
    Degradacion --> Notificacion["Notificacion automatica a cliente de Orden B"]
```

---

## Inventory Allocation Flow

```mermaid
sequenceDiagram
    participant Admin as Admin
    participant SS as SmartStockService
    participant DB as PostgreSQL
    participant Cliente as Cliente

    Admin->>SS: Nueva entrada de stock (1000 unidades)
    SS->>DB: SELECT ordenes_pendientes ORDER BY es_prioritario DESC, fecha_creacion ASC
    DB-->>SS: [OrdenA, OrdenB, OrdenC_VIP]
    SS->>SS: Ejecutar algoritmo FIFO + Priority Override
    SS->>DB: UPDATE pedidos SET estatus, cantidad_surtida, cantidad_backorder
    SS->>DB: INSERT INTO movimientos_inventario (Kardex)
    SS->>DB: INSERT INTO notificaciones (clientes afectados)
    DB-->>Cliente: Notificacion: "Tu pedido fue actualizado"
```

---

## Smart Reordering

Suppliers sell in packaging multiples: boxes of 12, packs of 24, packages of 6. When the system generates an automatic purchase order, it normalizes the requested quantity to the next multiple of the supplier's packaging.

```mermaid
flowchart LR
    Solicitud["Cantidad solicitada: 7"] --> Regla["Regla de empaque: 5"]
    Regla --> Calculo["ceil(7 / 5) * 5 = 10"]
    Calculo --> Resultado["Cantidad normalizada: 10\nSobrante: 3"]
```

The surplus is not wasted. It remains available in inventory to cover other pending or future orders. This reduces the number of required purchase orders and optimizes the logistics cost per unit.

---

## Restock Notifications to Favorites

When a client adds a product variant to favorites with the restock alert active, the system records it. When that variant receives new stock, the inventory service automatically notifies all clients with an active alert and deactivates the alert to avoid duplicate notifications.

```mermaid
sequenceDiagram
    participant Admin as Admin
    participant Inv as inventoryService
    participant DB as PostgreSQL
    participant Cliente as Cliente

    Admin->>Inv: Registrar entrada de stock (variante_id: 42)
    Inv->>DB: UPDATE stock_admin SET cantidad = cantidad + X
    Inv->>DB: SELECT clientes_favoritos WHERE variante_id = 42 AND alerta_restock_activa = TRUE
    DB-->>Inv: [cliente_1, cliente_2, cliente_3]
    Inv->>DB: INSERT INTO notificaciones (clienteid, tipo='restock', ...)
    Inv->>DB: UPDATE clientes_favoritos SET alerta_restock_activa = FALSE
    DB-->>Cliente: Notificacion: "Tu producto favorito esta disponible"
```

---

## OptimizationService

The OptimizationService analyzes the system's pending purchase orders and detects consolidation opportunities. When there are multiple pending orders for the same supplier and the same product, the service calculates the potential savings from consolidating them into a single order and the optimal quantity considering the supplier's packaging.

The result is a suggestion that the administrator can approve or reject. If approved, the system creates a consolidated order group while maintaining individual tracking of each order for billing and receiving purposes.

---

Developed by Fernando Ramírez | <a href="https://xcore-byg8fkdve4eyatbz.mexicocentral-01.azurewebsites.net/">xCore</a>

</details>
