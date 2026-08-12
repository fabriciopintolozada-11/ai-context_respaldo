# HU-01 - Registrar ingreso del vehículo

## Datos de la historia

- Actor: Recepcionista
- Prioridad: Alta
- Puntos de esfuerzo: 3

## Historia de usuario

Como Recepcionista, quiero registrar el ingreso de un vehículo capturando sus datos y el reclamo del cliente, para iniciar formalmente el flujo de atención y generar la Orden de Trabajo.

## Criterios de aceptación

- Al ingresar los datos obligatorios y el reclamo inicial, el sistema crea una nueva Orden de Trabajo (OT).
- Al digitar una placa existente, el sistema despliega el expediente e historial técnico del vehículo.
- Si el vehículo es 100% eléctrico, el sistema bloquea la recepción y muestra un mensaje de restricción.

## Reglas de negocio

- La placa es obligatoria, debe normalizarse y conservar un formato válido.
- El vehículo puede ser nuevo o existir previamente.
- La OT debe quedar asociada al vehículo, cliente, recepcionista y reclamo inicial.
- Un vehículo 100% eléctrico no puede generar una OT por este flujo.
- La consulta de expediente e historial no debe modificar información.
- La creación de la OT y sus relaciones debe ser atómica.
- La OT debe iniciar con estado `OPEN` o el estado equivalente definido por el dominio.

## Tareas de desarrollo

### Base de datos

- Revisar o definir las tablas `customers`, `vehicles`, `work_orders` y el historial técnico relacionado.
- Definir la unicidad de la placa normalizada.
- Definir el atributo que identifica si el vehículo es `100% eléctrico`.
- Relacionar la OT con vehículo, cliente y usuario recepcionista.
- Persistir reclamo inicial, estado y fechas de la OT.
- Agregar índices para búsqueda por placa y consulta de historial.
- Agregar restricciones de integridad y campos obligatorios.
- Crear migraciones versionadas, si las tablas aún no existen.

### Repositories

- Definir el contrato para buscar un vehículo por placa normalizada.
- Definir el contrato para obtener el expediente del vehículo y su historial técnico.
- Definir el contrato para crear una OT con sus relaciones.
- Definir consultas separadas para lectura del expediente y persistencia de la OT.
- Implementar transacción para crear cliente/vehículo cuando corresponda y registrar la OT.
- Mapear ausencia de vehículo, cliente inválido y errores de integridad a errores de aplicación.

### Services

- Crear el caso de uso `RegisterVehicleEntryService`.
- Normalizar y validar la placa y los datos obligatorios.
- Buscar el vehículo por placa antes de registrar la OT.
- Retornar expediente e historial cuando exista un registro previo.
- Evaluar la restricción de vehículo 100% eléctrico antes de iniciar la transacción.
- Crear o reutilizar el cliente y vehículo según las reglas del dominio.
- Crear la OT con el reclamo inicial y el usuario autenticado.
- Garantizar que una recepción bloqueada no genere cambios parciales.

### Controllers

- Crear el controlador del recurso `/work-orders`.
- Exponer un endpoint de consulta previa por placa para expediente e historial.
- Exponer un endpoint para registrar el ingreso y crear la OT.
- Recibir DTOs validados y el usuario autenticado.
- Delegar validaciones de negocio al service.
- Retornar `201 Created` para una OT creada.
- Retornar `403 Forbidden` o `409 Conflict` para la restricción de vehículo eléctrico, según la convención global de errores.
- No incluir consultas SQL ni reglas de negocio en el controlador.

## Contrato de endpoints

### Consultar expediente por placa

```http
GET /api/vehicles/{plate}/history
Authorization: Bearer <token>
```

Respuesta `200 OK`:

```json
{
  "vehicle": {
    "id": "uuid",
    "plate": "ABC123",
    "isFullyElectric": false
  },
  "customer": {
    "id": "uuid",
    "name": "Nombre del cliente"
  },
  "history": []
}
```

Respuesta `404 Not Found` si no existe un vehículo con esa placa.

### Registrar ingreso y crear OT

```http
POST /api/work-orders
Authorization: Bearer <token>
Content-Type: application/json
```

Request:

```json
{
  "plate": "ABC123",
  "customer": {
    "identification": "123456789",
    "name": "Nombre del cliente",
    "phone": "+57 3000000000"
  },
  "vehicle": {
    "brand": "Toyota",
    "model": "Corolla",
    "year": 2022,
    "isFullyElectric": false
  },
  "initialComplaint": "Ruido al frenar"
}
```

Respuesta `201 Created`:

```json
{
  "id": "uuid",
  "vehicleId": "uuid",
  "customerId": "uuid",
  "status": "OPEN",
  "initialComplaint": "Ruido al frenar",
  "createdAt": "2026-08-12T12:00:00.000Z"
}
```

### Errores esperados

- `400 Bad Request`: campos obligatorios ausentes o formato inválido.
- `401 Unauthorized`: usuario no autenticado.
- `403 Forbidden`: usuario sin rol de recepcionista o vehículo 100% eléctrico bloqueado.
- `404 Not Found`: vehículo o recurso relacionado no encontrado durante una consulta.
- `409 Conflict`: placa duplicada, recepción no permitida o conflicto de estado.
- `500 Internal Server Error`: error inesperado sin exponer detalles internos.
