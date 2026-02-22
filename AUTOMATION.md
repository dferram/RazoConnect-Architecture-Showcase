# Automatizaciones del Sistema

RazoConnect automatiza una parte significativa de su operacion a traves de tres mecanismos que trabajan en paralelo: tareas programadas con node-cron, tareas de base de datos con pg_cron y triggers PL/pgSQL, y logica de aplicacion que genera acciones automaticas en respuesta a eventos de negocio.

---

## Tabla de Contenidos

- [Mapa de Automatizaciones](#mapa-de-automatizaciones)
- [Tareas Cron](#tareas-cron)
- [Triggers de Base de Datos](#triggers-de-base-de-datos)
- [Generacion Automatica de Ordenes de Compra](#generacion-automatica-de-ordenes-de-compra)
- [Notificaciones Automaticas](#notificaciones-automaticas)
- [OptimizationService](#optimizationservice)

---

## Mapa de Automatizaciones

```mermaid
flowchart TD
    Cron["node-cron\nEjecucion diaria"] --> D1["Actualizar deudas vencidas\nactualizar_estatus_deuda_vencida()"]
    Cron --> D2["Verificar stock bajo\nAlertas a admins"]
    Cron --> D3["Limpiar sesiones expiradas"]

    Trigger1["Trigger: inventarios_admin UPDATE"] --> T1["sync_producto_variante_stock()\nRecalcula stock consolidado"]

    Trigger2["Trigger: pedidos UPDATE fecha_vencimiento"] --> T2["trigger_actualizar_estatus_deuda()\nMarca VENCIDA si fecha < hoy"]

    Trigger3["Trigger: detalles_del_pedido UPDATE"] --> T3["recalcular_total_pedido()\nDetecta discrepancias de total"]

    Pedido["Pedido sin stock suficiente"] --> OC["generarOrdenCompraAutomatica()\nCrea OC pendiente con\nSmart Reordering aplicado"]

    Stock["Nueva entrada de stock"] --> Restock["notificarRestockFavoritos()\nNotifica a clientes con\nalerta activa"]
```

---

## Tareas Cron

node-cron ejecuta `scheduleDailyMaintenance` una vez al dia. Esta funcion coordina las tareas de mantenimiento que deben ejecutarse en el contexto de la aplicacion.

| Tarea | Descripcion |
|---|---|
| Verificar stock bajo | Consulta variantes con stock por debajo del minimo configurado por tenant y crea notificaciones para los admins correspondientes |
| Limpiar sesiones expiradas | Elimina registros de sesion con fecha de expiracion anterior a la fecha actual |

Las tareas de deuda vencida y suspension de clientes se ejecutan directamente en PostgreSQL via pg_cron, sin pasar por la aplicacion. Ver [DATABASE_DESIGN.md](DATABASE_DESIGN.md) para el detalle de esas funciones.

---

## Triggers de Base de Datos

Los triggers de PostgreSQL son la forma mas confiable de garantizar consistencia porque se ejecutan dentro de la misma transaccion que la operacion que los dispara, sin posibilidad de ser omitidos.

```mermaid
flowchart TD
    subgraph Inventario
        I1["UPDATE inventarios_admin"] --> I2["sync_producto_variante_stock()\nRecalcula stock en producto_variantes"]
        I3["INSERT movimientos_inventario"] --> I4["fn_validar_movimiento_inventario()\nVerifica stock_posterior = stock_previo + cantidad"]
    end

    subgraph Pedidos
        P1["UPDATE/INSERT detalles_del_pedido"] --> P2["recalcular_total_pedido()\nDetecta discrepancias sin modificar"]
        P3["UPDATE pedidos fecha_vencimiento"] --> P4["trigger_actualizar_estatus_deuda()\nMarca VENCIDA si fecha < hoy"]
    end

    subgraph Devoluciones
        D1["INSERT/UPDATE devoluciones_detalles"] --> D2["actualizar_monto_total_devolucion()\nRecalcula monto de la devolucion"]
        D3["INSERT/UPDATE devoluciones_detalles"] --> D4["validar_cantidad_devuelta()\nImpide devolver mas de lo comprado"]
    end

    subgraph Notificaciones
        N1["INSERT notificaciones"] --> N2["limitar_notificaciones_por_cliente()\nMantiene las 100 mas recientes"]
    end
```

---

## Generacion Automatica de Ordenes de Compra

Cuando un pedido de cliente no puede ser surtido completamente por falta de stock, el sistema genera automaticamente una orden de compra al proveedor correspondiente.

```mermaid
flowchart TD
    Pedido["Pedido confirmado"] --> Stock["SmartStockService\nAsignar stock disponible"]
    Stock --> Backorder{"Hay items sin stock?"}
    Backorder -->|No| Completo["Pedido surtido completamente"]
    Backorder -->|Si| OC["generarOrdenCompraAutomatica()"]
    OC --> Smart["Aplicar Smart Reordering\nceil(cantidad / empaque) * empaque"]
    Smart --> Proveedor["Identificar proveedor de la variante"]
    Proveedor --> Insert["INSERT INTO ordenesdecompra (estatus=PENDIENTE)"]
    Insert --> Notif["Notificacion al admin:\nOC generada automaticamente"]
```

La orden de compra queda en estado PENDIENTE para que el administrador la revise y la convierta en una compra real al proveedor. El sistema no envia ordenes directamente a proveedores: genera el documento interno y notifica al responsable.

---

## Notificaciones Automaticas

El sistema genera notificaciones in-app automaticas en respuesta a los siguientes eventos:

| Evento | Destinatario | Tipo de Notificacion |
|---|---|---|
| Nueva entrada de stock en variante con alertas activas | Clientes con alerta en favoritos | restock |
| Cambio de estatus de pedido | Cliente dueno del pedido | actualizacion_pedido |
| Orden de compra generada automaticamente | Admins del tenant | oc_automatica |
| Stock por debajo del minimo | Admins del tenant | stock_bajo |
| Devolucion aprobada o rechazada | Cliente solicitante | devolucion_resultado |
| Deuda proxima a vencer | Cliente con credito activo | deuda_por_vencer |

---

## OptimizationService

El OptimizationService no genera acciones automaticas; genera sugerencias que el administrador puede aprobar. Analiza las ordenes de compra pendientes y detecta dos tipos de oportunidades:

**Consolidacion de ordenes.** Cuando hay multiples ordenes de compra pendientes para el mismo proveedor, calcula si consolidarlas en una sola orden reduce la cantidad total necesaria (considerando el empaque del proveedor) y cuanto ahorro representa.

**Deteccion de sobrestock potencial.** Si consolidar varias ordenes resultaria en un sobrestock significativo de alguna variante, el servicio lo indica para que el administrador ajuste las cantidades antes de aprobar.

La sugerencia incluye: ordenes involucradas, cantidad actual vs cantidad optimizada, ahorro estimado y el impacto en cada pedido de cliente pendiente.

---

Desarrollado por Fernando | xCore 
