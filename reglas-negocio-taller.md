# Reglas de negocio del taller

Estas reglas son el contexto funcional obligatorio del sistema. Las implementaciones nuevas deben respetarlas y no deben introducir estados, cobros o permisos que las contradigan.

- **RN-01:** Todo vehículo que ingrese debe tener una OT registrada antes de iniciar actividades, con datos obligatorios del vehículo y reclamo inicial.
- **RN-02:** No puede iniciar reparación ni consumirse insumos sin aprobación explícita del cliente sobre el presupuesto.
- **RN-03:** Trabajos o fallas adicionales suspenden la OT y la devuelven a presupuesto hasta una nueva aprobación.
- **RN-04:** Un mecánico solo visualiza y trabaja OT asignadas explícitamente.
- **RN-05:** Falta de repuestos cambia la OT a `En Espera de Repuesto` y libera al mecánico.
- **RN-06:** Tras 15 días sin respuesta de aprobación se alerta a Recepción.
- **RN-07:** Los repuestos de un presupuesto aprobado quedan reservados exclusivamente para esa OT.
- **RN-08:** El inventario se descuenta solo al confirmar instalación y uso efectivo.
- **RN-09:** No hay salida de almacén sin OT aprobada y mecánico solicitante vinculado.
- **RN-10:** Se alertan repuestos sin rotación durante al menos 2 meses.
- **RN-11:** Los servicios tienen garantía predeterminada de 30 días desde la entrega.
- **RN-12:** Un reingreso relacionado dentro de 30 días crea una OT de garantía sin cobro.
- **RN-13:** Después de 30 días, una exención requiere autorización del Jefe de Taller.
- **RN-14:** Solo el Jefe de Taller asigna OT y define prioridades de bahías.
- **RN-15:** Solo el Jefe de Taller aplica descuentos, anulaciones o modifica montos liquidados.
- **RN-16:** Los mecánicos no visualizan precios ni modifican importes.
- **RN-17:** El cliente consulta estado con placa y documento, sin cuenta.
- **RN-18:** Se bloquea el registro de vehículos 100% eléctricos.
- **RN-19:** El historial técnico es permanente y no se elimina.
- **RN-20:** Al ingresar una placa se muestra automáticamente el historial previo.
- **RN-21:** Todas las transacciones se gestionan en Bolivianos (BOB).
- **RN-22:** El sistema genera Cuenta de Taller, no factura fiscal tributaria.
- **RN-23:** Estados permitidos: `Recibido`, `En Diagnóstico`, `Presupuesto Enviado`, `Aprobado`, `En Reparación`, `Listo`, `Entregado`, `Rechazado` y `En Espera de Repuesto`.
- **RN-24:** No se puede pasar directamente de diagnóstico a reparación sin presupuesto y aprobación.
- **RN-25:** El Jefe de Taller valida el trabajo antes de `Listo`.
- **RN-26:** Rechazar presupuesto cambia a `Rechazado` y cobra el diagnóstico.
- **RN-27:** Con cuatro bahías ocupadas se registra el vehículo, pero queda en espera de asignación.
- **RN-28:** Los ajustes de inventario requieren motivo, fecha y usuario responsable.
- **RN-29:** El cliente solo consulta estado y presupuesto, no horas ni repuestos en tiempo real.
- **RN-30:** `Entregado` requiere trabajo validado, cuenta preparada, pago y devolución de llaves por Recepción.

## Alcance actual: HU01

HU01 implementa recepción del vehículo y creación de la OT. En este alcance aplican directamente RN-01, RN-18, RN-20, RN-23 y RN-27. Las restantes reglas se implementarán junto con las funcionalidades que las soportan.
