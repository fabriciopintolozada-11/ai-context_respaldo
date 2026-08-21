# Épica E2 — Presupuestos y Aprobación del Cliente

Gestiona el flujo técnico y administrativo desde que el mecánico inspecciona el vehículo hasta que el cliente autoriza formalmente la intervención, asegurando el cálculo en Bolivianos (BOB) y la reserva de repuestos.

Origen: reglas de negocio de taller automotriz (RN-02, RN-03, RN-04, RN-07, RN-08, RN-16, RN-19, RN-21) sobre transparencia de precios, auditoría técnica, control de repuestos y validación obligatoria del cliente antes de intervenir el vehículo.

Prioridad de la épica: **Must** — sin diagnóstico estructurado, costeo formal y autorización expresa del cliente, el taller no puede iniciar labores ni garantizar el control de stock ni la facturación.

## Historias de usuario

| ID | Título | Prioridad | Puntos |
| --- | --- | --- | --- |
| US-11 | Registrar diagnóstico técnico | Must | 5 |
| US-12 | Generar presupuesto | Must | 3 |
| US-09 | Registrar aprobación o rechazo del presupuesto | Must | 5 |

## US-11: Registrar diagnóstico técnico

**Como** Mecánico, **quiero** registrar el diagnóstico técnico, hallazgos y la lista preliminar de repuestos y tareas requeridas en la Orden de Trabajo, **para** dejar constancia de las fallas encontradas y permitir la cotización del servicio.

| Prioridad | Puntos | Rol | Depende de |
| --- | --- | --- | --- |
| Must | 5 | Mecánico | Recepción de vehículo / OT creada |

### Criterios de aceptación

```gherkin
Escenario: Registro exitoso de diagnóstico técnico
  Dado que la OT está en estado "recibido" o "en_diagnostico" y asignada al mecánico autenticado (RN-04)
  Cuando el mecánico ingresa el detalle de fallas detectadas, horas estimadas de mano de obra y selecciona los repuestos sugeridos del catálogo
  Y confirma el envío del diagnóstico
  Entonces la OT actualiza su estado a "en_diagnostico" (o lista para presupuesto)
  Y los ítems y horas quedan registrados sin mostrar precios de venta al mecánico (RN-16)
  Y el historial técnico del vehículo registra la actividad como inmutable (RN-19)

Escenario: Detección de fallas adicionales durante una reparación en curso (RN-03)
  Dado que la OT se encuentra en estado "en_reparacion"
  Cuando el mecánico detecta un daño adicional imprevisto y registra un nuevo hallazgo
  Entonces la OT se suspende de forma automática y cambia su estado a "presupuesto_enviado" (o pendiente de re-cotización)
  Y se bloquea la confirmación de nuevos repuestos hasta recibir una nueva aprobación explícita del cliente (RN-02, RN-03)
```

### Reglas de negocio y consideraciones técnicas asociadas

- **RN-04 / RN-16:** el mecánico opera en vista técnica sin visibilidad de tarifas ni precios de venta.
- **RN-03:** bloqueo y re-cotización mandatoria ante imprevistos en bahía.
- **Backend:** `CreateDiagnosticDto`, endpoint `POST /api/v1/work-orders/:id/diagnostic` y control transaccional de suspensión.
- **Frontend:** interfaz táctil `DiagnosticForm.tsx` con componentes UI táctiles (`>= 44px`) mediante `react-hook-form` + `zod` e invalidación en React Query.

### Desglose de tareas técnicas

**Backend (`modules/work-orders`, `modules/quotes`):**

- **BE-T11.1:** crear `CreateDiagnosticDto` con `class-validator` (`description`, `suggestedTasks`, `suggestedPartIds`, `estimatedHours`).
- **BE-T11.2:** endpoint `POST /api/v1/work-orders/:id/diagnostic` validando que la OT pertenece al mecánico (RN-04).
- **BE-T11.3:** suspensión automática y transición de estado si la OT estaba `en_reparacion` (RN-03).

**Frontend (`features/work-orders`):**

- **FE-T11.1:** pantalla táctil `DiagnosticForm.tsx` con botones de al menos 44 px, sin campos de precio (FE-14, RN-16).
- **FE-T11.2:** formulario con `react-hook-form` + `zod` (FE-11).
- **FE-T11.3:** mutación con React Query que invalide el estado de la OT (FE-09).

## US-12: Generar presupuesto

**Como** Recepcionista, **quiero** estructurar y costear el presupuesto formal en base al diagnóstico técnico del mecánico, **para** comunicar al cliente el costo total y el desglose de trabajos antes de iniciar cualquier labor.

| Prioridad | Puntos | Rol | Depende de |
| --- | --- | --- | --- |
| Must | 3 | Recepcionista, Jefe de Taller, Administrador | US-11 |

### Criterios de aceptación

```gherkin
Escenario: Generación correcta del presupuesto en moneda nacional
  Dado que existe una OT con diagnóstico registrado y pendiente de presupuesto
  Cuando la Recepcionista revisa las horas de mano de obra y los repuestos solicitados
  Y el sistema calcula los subtotales usando los precios de lista del catálogo
  Entonces todos los montos se presentan exclusivamente en Bolivianos (BOB) (RN-21)
  Y el estado de la OT pasa a "presupuesto_enviado"
  Y la OT queda visible en el panel de seguimiento de bahías y tiempos de espera
```

### Reglas de negocio y consideraciones técnicas asociadas

- **RN-21:** manejo estricto de montos monetarios en moneda nacional (BOB) con precisión decimal (`Prisma.Decimal`).
- **Backend:** DTOs `CreateQuoteDto` y `QuoteResponseDto`, endpoint `POST /api/v1/work-orders/:id/quote` protegido por roles (`RECEPTIONIST`, `WORKSHOP_LEAD`, `ADMIN`).
- **Frontend:** pantalla `QuoteBuilderPage.tsx` con cálculo reactivo de subtotales, validación numérica positiva con Zod y exportación/resumen para envío vía WhatsApp o formato imprimible.

### Desglose de tareas técnicas

**Backend (`modules/quotes`, `modules/settlements`):**

- **BE-T12.1:** crear `CreateQuoteDto` y `QuoteResponseDto` manejando precios con `Decimal` de Prisma (BE-13, RN-21).
- **BE-T12.2:** endpoint `POST /api/v1/work-orders/:id/quote` accesible por `RECEPTIONIST`, `WORKSHOP_LEAD` y `ADMIN`.
- **BE-T12.3:** transición de la máquina de estados de la OT a `PRESUPUESTO_ENVIADO`.

**Frontend (`features/quotes`):**

- **FE-T12.1:** vista `QuoteBuilderPage.tsx` con desglose, selector de repuestos y cálculo dinámico de subtotales y total en BOB.
- **FE-T12.2:** resumen imprimible o compartible para envío por WhatsApp/teléfono al cliente.
- **FE-T12.3:** validación de campos numéricos positivos con Zod (FE-11).

## US-09: Registrar aprobación o rechazo del presupuesto

**Como** Recepcionista, **quiero** registrar la decisión formal (aprobación o rechazo) del cliente especificando el canal de comunicación, **para** habilitar el inicio de los trabajos mecánicos y reservar los repuestos requeridos.

| Prioridad | Puntos | Rol | Depende de |
| --- | --- | --- | --- |
| Must | 5 | Recepcionista, Administrador | US-12 |

### Criterios de aceptación

```gherkin
Escenario: Aprobación total del presupuesto y reserva de inventario
  Dado que la OT está en estado "presupuesto_enviado"
  Cuando la Recepcionista registra la aprobación del cliente indicando el medio ("Llamada", "WhatsApp", "Presencial") y notas de respaldo
  Entonces la OT cambia su estado a "aprobado"
  Y el sistema pasa automáticamente los repuestos presupuestados al estado "Reservado" (RN-07)
  Y las existencias disponibles se reducen para otras órdenes pero el stock físico real no se descuenta aún (RN-07, RN-08)
  Y el mecánico asignado recibe la notificación de inicio habilitado en su tablet

Escenario: Rechazo del presupuesto por parte del cliente
  Dado que el cliente decide no realizar la reparación
  Cuando la Recepcionista registra el rechazo con su motivo obligatorio
  Entonces la OT pasa al estado "rechazado"
  Y los repuestos previamente considerados quedan liberados sin reserva
  Y la orden permanece visible hasta que el vehículo sea retirado físicamente de la bahía
```

### Reglas de negocio y consideraciones técnicas asociadas

- **RN-07 / RN-08:** distinción estricta entre stock disponible, stock reservado y stock físico.
- **Backend:** endpoints `POST /api/v1/work-orders/:id/approve-quote` y `POST /api/v1/work-orders/:id/reject-quote`, ejecutados bajo una transacción atómica para consistencia entre auditoría, inventario y OT.
- **Frontend:** `QuoteApprovalModal.tsx` con captura de canal y notas, badge reactivo de estado (`[ APROBADO ]` / `[ RECHAZADO ]`) e invalidación de caché en el panel de bahías y stock.

### Desglose de tareas técnicas

**Backend (`modules/quotes`, `modules/inventory`, `modules/work-orders`):**

- **BE-T09.1:** crear `ApproveQuoteDto` y `RejectQuoteDto` con medio de aprobación, nombre, notas y motivo de rechazo.
- **BE-T09.2:** implementar los endpoints `approve-quote` y `reject-quote`.
- **BE-T09.3:** ejecutar en una transacción atómica el log auditable, cambio a `APROBADO` y marcado de repuestos como `RESERVED` (BE-16, BE-19, RN-07).

**Frontend (`features/quotes`, `features/work-orders`):**

- **FE-T09.1:** modal `QuoteApprovalModal.tsx` con selector de medio y campo de notas.
- **FE-T09.2:** mutación React Query que invalide OT, bahías e inventario (FE-09).
- **FE-T09.3:** badge `[ APROBADO ]` o `[ RECHAZADO ]` en el tablero principal.
