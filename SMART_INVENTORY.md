# Sistema de Inventario Inteligente

RazoConnect gestiona el inventario con tres mecanismos que trabajan en conjunto: asignacion FIFO con Priority Override para distribuir el stock disponible entre ordenes pendientes, Smart Reordering para normalizar cantidades de compra al empaque del proveedor, y notificaciones de restock a clientes con alertas activas en favoritos.

---

## Tabla de Contenidos

- [FIFO con Priority Override](#fifo-con-priority-override)
- [Flujo de Asignacion de Inventario](#flujo-de-asignacion-de-inventario)
- [Smart Reordering](#smart-reordering)
- [Notificaciones de Restock a Favoritos](#notificaciones-de-restock-a-favoritos)
- [OptimizationService](#optimizationservice)

---

## FIFO con Priority Override

El algoritmo base es FIFO (First In, First Out): las ordenes se surten en el orden en que llegaron. Sin embargo, las ordenes de clientes VIP pueden adelantarse en la fila. Cuando una orden VIP toma stock que estaba asignado a una orden normal, esa orden normal es degradada automaticamente y el cliente recibe una notificacion.

Este comportamiento es deliberado. FIFO puro es justo pero ignorante del valor comercial de cada cliente. Priority Override permite que el negocio honre sus compromisos con clientes estrategicos sin abandonar la transparencia: cada degradacion queda registrada y notificada.

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

## Flujo de Asignacion de Inventario

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

Los proveedores venden en multiplos de empaque: cajas de 12, packs de 24, paquetes de 6. Cuando el sistema genera una orden de compra automatica, normaliza la cantidad solicitada al siguiente multiplo del empaque del proveedor.

```mermaid
flowchart LR
    Solicitud["Cantidad solicitada: 7"] --> Regla["Regla de empaque: 5"]
    Regla --> Calculo["ceil(7 / 5) * 5 = 10"]
    Calculo --> Resultado["Cantidad normalizada: 10\nSobrante: 3"]
```

El sobrante no se desperdicia. Queda disponible en el inventario para cubrir otras ordenes pendientes o futuras. Esto reduce el numero de ordenes de compra necesarias y optimiza el costo logistico por unidad.

---

## Notificaciones de Restock a Favoritos

Cuando un cliente agrega una variante de producto a favoritos con la alerta de restock activa, el sistema la registra. Cuando esa variante recibe nueva entrada de stock, el servicio de inventario notifica automaticamente a todos los clientes con alerta activa y desactiva la alerta para evitar notificaciones duplicadas.

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

El OptimizationService analiza las ordenes de compra pendientes del sistema y detecta oportunidades de consolidacion. Cuando hay multiples ordenes pendientes para el mismo proveedor y el mismo producto, el servicio calcula el ahorro potencial de consolidarlas en una sola orden y la cantidad optima considerando el empaque del proveedor.

El resultado es una sugerencia que el administrador puede aprobar o rechazar. Si la aprueba, el sistema crea un grupo de ordenes consolidadas manteniendo el tracking individual de cada orden para efectos de facturacion y recepcion.

---

Desarrollado por Diego Ferram | xCore — 2025
