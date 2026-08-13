# HU-02 - Visualizar órdenes de trabajo asignadas al mecánico

## Datos de la historia

- Actor: Mecánico
- Prioridad: Alta
- Puntos de esfuerzo: 3
- Alcance: Backend

## Historia de usuario

Como Mecánico, quiero visualizar las órdenes de trabajo que me han sido asignadas, para enfocarme en los trabajos que me corresponden.

## Criterios de aceptación

- Dado que un mecánico inicia sesión, cuando accede a su panel, entonces el sistema le muestra única y exclusivamente las OTs que le han sido asignadas de forma explícita.
- Dado que el mecánico visualiza los detalles de su OT asignada, cuando revisa la información de repuestos o mano de obra, entonces el sistema no debe mostrar ningún importe, precio de venta ni valor monetario.

## Reglas de negocio

- El mecánico solo puede consultar las OTs donde conste una asignación explícita a su usuario.
- No existe una asignación implícita por área, categoría o turno; toda OT debe tener un `mechanicId` explícito para ser visible.
- La asignación de la OT al mecánico se identifica por el ID del usuario autenticado, no por un ID enviado por el cliente.
- La respuesta del mecánico nunca debe incluir campos monetarios (precio de venta, importe, valor de repuestos o mano de obra), ni ahora ni cuando existan.
- La consulta debe ser de solo lectura; no debe modificar la OT ni su estado.
- Si el usuario autenticado no tiene el rol de mecánico, la consulta debe ser rechazada.

## Tareas de desarrollo

### Base de datos

- Revisar el modelo actual de `work_orders`; actualmente solo existe `receptionistId` y no hay asignación a mecánico.
- Agregar la columna `mechanic_id` (nullable) en `work_orders` como FK a la tabla de usuarios/mecánicos, mediante migración versionada.
- Evaluar si existe una tabla de usuarios; si no existe, definir el origen de datos del mecánico y documentar la FK correspondiente.
- Agregar índice para la consulta por `mechanic_id` y estado, optimizado para el listado del panel.
- Mantener `NOT NULL` para `receptionistId` y no alterar los campos existentes de la OT.
- No incluir columnas de precios en esta iteración; si existieran, verificar que la consulta del mecánico nunca las seleccione.
- Crear migración versionada y ejecutarla en un entorno de pruebas antes de integrarla.

### Repositories

- Definir el contrato `MechanicWorkOrdersRepository` con un método para listar OTs por ID de mecánico.
- Implementar la consulta que filtra por `mechanicId` (ID del usuario autenticado) y solo devuelve campos del detalle mecánico.
- Asegurar que la consulta seleccione columnas de forma explícita y nunca incluya columnas monetarias, incluso si se agregan en el futuro.
- Ordenar los resultados de forma estable y útil para el panel (por fecha de creación o prioridad definida).
- Mapear la ausencia de resultados como lista vacía y los errores de integridad a errores de aplicación.
- No aplicar reglas de negocio de autorización dentro del repositorio; esa responsabilidad pertenece al caso de uso.

### Services

- Crear el caso de uso `GetMechanicWorkOrdersService`.
- Recibir el ID del mecánico desde el usuario autenticado, nunca de un parámetro no confiable.
- Validar que el usuario autenticado tenga el rol de mecánico antes de consultar.
- Consultar únicamente las OTs asignadas explícitamente al ID del usuario autenticado.
- Proyectar los datos al DTO del mecánico, omitiendo intencionalmente campos de precio, importe y valor monetario.
- Mantener el caso de uso desacoplado de HTTP y PostgreSQL.
- Propagar errores de forma controlada usando excepciones de aplicación consistentes.

### Controllers

- Crear o extender el controlador del recurso de órdenes de trabajo.
- Exponer el endpoint de consulta del panel del mecánico.
- Proteger el endpoint con un guard que exija autenticación y rol `MECHANIC`.
- Adjuntar el ID del usuario autenticado al request mediante el guard y pasarlo al service.
- Retornar `200 OK` con el listado de OTs en el DTO del mecánico, sin campos monetarios.
- No incluir consultas SQL ni reglas de negocio en el controlador.

## Contrato del endpoint

### Listar OTs asignadas al mecánico autenticado

```http
GET /api/ordenes/mecanico/{id}
Authorization: Bearer <token>
```

Nota: el `{id}` del path debe coincidir con el usuario autenticado; el service debe resolver el mecánico a partir del request autenticado y no confiar únicamente en el path. Por convención del repositorio (recursos en inglés), el endpoint puede exponerse como `GET /api/work-orders/mechanic/{id}`.

Respuesta `200 OK`:

```json
{
  "workOrders": [
    {
      "id": "uuid",
      "plate": "ABC123",
      "brand": "Toyota",
      "model": "Corolla",
      "year": 2022,
      "customerName": "Nombre del cliente",
      "initialComplaint": "Ruido al frenar",
      "status": "OPEN",
      "createdAt": "2026-08-12T12:00:00.000Z"
    }
  ]
}
```

- La respuesta no incluye ningún campo de precio, importe, valor de repuestos ni mano de obra.
- Si el mecánico no tiene OTs asignadas, responde `200 OK` con `"workOrders": []`.

Respuesta `401 Unauthorized` si no hay usuario autenticado.

Respuesta `403 Forbidden` si el usuario autenticado no tiene rol de mecánico.

### Errores esperados

- `400 Bad Request`: parámetros de ruta inválidos.
- `401 Unauthorized`: usuario no autenticado.
- `403 Forbidden`: usuario autenticado sin rol de mecánico.
- `500 Internal Server Error`: error inesperado sin exponer detalles internos.
