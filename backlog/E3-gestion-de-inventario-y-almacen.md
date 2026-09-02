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
| US-07 | Confirmar uso e instalación de repuestos | Must | 5 |
| US-08 | Consultar alertas de rotación y disponibilidad | Must | 5 |
| US-13 | Gestionar espera de repuesto | Must | 3 |
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

**Backend (`modules/work-orders`, `modules/inventory`):**

- **BE-T07.1:** crear `POST /api/v1/work-orders/:id/consume-part`, protegido por `JwtAuthGuard` global y `@Roles(Role.MECHANIC, Role.WORKSHOP_LEAD)`, con decoradores Swagger (BE-25).
- **BE-T07.2:** crear `ConsumeSparePartDto` con `class-validator`: `workOrderPartId` como UUID, `quantity` como entero positivo y `notes` opcional y acotado (BE-10, BE-11).
- **BE-T07.3:** implementar en el service una `$transaction` coordinada con repositorios: validar estado `EN_REPARACION`, presupuesto aprobado y que el mecánico autenticado esté asignado; permitir al `WORKSHOP_LEAD` solo como supervisor autorizado. Validar que `WorkOrderPart` pertenece a la OT, conserva saldo reservado suficiente y está reservado; descontar `physicalStock` y `reservedStock` sin saldos negativos; incrementar su cantidad consumida y marcarlo `INSTALLED` únicamente al agotar el saldo reservado; insertar el `StockMovement` inmutable de tipo `CONSUMPTION` con usuario, OT, cantidad, saldos y fecha (RN-02, RN-04, RN-07 a RN-09, RN-19; BE-08, BE-16, BE-17, BE-19).
- **BE-T07.4:** crear `WorkOrderPartResponseDto` y el mapeo por rol que omita sistemáticamente precio, costo y totales monetarios para `MECHANIC` (RN-16, BE-12).
- **BE-T07.5:** escribir pruebas unitarias Jest y e2e con Supertest para consumo total y parcial, atomicidad ante fallo, rechazo `422` para OT no aprobada, reserva insuficiente o stock físico insuficiente, y rechazo `403` para mecánico no asignado (BE-31, BE-32).
- La transición no libera automáticamente la bahía ni cancela la reserva.
- Endpoint: `POST /api/v1/work-orders/:id/awaiting-part`.
- DTO: `SetAwaitingPartDto` con `missingPartId`, `quantity` y `reason`.
- El motivo, la pieza, cantidad y usuario quedan en el historial inmutable de la OT.

## US-08: Consultar alertas de rotación y disponibilidad

**Como** Jefe de Taller, **quiero** consultar alertas sobre repuestos sin rotación y disponibilidad insuficiente, **para** decidir ajustes operativos y evitar retrasos.

- **FE-T07.1:** crear `ReservedPartsPanel.tsx` para la OT del mecánico, con acciones táctiles de al menos `44x44 px` y botón accesible `Confirmar uso` (FE-14).
- **FE-T07.2:** crear `useConsumeSparePart` en `features/work-orders/api/` con `@tanstack/react-query` y el cliente HTTP centralizado; al confirmar, invalidar detalle de OT, lista del mecánico, catálogo y alertas de inventario (FE-03, FE-08, FE-09).
- **FE-T07.3:** omitir el renderizado de costos, precios unitarios y totales monetarios en toda vista de mecánico, incluso si la respuesta fuera manipulada en cliente (RN-16, FE-18).
- **FE-T07.4:** mostrar una notificación de éxito accesible y no intrusiva tras el consumo, y mostrar errores `403` y `422` contextualizados (FE-04, FE-16).
- **FE-T07.5:** crear pruebas con Vitest, Testing Library y MSW para confirmar el consumo, invalidar/refrescar datos y ocultar precios al mecánico (FE-20 a FE-22).
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

**Como** Mecánico, **quiero** poner mi OT en espera cuando una pieza requerida no está físicamente disponible, **para** registrar la detención, informar al taller y quedar disponible para otra OT.

| Prioridad | Puntos | Rol | Depende de |
| --- | --- | --- | --- |
| Must | 3 | Mecánico asignado; Jefe de Taller | US-09, US-11, US-23 |
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
Escenario: Registrar espera por diferencia de stock
  Dado que la OT está aprobada, en estado "en_reparacion" y asignada al mecánico
  Y la pieza requerida está asociada a la OT
  Cuando el mecánico registra que la pieza no está en el estante
  Y indica pieza, cantidad faltante y motivo
  Entonces la OT cambia a "en_espera_de_repuesto"
  Y conserva el vehículo y la bahía como ocupados hasta una decisión del jefe
  Y el mecánico queda disponible para otra OT
  Y si el stock teórico de la pieza es positivo, se registra una inconsistencia de inventario pendiente de ajuste

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
- DTO: `SetAwaitingPartDto` con `workOrderPartId`, `quantity` y `reason`.
- El motivo, la pieza, cantidad y usuario quedan en el historial inmutable de la OT.
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

- **BE-T13.1:** crear `SetAwaitingPartDto` validado con `class-validator`: `workOrderPartId` UUID, `quantity` entero positivo y `reason` obligatorio y acotado (BE-10, BE-11).
- **BE-T13.2:** crear `POST /api/v1/work-orders/:id/awaiting-part` para `MECHANIC` y `WORKSHOP_LEAD`; validar que el mecánico opera su propia OT y que el jefe actúa como supervisor autorizado (RN-04, BE-25, BE-29).
- **BE-T13.3:** implementar una `$transaction` de service que valide la OT aprobada en `EN_REPARACION`, que el `WorkOrderPart` pertenece a ella y que la cantidad reportada no supera el saldo reservado; después transicionar la OT a `EN_ESPERA_DE_REPUESTO` sin liberar la bahía ni cancelar la reserva (RN-02, RN-05, RN-07, BE-16).
- **BE-T13.4:** insertar en `WorkOrderStatusHistory` el motivo, repuesto, cantidad, usuario y fecha sin editar historiales previos (RN-19, BE-17, BE-19).
- **BE-T13.5:** dentro de la misma transacción, crear una `InventoryDiscrepancy` pendiente con el `sparePartId` derivado de `WorkOrderPart`, OT, usuario, cantidad y motivo únicamente si el stock teórico positivo no coincide con la falta física reportada; la posterior corrección se gestiona mediante US-14 (SUP-04).
- **BE-T13.6:** escribir pruebas unitarias y e2e para la transición válida, OT o pieza inválida (`409`/`422`), rechazo al mecánico no asignado y creación condicional de la discrepancia (BE-31, BE-32).
- Endpoint de consulta: `GET /api/v1/inventory/spare-parts`.
- Alta: `POST /api/v1/inventory/spare-parts`.
- Desactivación: `POST /api/v1/inventory/spare-parts/:id/deactivate`.
- Catálogo y movimientos usan `SparePart`, `WorkOrderPart` y `StockMovement`.
- `reservedStock` solo cambia mediante reservas, liberaciones, devoluciones o consumos transaccionales; nunca mediante edición manual del catálogo.
- Los listados responden paginados y las búsquedas por código/nombre/categoría se ejecutan con debounce de 300 ms en la interfaz.

## Definition of Done específica de E3

- **FE-T13.1:** crear `AwaitingPartModal.tsx` táctil con controles de al menos `44x44 px`, selector de repuesto reservado, cantidad y justificación mediante `react-hook-form` y Zod (FE-11, FE-14).
- **FE-T13.2:** crear `useSetAwaitingPart` con React Query; invalidar detalle de OT, lista del mecánico, tablero de bahías y alertas de inventario después de una mutación exitosa (FE-08, FE-09).
- **FE-T13.3:** representar el estado con un badge ámbar no intrusivo, pieza faltante y tiempo de espera en el tablero de OTs (FE-13, FE-16).
- **FE-T13.4:** crear pruebas con Vitest, Testing Library y MSW para validación del formulario, transición y visualización de espera (FE-20 a FE-22).

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

### Desglose de tareas técnicas

**Backend (`modules/inventory`):**

- **BE-T08.1:** crear `GET /api/v1/inventory/alerts` paginado (`page`, `limit`) con filtro `alertType`, protegido para `WORKSHOP_LEAD` y `ADMIN`, y documentarlo en Swagger (BE-24, BE-25, BE-29).
- **BE-T08.2:** implementar en el repositorio una consulta indexada sobre `SparePart.lastMovementAt` para identificar piezas con `physicalStock > 0` y sin salida o consumo durante 60 días o más; no usar la fecha de creación ni movimientos de entrada como sustituto (RN-10, BE-09, BE-21).
- **BE-T08.3:** calcular `availableStock = physicalStock - reservedStock` y generar alerta `STOCK_OUT` solo cuando sea menor o igual a cero; no introducir `minSafetyStock` ni `LOW_STOCK` porque no forman parte del alcance confirmado.
- **BE-T08.4:** crear `InventoryAlertResponseDto` con `partId`, `code`, `name`, `category`, `physicalStock`, `reservedStock`, `availableStock`, `daysWithoutMovement` y `alertType: NO_ROTATION | STOCK_OUT`.
- **BE-T08.5:** escribir pruebas unitarias y e2e para ambas alertas, paginación/filtro y rechazo `403` para `MECHANIC` y `RECEPTIONIST` (BE-31, BE-32).

**Frontend (`features/inventory`):**

- **FE-T08.1:** crear `InventoryAlertsList.tsx` con badges visuales: rojo para `STOCK_OUT` y ámbar para `NO_ROTATION` de 60 días o más (FE-16).
- **FE-T08.2:** crear `useInventoryAlerts` con `@tanstack/react-query`, `staleTime: 5 * 60 * 1000` y el cliente HTTP centralizado (FE-03, FE-08).
- **FE-T08.3:** incluir filtro por tipo, búsqueda por nombre/código con debounce de 300 ms y paginación; el parámetro de búsqueda debe estar soportado por el contrato del endpoint.
- **FE-T08.4:** renderizar skeleton, error con reintento y estado vacío: `No se registran alertas de inventario ni repuestos estancados` (FE-13).
- **FE-T08.5:** crear pruebas con Vitest, Testing Library y MSW para filtros, estados de vista y acceso oculto a roles no autorizados (FE-18, FE-20 a FE-22).

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

### Desglose de tareas técnicas

**Backend (`modules/inventory`):**

- **BE-T14.1:** crear `CreateInventoryAdjustmentDto` con `sparePartId` UUID, `quantity` entero positivo, `type: POSITIVE | NEGATIVE`, `reason` obligatorio e `inventoryDiscrepancyId` UUID opcional; los datos de auditoría nunca se aceptan en el payload (BE-10, BE-19).
- **BE-T14.2:** crear `POST /api/v1/inventory/adjustments` restringido a `WORKSHOP_LEAD` y `ADMIN`, con documentación Swagger (BE-25, BE-29).
- **BE-T14.3:** implementar una `$transaction` con el bloqueo de fila PostgreSQL encapsulado en el repositorio. Validar que el ajuste no deje `physicalStock` negativo ni por debajo de `reservedStock`; actualizar el stock y registrar un `StockMovement` inmutable de tipo `ADJUSTMENT` con dirección, usuario, stock anterior/nuevo, motivo y fecha. Retornar `422` ante saldo inválido o conflicto concurrente (BE-08, BE-16, BE-17, BE-23).
- **BE-T14.4:** cuando el payload referencia una discrepancia pendiente, verificar que corresponde al mismo repuesto y resolverla dentro de la misma transacción; si no se referencia una discrepancia, el ajuste permanece independiente (SUP-04).
- **BE-T14.5:** escribir pruebas unitarias y e2e para ajustes positivo/negativo, rechazo `422` por reservas o concurrencia y rechazo `403` por rol (BE-31, BE-32).

**Frontend (`features/inventory`):**

- **FE-T14.1:** crear `InventoryAdjustmentModal.tsx`, accesible desde el catálogo y la lista de alertas, para usuarios autorizados; al abrirse desde una discrepancia, enviar su identificador en la mutación.
- **FE-T14.2:** implementar el formulario con `react-hook-form` y Zod: cantidad entera positiva y justificación mínima de 10 caracteres (FE-11).
- **FE-T14.3:** bloquear preventivamente un ajuste negativo que supere `physicalStock - reservedStock`, sin sustituir la validación transaccional del backend (FE-12).
- **FE-T14.4:** crear `useCreateAdjustment`; invalidar catálogo, alertas y detalle del repuesto al finalizar correctamente (FE-08, FE-09).
- **FE-T14.5:** renderizar la acción únicamente a `WORKSHOP_LEAD` y `ADMIN`, y probar la restricción de interfaz con Vitest, Testing Library y MSW (FE-18, FE-20 a FE-22).

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

### Desglose de tareas técnicas

**Backend (`modules/inventory`, `prisma`):**

- **BE-T23.1:** definir `SparePart` en Prisma con `code`, `name`, `category`, `physicalStock`, `reservedStock`, `priceInBob Decimal @db.Decimal(10, 2)`, `lastMovementAt` e `isActive`; mapear tabla/columnas a `snake_case` e indexar `code`, `name`, `category` y `lastMovementAt` (BE-13, BE-18, BE-21).
- **BE-T23.2:** crear `GET /api/v1/inventory/spare-parts` paginado con búsqueda textual por código/nombre y filtro por categoría, retornando `{ data, total, page, pageSize }` (BE-24).
- **BE-T23.3:** crear la carga inicial idempotente de aproximadamente 300 repuestos ficticios en `tools/scripts/seed/`, con `--dry-run`, precios BOB y fechas de rotación variadas; debe respetar las reglas de inventario y emitir el resumen de importación exigido (TL-05, TL-06, TL-12, TL-13).
- **BE-T23.4:** definir DTOs de respuesta y mapeo por rol para omitir `priceInBob` y cualquier importe monetario cuando el solicitante es `MECHANIC` (RN-16, RN-21, BE-12).
- **BE-T23.5:** incorporar pruebas de repositorio para filtros e índices consultados y una prueba de rendimiento con el catálogo semilla; documentar el presupuesto de menos de 100 ms como objetivo medido en base de datos local, no como garantía absoluta de red.
- **BE-T23.6:** implementar alta y desactivación lógica mediante `POST /api/v1/inventory/spare-parts` y `POST /api/v1/inventory/spare-parts/:id/deactivate`, restringidos a `WORKSHOP_LEAD` y `ADMIN`; el alta con stock inicial debe generar un movimiento de ajuste y la desactivación debe bloquearse con reservas pendientes (BE-17, BE-18, BE-29).

**Frontend (`features/inventory`):**

- **FE-T23.1:** crear `InventoryCatalogPage.tsx` con tabla interactiva, filtro por categoría, búsqueda por código/nombre con debounce de 300 ms y paginación.
- **FE-T23.2:** mostrar badges de existencias calculados desde `availableStock`: verde disponible, amarillo reservado parcialmente y rojo sin disponibilidad; no inventar umbrales de stock mínimo no definidos.
- **FE-T23.3:** crear `useInventoryList` con `@tanstack/react-query`, cliente HTTP centralizado y estados de carga, error con reintento y vacío descriptivo (FE-03, FE-08, FE-13).
- **FE-T23.4:** ocultar la columna `Precio (BOB)`, importes y acciones de administración al `MECHANIC`; el backend sigue siendo la fuente de seguridad (RN-16, FE-18).
- **FE-T23.5:** crear formularios de alta y desactivación para `WORKSHOP_LEAD` y `ADMIN` con `react-hook-form` y Zod, y pruebas con Vitest, Testing Library y MSW para catálogo, permisos y ocultamiento de precios (FE-11, FE-20 a FE-22).

## Definition of Done específica de E3

- Cada criterio Gherkin tiene pruebas unitarias y e2e.
- Las operaciones de reserva, consumo, liberación, devolución y ajuste son atómicas.
- Los movimientos del kardex son inmutables.
- Ningún DTO para `MECHANIC` contiene precios.
- Se validan saldos no negativos, permisos, asignación de OT y transiciones de estado.
- La interfaz funciona en escritorio y tablet.
