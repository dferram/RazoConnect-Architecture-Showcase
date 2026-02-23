# Sistema de Credito y Devoluciones

RazoConnect incluye un modulo de credito completo que cubre el ciclo de vida desde la solicitud hasta la suspension, y un modulo RMA (Return Merchandise Authorization) para gestionar devoluciones con reintegro automatico de inventario y ajuste de cuentas por cobrar.

---

## Tabla de Contenidos

- [Flujo de Solicitud de Credito](#flujo-de-solicitud-de-credito)
- [Scoring de Riesgo Crediticio](#scoring-de-riesgo-crediticio)
- [Estados del Credito](#estados-del-credito)
- [Middleware checkCreditStatus](#middleware-checkcreditstatus)
- [Flujo RMA — Devoluciones](#flujo-rma--devoluciones)

---

## Flujo de Solicitud de Credito

```mermaid
flowchart TD
    Cliente["Cliente solicita crédito"] --> Solicitud["POST /api/creditos/solicitar"]
    Solicitud --> Analisis["creditAnalysisService.analizarRiesgoCredito()"]
    Analisis --> Factores["Calcular:\n- Antigüedad en meses\n- Max ticket histórico\n- Frecuencia de compras\n- Pagos vencidos"]
    Factores --> Nivel{"¿Nivel de riesgo?"}
    
    Nivel -->|"Antigüedad > 6 meses\nMonto <= max * 1.5\nPedidos > 3"| Bajo["BAJO"]
    Nivel -->|"Antigüedad 1-6 meses\nMonto <= max * 2.5"| Medio["MEDIO"]
    Nivel -->|"Nuevo o monto excesivo\nSin historial"| Alto["ALTO"]
    
    Bajo --> Admin["Admin revisa y aprueba/rechaza"]
    Medio --> Admin
    Alto --> Admin
    
    Admin --> Aprobado["Crédito ACTIVO\nLímite asignado"]
```

El sistema genera una recomendacion automatica pero no aprueba ni rechaza de forma autonoma. El administrador siempre tiene la decision final. Esto preserva el control humano sobre compromisos financieros mientras elimina el trabajo manual de recopilar y calcular los factores de riesgo.

---

## Scoring de Riesgo Crediticio

El `creditAnalysisService` evalua cuatro factores del historial del cliente para calcular un nivel de riesgo.

| Factor | Indicador de Riesgo Bajo | Indicador de Riesgo Alto |
|---|---|---|
| Antiguedad en la plataforma | Mas de 6 meses | Menos de 1 mes |
| Frecuencia de compras | Mas de 3 pedidos al mes | Sin historial de pedidos |
| Monto solicitado vs max historico | Dentro de 1.5x el maximo historico | Mas de 2.5x el maximo historico |
| Pagos vencidos | Sin deudas vencidas | Deuda activa o historial de incumplimiento |

Un cliente con riesgo BAJO puede ver aprobado su credito rapidamente. Un cliente con riesgo ALTO probablemente sea rechazado, aunque el administrador puede override la recomendacion con justificacion.

---

## Estados del Credito

```mermaid
stateDiagram-v2
    [*] --> PENDIENTE : Solicitud enviada
    PENDIENTE --> ACTIVO : Admin aprueba
    PENDIENTE --> RECHAZADO : Admin rechaza
    ACTIVO --> SUSPENDIDO : Deuda vencida > 15 dias
    SUSPENDIDO --> ACTIVO : Cliente regulariza deuda
    ACTIVO --> CANCELADO : Admin cancela manualmente
    CANCELADO --> [*]
    RECHAZADO --> [*]
```

| Estado | Descripcion |
|---|---|
| ACTIVO | El cliente puede realizar pedidos a credito dentro de su limite |
| SUSPENDIDO | Credito bloqueado automaticamente por deuda vencida; se reactiva al regularizar |
| CANCELADO | Credito cancelado permanentemente por el administrador |

La suspension automatica es ejecutada por la funcion PL/pgSQL `suspender_clientes_morosos()` que corre diariamente via pg_cron.

---

## Middleware checkCreditStatus

Antes de confirmar un pedido con pago a credito, el middleware `checkCreditStatus` verifica:

```mermaid
flowchart TD
    Pedido["Cliente confirma pedido a credito"] --> C1["checkCreditAccess\nTenant tiene modulo de credito activo?"]
    C1 --> C2["checkCreditStatus\nCredito del cliente esta ACTIVO?"]
    C2 --> C3["Limite disponible >= monto del pedido?"]
    C3 --> C4["Sin deuda vencida activa?"]
    C4 --> Confirmado["Pedido confirmado\nCxC actualizada"]
    C2 -->|"SUSPENDIDO o CANCELADO"| Rechazo["HTTP 403\nMensaje descriptivo al cliente"]
    C3 -->|"Limite insuficiente"| Rechazo
    C4 -->|"Deuda vencida"| Rechazo
```

---

## Flujo RMA — Devoluciones

El modulo de devoluciones implementa un flujo RMA completo con cuatro validaciones antes de crear la solicitud y procesamiento automatico al ser aprobada por el administrador.

```mermaid
flowchart TD
    Cliente["Cliente solicita devolucion"] --> V1["Validacion 1: Pedido pertenece al cliente"]
    V1 --> V2["Validacion 2: Han pasado menos de 30 dias"]
    V2 --> V3["Validacion 3: Pedido esta Completado o Entregado"]
    V3 --> V4["Validacion 4: Items existen en el pedido"]
    V4 --> Pendiente["Devolucion en estado PENDIENTE"]
    Pendiente --> Admin["Admin revisa"]
    Admin -->|"Aprobar"| Proceso["Procesar automaticamente:"]
    Proceso --> I1["Reintegro de inventario al stock"]
    Proceso --> I2["Ajuste de CxC del cliente"]
    Proceso --> I3["Actualizacion de estado del pedido"]
    Proceso --> I4["Email de confirmacion al cliente"]
    Admin -->|"Rechazar"| Rechazo["Email de rechazo al cliente"]
```

Al aprobar una devolucion, el sistema ejecuta las cuatro acciones de forma atomica dentro de una transaccion. Si cualquiera de ellas falla (por ejemplo, el reintegro de inventario), la transaccion completa se revierte y la devolucion permanece en estado PENDIENTE con el error registrado.

---

Desarrollado por Fernando Ramírez | <a href="https://xcore-byg8fkdve4eyatbz.mexicocentral-01.azurewebsites.net/">xCore</a>
