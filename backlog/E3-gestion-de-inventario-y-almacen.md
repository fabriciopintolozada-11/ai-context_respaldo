# Épica E3 — Gestión de Inventario y Almacén

## Objetivo

Controlar el catálogo y las existencias de repuestos del único almacén del taller, diferenciando stock físico, reservado y disponible. E3 cubre reservas, consumo efectivo, liberación de reservas, devoluciones internas, ajustes por diferencias físicas y alertas de rotación o disponibilidad.

E3 no incluye compras, proveedores, facturación fiscal ni integración con sistemas externos. El ingreso inicial o extraordinario de existencias se registra únicamente como un ajuste de inventario auditable.

## Alcance y modelo de stock

- `physicalStock`: existencias teóricas disponibles físicamente en el almacén.
- `reservedStock`: unidades reservadas para OTs aprobadas y aún no consumidas.
- `availableStock`: `physicalStock - reservedStock`.
- Una reserva no descuenta `physicalStock`.
- Un consumo confirmado descuenta `physicalStock` y `reservedStock` en la misma transacción.
- Una liberación o devolución reduce `reservedStock`; una devolución física aumenta `physicalStock` si las unidades regresan al almacén.
- Ninguna operación puede producir saldos negativos.
- Todo movimiento se registra en el kardex inmutable con usuario, fecha, OT cuando corresponda, cantidad, saldos anterior/nuevo y motivo.

## Historias de usuario

| ID | Título | Prioridad | Puntos |
| --- | --- | --- | --- |
| US-07 | Confirmar uso e instalación de repuestos | Must | 3 |
| US-08 | Consultar alertas de rotación y disponibilidad | Must | 2 |
| US-13 | Gestionar espera de repuesto | Must | 5 |
| US-14 | Registrar ajustes físicos de inventario | Must | 3 |
| US-23 | Consultar y gestionar catálogo de repuestos | Must | 5 |

## Trazabilidad

- `RN-01`, `RN-02`, `RN-04`, `RN-05`, `RN-07`, `RN-08`, `RN-09`, `RN-10`, `RN-14`, `RN-16`, `RN-19`, `RN-21`.
- `SUP-02`, `SUP-04`, `SUP-12`.
- Preguntas `P-05`, `P-09`, `P-13`, `P-16`, `P-20`, `P-25`, `P-26`.
- E1 crea y asigna la OT; E2 registra diagnóstico, presupuesto y aprobación; E3 controla inventario; E4 comunica los estados de espera.

## US-07: Confirmar uso e instalación de repuestos

**Como** Mecánico, **quiero** confirmar la instalación y uso efectivo de un repuesto reservado en mi OT, **para** actualizar de forma atómica las existencias físicas y la trazabilidad del almacén.

| Prioridad | Puntos | Rol | Depende de |
| --- | --- | --- | --- |
| Must | 5 | Mecánico asignado; Jefe de Taller en supervisión | US-09, US-11 |

### Criterios de aceptación

```gherkin
Escenario: Consumo total de un repuesto reservado
  Dado que la OT está aprobada y en estado "en_reparacion"
  Y la OT está asignada al mecánico autenticado
  Y el repuesto tiene una cantidad reservada suficiente
  Cuando el mecánico confirma la instalación de la cantidad indicada
  Entonces el sistema reduce physicalStock y reservedStock en esa cantidad dentro de una transacción
  Y marca el consumo como "consumido" cuando se agota la cantidad reservada
  Y registra el movimiento en el kardex inmutable con usuario, OT y fecha

Escenario: Consumo parcial de un repuesto reservado
  Dado que la OT tiene una cantidad reservada mayor que la cantidad instalada
  Cuando el mecánico confirma una cantidad positiva menor que la pendiente
  Entonces el sistema registra únicamente esa cantidad como consumida
  Y conserva la cantidad restante como reservada

Escenario: Consumo sin aprobación o sin reserva
  Dado que la OT no está aprobada o el repuesto no está reservado para ella
  Cuando se intenta confirmar el uso
  Entonces el sistema rechaza la operación con 422
  Y no modifica existencias, reservas ni kardex

Escenario: Stock físico insuficiente
  Dado que la cantidad física disponible es menor que la cantidad a consumir
  Cuando se confirma la instalación
  Entonces el sistema rechaza la operación con 422
  Y no permite saldos negativos ni cambios parciales

Escenario: Liberación de una reserva no consumida
  Dado que una OT aprobada libera una cantidad reservada que ya no será utilizada
  Cuando el usuario autorizado registra la liberación
  Entonces el sistema reduce reservedStock en esa cantidad
  Y aumenta availableStock sin modificar physicalStock
  Y registra la liberación en el kardex vinculada a la OT y con su motivo

Escenario: Devolución física de un sobrante
  Dado que una cantidad reservada fue parcialmente consumida y quedan unidades sin instalar
  Cuando el usuario autorizado confirma la devolución física al almacén
  Entonces el sistema reduce reservedStock y aumenta physicalStock en la cantidad devuelta
  Y registra la devolución en el kardex sin permitir cantidades superiores a la pendiente

Escenario: Ocultamiento de precios para mecánicos
  Dado que el usuario tiene rol "MECHANIC"
  Cuando consulta repuestos reservados o consumidos
  Entonces puede ver código, nombre y cantidades
  Y no recibe costo, precio ni totales monetarios en la respuesta de API o interfaz
```

### Reglas y dependencias técnicas

- El mecánico solo opera sobre una OT asignada (`RN-04`); el jefe solo puede actuar como usuario autorizado de supervisión.
- La operación valida estado de OT, pertenencia de la OT, reserva, cantidad y stock dentro de una transacción (`RN-02`, `RN-08`, `RN-09`).
- Endpoint: `POST /api/v1/work-orders/:id/consume-part`.
- DTO: `ConsumeSparePartDto` con `workOrderPartId`, `quantity` y `notes`.
- El kardex es de solo inserción (`RN-19`).

## US-13: Gestionar espera de repuesto

**Como** Mecánico, **quiero** poner mi OT en espera cuando una pieza requerida no está físicamente disponible, **para** registrar la detención, informar al taller y quedar disponible para otra OT.

| Prioridad | Puntos | Rol | Depende de |
| --- | --- | --- | --- |
| Must | 3 | Mecánico asignado; Jefe de Taller | US-09, US-11, US-23 |

### Criterios de aceptación

```gherkin
Escenario: Registrar espera por diferencia de stock
  Dado que la OT está aprobada, en estado "en_reparacion" y asignada al mecánico
  Y la pieza requerida está asociada a la OT
  Cuando el mecánico registra que la pieza no está en el estante
  Y indica pieza, cantidad faltante y motivo
  Entonces la OT cambia a "en_espera_de_repuesto"
  Y conserva el vehículo y la bahía como ocupados hasta una decisión del jefe
  Y el mecánico queda disponible para otra OT
  Y se registra una inconsistencia de inventario pendiente de ajuste

Escenario: Visibilidad de la espera
  Dado que una OT está en "en_espera_de_repuesto"
  Cuando Recepción o Jefatura consulta el seguimiento
  Entonces ve la pieza faltante, motivo, fecha y días de espera
  Y el estado queda disponible para informar al cliente mediante E4

Escenario: Transición inválida
  Dado que la OT no está en "en_reparacion" o la pieza no pertenece a la OT
  Cuando se intenta registrar la espera
  Entonces el sistema rechaza la operación con 409 o 422
  Y no crea una inconsistencia
```

### Reglas y dependencias técnicas

- La transición no libera automáticamente la bahía ni cancela la reserva.
- Endpoint: `POST /api/v1/work-orders/:id/awaiting-part`.
- DTO: `SetAwaitingPartDto` con `missingPartId`, `quantity` y `reason`.
- El motivo, la pieza, cantidad y usuario quedan en el historial inmutable de la OT.

## US-08: Consultar alertas de rotación y disponibilidad

**Como** Jefe de Taller, **quiero** consultar alertas sobre repuestos sin rotación y disponibilidad insuficiente, **para** decidir ajustes operativos y evitar retrasos.

| Prioridad | Puntos | Rol | Depende de |
| --- | --- | --- | --- |
| Must | 5 | Jefe de Taller, Administrador | US-07, US-23 |

### Criterios de aceptación

```gherkin
Escenario: Alerta de repuesto sin rotación
  Dado que un repuesto tiene stock físico positivo
  Y no registra movimientos de salida o consumo durante 60 días o más
  Cuando Jefatura consulta las alertas
  Entonces el sistema lo marca como "Sin Rotación / Estancado"
  Y muestra último movimiento, días sin movimiento y stock actual

Escenario: Alerta de disponibilidad insuficiente
  Dado que availableStock es menor o igual a cero por efecto de reservas
  Cuando Jefatura consulta las alertas
  Entonces visualiza "Stock Crítico / Insuficiente"
  Y ve stock físico, reservado y faltante estimado

Escenario: Acceso restringido
  Dado que un Mecánico o Recepcionista consulta el endpoint de alertas
  Cuando el backend procesa la petición
  Entonces responde 403
  Y la interfaz no muestra el acceso a alertas
```

### Reglas y dependencias técnicas

- La rotación usa `lastMovementAt` como única fuente de verdad y considera movimientos de salida o consumo.
- El umbral de disponibilidad crítica es `availableStock <= 0`; no se introduce un umbral mínimo configurable fuera del alcance confirmado.
- Endpoint: `GET /api/v1/inventory/alerts` con filtros paginados.
- Respuesta: `{ data, total, page, pageSize }` con `alertType: NO_ROTATION | STOCK_OUT`.

## US-14: Registrar ajustes físicos de inventario

**Como** Jefe de Taller, **quiero** registrar una diferencia entre el stock teórico y el conteo físico, **para** corregir el saldo sin alterar el historial de movimientos.

| Prioridad | Puntos | Rol | Depende de |
| --- | --- | --- | --- |
| Must | 3 | Jefe de Taller, Administrador | US-13, US-23 |

### Criterios de aceptación

```gherkin
Escenario: Ajuste positivo o negativo autorizado
  Dado que un repuesto tiene una diferencia física comprobada
  Cuando Jefatura registra cantidad, tipo y motivo del ajuste
  Entonces el sistema actualiza physicalStock sin modificar reservedStock
  Y recalcula availableStock
  Y registra el movimiento de ajuste en el kardex con usuario, fecha y saldos anterior/nuevo

Escenario: Ajuste que produce stock negativo o afecta reservas
  Dado que el ajuste dejaría physicalStock por debajo de reservedStock o de cero
  Cuando se intenta guardar
  Entonces el sistema rechaza la operación con 422
  Y no modifica el inventario

Escenario: Ajuste no autorizado
  Dado que el usuario no es Jefe de Taller ni Administrador
  Cuando intenta registrar un ajuste
  Entonces el backend responde 403
```

### Reglas y dependencias técnicas

- Endpoint: `POST /api/v1/inventory/adjustments`.
- DTO: `CreateInventoryAdjustmentDto` con `sparePartId`, `quantity`, `type` y `reason`.
- Un ajuste no elimina ni edita movimientos previos.
- Este flujo cubre también el registro de existencias iniciales o extraordinarias sin crear un módulo de compras.

## US-23: Consultar y gestionar catálogo de repuestos

**Como** Jefe de Taller o Recepcionista, **quiero** consultar el catálogo y la disponibilidad de repuestos, y como Jefe de Taller o Administrador quiero registrar o desactivar piezas, **para** trabajar con información actualizada del almacén.

| Prioridad | Puntos | Rol | Depende de |
| --- | --- | --- | --- |
| Must | 5 | Recepcionista, Mecánico, Jefe de Taller, Administrador | US-00 |

### Criterios de aceptación

```gherkin
Escenario: Consulta de stock
  Dado que un usuario autenticado consulta el catálogo
  Cuando carga o filtra la lista
  Entonces visualiza código, nombre, categoría, physicalStock, reservedStock y availableStock
  Y solo los roles autorizados para precios reciben priceInBob

Escenario: Alta de repuesto
  Dado que Jefatura o Administración abre el formulario
  Cuando registra código único, nombre, categoría, precio en BOB y stock inicial no negativo
  Entonces se crea el repuesto activo
  Y el stock inicial queda respaldado por un movimiento de ajuste

Escenario: Desactivación de repuesto
  Dado que un repuesto no tiene unidades reservadas ni pendientes de consumo
  Cuando Jefatura o Administración lo desactiva
  Entonces deja de estar disponible para nuevas cotizaciones
  Y conserva su historial y movimientos

Escenario: Restricción de catálogo
  Dado que el usuario es Mecánico o Recepcionista
  Cuando intenta crear, desactivar o editar un repuesto
  Entonces el backend responde 403
  Y la interfaz no muestra esas acciones

Escenario: Ocultamiento de precios
  Dado que el usuario tiene rol "MECHANIC"
  Cuando consulta el catálogo
  Entonces no recibe precios ni importes monetarios
```

### Reglas y dependencias técnicas

- Endpoint de consulta: `GET /api/v1/inventory/spare-parts`.
- Alta: `POST /api/v1/inventory/spare-parts`.
- Desactivación: `POST /api/v1/inventory/spare-parts/:id/deactivate`.
- Catálogo y movimientos usan `SparePart`, `WorkOrderPart` y `StockMovement`.
- `reservedStock` solo cambia mediante reservas, liberaciones, devoluciones o consumos transaccionales; nunca mediante edición manual del catálogo.
- Los listados responden paginados y las búsquedas por código/nombre/categoría se ejecutan con debounce de 300 ms en la interfaz.

## Definition of Done específica de E3

- Cada criterio Gherkin tiene pruebas unitarias y e2e.
- Las operaciones de reserva, consumo, liberación, devolución y ajuste son atómicas.
- Los movimientos del kardex son inmutables.
- Ningún DTO para `MECHANIC` contiene precios.
- Se validan saldos no negativos, permisos, asignación de OT y transiciones de estado.
- La interfaz funciona en escritorio y tablet.
