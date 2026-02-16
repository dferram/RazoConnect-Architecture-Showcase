# RazoConnect - Architecture Case Study

> **Cómo construir una plataforma B2B multi-tenant escalable para venta al mayoreo**
>
> Este repositorio documenta la arquitectura completa de **RazoConnect**, un SaaS B2B en producción que maneja inventario, crédito y órdenes para múltiples negocios simultáneamente. El código fuente es privado (producto comercial), pero toda la arquitectura, patrones de diseño y decisiones técnicas están completamente documentadas.

---

## 📊 Resumen Ejecutivo

| Métrica | Valor |
|---------|-------|
| **Líneas de Código** | 50,000+ |
| **Commits en 46 días** | 560+ commits (12/día) |
| **Tenants en Producción** | 3+ clientes activos |
| **Stack** | Node.js + PostgreSQL + Azure |
| **Uptime** | 99.8% |
| **Concurrent Users** | 500+ soportados |

---

## 🎯 ¿Qué es RazoConnect?

RazoConnect es una plataforma SaaS que permite a negocios mayoristas vender productos al por mayor manteniendo su propia marca, clientes y configuración. Múltiples negocios (tenants) conviven en la misma infraestructura de forma completamente aislada.

### El Problema Original

Cada negocio mayorista tenía su propio e-commerce separado, lo que resultaba en:
- ❌ Código duplicado
- ❌ Mantenimiento triplicado
- ❌ Costos de infraestructura altos
- ❌ Features no escalaban a todos

### La Solución

Una plataforma multi-tenant donde:
- ✅ Un código base sirve a múltiples negocios
- ✅ Cada tenant tiene datos completamente aislados
- ✅ Nuevas features se despliegan a todos automáticamente
- ✅ Costos operativos 60% menores

---

## 🏗️ Las 5 Capas de la Arquitectura

### **Capa 1: Presentación (Frontend)**
- Interface web responsiva construida con JavaScript vanilla + Bootstrap
- Temas personalizables según el tenant (Razo theme, Fashion theme)
- Carga dinámica de contenido basada en el tenant
- Componentes reutilizables para máxima eficiencia

### **Capa 2: Seguridad (4 Capas de Aislamiento)**
RazoConnect implementa validación multi-capa para garantizar que los datos de un tenant jamás se filtren a otro:

1. **Detección de Tenant por Dominio:** Cuando accedes a `razo.com.mx` vs `fashion.shop.mx`, el sistema detecta automáticamente cuál tenant está siendo accedido.
2. **Validación User-Tenant:** Se verifica que el usuario actual pertenezca realmente al tenant que está visitando.
3. **JWT Token Binding:** Los tokens contienen el ID del tenant y no pueden ser usados en otro tenant.
4. **Row-Level Security en BD:** Cada query filtra automáticamente por tenant_id, proteggiendo los datos incluso si alguien obtiene acceso directo a la BD.

### **Capa 3: API (Express.js + Custom Middlewares)**
- API RESTful con endpoints organizados por recurso
- Middlewares personalizados para autenticación, validación y auditoría
- Manejo centralizado de errores
- Rate limiting para protección contra abuso

### **Capa 4: Lógica de Negocio (Services)**
Servicios especializados que manejan la inteligencia del sistema:

- **SmartStockService:** Asigna inventario inteligentemente usando FIFO con prioridades
- **CreditAnalysisService:** Evalúa automáticamente riesgo de crédito de clientes
- **OptimizationService:** Sugiere consolidaciones de órdenes para ahorrar costos
- **KardexService:** Registra de forma inmutable cada movimiento de inventario
- **AuditLogger:** Registra todas las acciones para compliance

### **Capa 5: Datos (PostgreSQL + Azure)**
- Base de datos centralizada con aislamiento por tenant
- Transacciones ACID para garantizar consistencia
- Kardex (movimientos inmutables)
- Audit tables para trazabilidad legal

---

## 🔐 El Desafío: Multi-Tenancy

### ¿Por qué es crítico?

Imagina que tienes 3 clientes usando RazoConnect:
- **Razo:** Mayorista de ropa
- **Fashion Plus:** Distribuidor de moda
- **TechPro:** Mayorista de electrónica

Todos comparten servidores, base de datos e infraestructura, pero JAMÁS deben verse datos entre ellos. Es como tener 3 bancos compartiendo el mismo edificio pero con bóvedas completamente separadas.

### La Solución: Validación en 4 Capas

**Layer 1 - Tenant Guard (Detección)**
El sistema detecta automáticamente cuál tenant está siendo accedido basado en el dominio de la petición. Si no existe o no está activo, rechaza la petición.

**Layer 2 - User Validation (Matching)**
Verifica que el usuario que está usando la sesión realmente pertenece al tenant que está intentando acceder. Si hay un mismatch (imposible en producción, pero si ocurre), destruye la sesión inmediatamente.

**Layer 3 - Token Binding (Stateless)**
Los JWT tokens incluyen el tenant_id. Un token válido para Razo será rechazado automáticamente si intenta ser usado en Fashion. Esto funciona incluso sin acceso a base de datos.

**Layer 4 - Database Filtering (Defense in Depth)**
En la base de datos misma, cada query filtra automáticamente por tenant_id. Incluso si alguien obtiene credenciales de DB, solo puede ver datos de su tenant.

**Resultado:** 4 niveles de protección hacen que sea prácticamente imposible un data breach entre tenants.

---

## 🧠 El Algoritmo: FIFO Inteligente + Priority Override

### El Problema Real

Un mayorista vende en múltiplos:
- Proveedor A vende en cajas de 12
- Un cliente pequeño pide 50 unidades
- Un cliente VIP pide 30 unidades pero está esperando desde hace poco
- ¿A quién le das el stock?

### La Solución: FIFO Modificado

El sistema utiliza **FIFO (First In, First Out)** con la capacidad de quebrar la fila para clientes VIP:

**Regla 1:** Órdenes normales se cumplen por orden de llegada (FIFO clásico)
**Regla 2:** Órdenes VIP pueden saltarse en la fila
**Regla 3:** Si una VIP toma stock destinado a una normal, la normal se degrada automáticamente a "bajo pedido"

**Ejemplo en números:**
```
Stock: 100 unidades

Orden A (Normal, 50 piezas, hace 5 días)
  → ✅ Surtida completamente (100 - 50 = 50 restante)

Orden B (Normal, 50 piezas, hace 2 días)
  → ⚠️ Parcial (50 surtida, 10 backorder)

Orden C (VIP, 30 piezas, hace 1 día)
  → ✅ Surtida (rompe FIFO, pero es VIP)

Efecto cascada: Orden B ahora es 100% backorder (sistema notifica automáticamente)
```

### Smart Reordering

El sistema también normaliza cantidades según el empaque del proveedor:
- Si el mínimo es 12 y pides 15, compra 24
- El "sobrante" (9 unidades) se usa para otros clientes después
- Esto optimiza compras y reduce costos logísticos

---

## 📊 Sistema de Auditoría Forense

### ¿Qué es el Kardex?

Un registro inmutable de CADA movimiento de inventario:

Cuando algo sucede (compra, venta, merma, ajuste), se registra:
- Fecha y hora exacta
- Quién lo hizo (admin/sistema)
- La razón específica
- Stock anterior y posterior
- IP del usuario

### ¿Por qué es Importante?

Si hay discrepancia (el stock físico no coincide con el teórico), puedes:
- Rastrear exactamente cuándo ocurrió
- Saber quién accedió en ese momento
- Detectar si es error humano o fraude
- Tener trail legal completo para auditorías

### Auditoría Mensual

Cada mes el sistema:
1. Calcula stock teórico (inicial + entradas - salidas - mermas)
2. Se compara con stock físico (conteo manual)
3. Identifica discrepancias
4. Si es pequeña (1-2 unidades) → Aceptable 🟡
5. Si es grande (>2 unidades) → Requiere justificación 🔴

Los admins deben documentar cada discrepancia roja. Sistema crea reporte mensual con tendencias.

---

## 💳 Credit Risk Analysis

### El Desafío

Un cliente quiere $5,000 a crédito. ¿Lo apruebas? ¿Cuánto es seguro?

### La Solución: Scoring Automático

El sistema evalúa automáticamente en segundos:

**Antigüedad del Cliente**
- < 1 mes → Riesgo ALTO
- 3-6 meses → Riesgo MEDIO  
- > 6 meses → Riesgo BAJO

**Historial de Compras**
- Sin historial → ALTO
- Compras pequeñas/inconsistentes → MEDIO
- Compras grandes/regulares → BAJO

**Máximo Histórico**
- Si pide 3x más de lo que ha gastado → MEDIO/ALTO
- Si está dentro de lo normal → BAJO

**Pagos Vencidos**
- Tiene deudas sin pagar → Rechazar automáticamente
- No tiene deudas → Continuar análisis

**Resultado:** Recomendación automática en segundos
- 🟢 BAJO → Aprobar (inmediato)
- 🟡 MEDIO → Revisar manualmente
- 🔴 ALTO → Rechazar (automático)

**Impacto:** En lugar de evaluar manualmente cada solicitud (3-5 horas por semana), el sistema lo hace automáticamente.

---

## 🎯 Casos de Uso Principales

### 1. Cliente Realiza Pedido

Cliente entra → ve catálogo → agrega a carrito → checkout

Sistema valida:
- ¿Hay stock suficiente?
- ¿Cliente tiene crédito disponible?
- ¿Es cliente del tenant correcto?

Si pasa: reserva stock, procesa pago, crea orden de compra al proveedor (automática si hay backorder)

### 2. Admin Recibe Orden de Compra

Admin compra 100 unidades al proveedor → registra recepción en sistema

Sistema automáticamente:
- Actualiza stock global
- Calcula si hay backorders pendientes
- Asigna stock usando FIFO
- Degrada/Surtidiza órdenes automáticamente
- Notifica a clientes: "Tu pedido está listo" o "Sigue en espera"

Resultado: 0 órdenes manuales, todo automático

### 3. Consolidación de Órdenes

5 órdenes pendientes del mismo proveedor con diferentes cantidades

Sistema analiza automáticamente:
- ¿Hay sobrestock si consolidamos?
- ¿Cuánto ahorraríamos?
- ¿Cuál es el impacto en cada orden?

Propone consolidación: "Si agrupas, compras 120 en lugar de 150, ahorras 30 unidades"

Admin aprueba → Sistema crea grupo, mantiene órdenes separadas para billing

---

## 🚀 Stack Tecnológico

### Frontend
- **Lenguaje:** JavaScript Vanilla (ES6+)
- **UI Framework:** Bootstrap 5
- **Validación:** Regex + lógica custom

### Backend
- **Runtime:** Node.js 18+
- **Framework:** Express.js
- **Autenticación:** JWT + Passport.js

### Base de Datos
- **Motor:** PostgreSQL 17+
- **Almacenamiento:** Azure Database for PostgreSQL

### Infraestructura
- **Hosting:** Azure App Service
- **CDN Imágenes:** Cloudinary
- **Pagos:** MercadoPago SDK
- **Email:** Nodemailer SMTP
- **CI/CD:** GitHub Actions

---

## 📚 Documentación Técnica Detallada

Este repositorio contiene documentación sobre:

- **[MULTI_TENANCY.md](./docs/MULTI_TENANCY.md)** - Cómo funciona el aislamiento
- **[SMART_INVENTORY.md](./docs/SMART_INVENTORY.md)** - Algoritmo FIFO y asignación
- **[CREDIT_SYSTEM.md](./docs/CREDIT_SYSTEM.md)** - Análisis de riesgo automático
- **[AUDIT_LOGGING.md](./docs/AUDIT_LOGGING.md)** - Trazabilidad forense
- **[SECURITY_LAYERS.md](./docs/SECURITY_LAYERS.md)** - Las 4 capas de validación
- **[DATABASE_DESIGN.md](./docs/DATABASE_DESIGN.md)** - Schema y decisiones
- **[LESSONS_LEARNED.md](./docs/LESSONS_LEARNED.md)** - Errores y aciertos

---

## 💡 Decisiones Arquitectónicas Clave

### ¿Por qué Multi-Tenant?

**Alternativa:** Monolítica por tenant (3 aplicaciones separadas)
- Ventaja: Aislamiento perfecto
- Desventaja: Triple mantenimiento, triple costo

**Elegida: Multi-tenant**
- Ventaja: Mantenimiento único, costos 60% menores, features escalan a todos
- Desventaja: Complejidad de aislamiento

**Decisión:** Multi-tenant porque el ROI operativo es exponencial.

### ¿Por qué PostgreSQL?

**Alternativa:** MongoDB (NoSQL)
- Ventaja: Flexible, escalable
- Desventaja: ACID débil, integridad en auditoría comprometida

**Elegida: PostgreSQL**
- Ventaja: ACID perfectas, row-level security, triggers, stored procedures
- Desventaja: Menos flexible

**Decisión:** PostgreSQL porque la auditoría requiere garantías de consistencia.

### ¿Por qué JavaScript Vanilla?

**Alternativa:** Framework (React, Vue)
- Ventaja: Componentes reutilizables
- Desventaja: Build step, bundle grande

**Elegida: Vanilla JS**
- Ventaja: Sin dependencias, bundle pequeño, carga rápida
- Desventaja: Más código para features complejas

**Decisión:** Vanilla JS porque el frontend es CRUD y conexiones lentas son problema.

---

## 🎓 Lecciones Aprendidas

### 1. Multi-Tenancy desde el Inicio
Si la agregas después, necesitas reescribir todo. Cada tabla debe tener tenant_id desde el primer migration.

### 2. Auditoría desde Día 1
No esperes a que reguladores lo pidan. Cada movimiento crítico debe estar auditado desde el principio.

### 3. Transacciones ACID son No-Negociables
Si un pago se procesa pero el inventario no se actualiza, tienes caos. ACID garantiza "todo o nada".

### 4. Testing Manual es Mejor que Nada
RazoConnect tiene scripts de test manuales que validan lógica compleja. Deberían ser tests unitarios, pero es mejor tenerlos.

### 5. Documentación es tu Ventaja Competitiva
El código sin documentación es inutilizable. RazoConnect tiene 10+ documentos que permiten onboarding en horas, no semanas.

---

## 📈 Métricas de Éxito

| Métrica | Target | Actual | Status |
|---------|--------|--------|--------|
| Uptime | 99.5% | 99.8% | ✅ |
| Respuesta API | <200ms | 150ms | ✅ |
| Concurrent Users | 500+ | 500+ | ✅ |
| Errores de Auditoría | 0 | 0 | ✅ |
| Discrepancias Inventario | <0.5% | 0.3% | ✅ |

---

## Conclusión

RazoConnect es una demostración de cómo construir:

✅ **Sistemas escalables** para múltiples usuarios simultáneamente  
✅ **Arquitecturas seguras** con validación en capas  
✅ **Lógica inteligente** que automatiza decisiones  
✅ **Auditoría completa** para compliance legal  
✅ **Documentación** que permite onboarding rápido  

**El código es privado porque genera ingresos, pero la arquitectura es tu mejor portfolio.**

---

**Última actualización:** Febrero 2026  
**Versión:** 1.0 - Case Study público
