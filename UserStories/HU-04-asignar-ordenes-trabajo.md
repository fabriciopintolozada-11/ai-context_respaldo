# HU-04 - Asignar órdenes de trabajo

## Datos de la historia

- Actor: Jefe de Taller
- Prioridad: Alta
- Puntos de esfuerzo: 3

## Historia de usuario

Como Jefe de Taller, quiero asignar una Orden de Trabajo a un mecánico, para distribuir los trabajos y organizar la carga del taller.

## Criterios de aceptación

- Dado que un usuario intenta asignar una OT a una bahía o mecánico, cuando no tiene el rol de Jefe de Taller, el sistema deniega la acción y oculta la opción.
- Dado que el Jefe de Taller selecciona una OT en estado inicial, cuando le asigna un mecánico, la OT cambia de estado y pasa a estar visible en el tablero personal de dicho mecánico.

## Tareas de desarrollo

### Frontend

- Desarrollar la vista de gestión para el Jefe de Taller.
- Integrar un selector o tablero drag-and-drop para vincular una OT con un mecánico.
- Ocultar la acción de asignación para usuarios que no tengan el rol de Jefe de Taller.
- Reflejar el cambio de estado y la asignación después de una operación exitosa.

### Backend

- Implementar middleware, guard o validación a nivel de controlador para restringir la asignación al rol `JEFE_TALLER`.
- Crear el endpoint de actualización que asigne el ID del mecánico a la OT.
- Cambiar el estado de la OT después de una asignación válida.
- Validar que la OT exista y se encuentre en un estado asignable.
- Validar que el mecánico exista y pueda recibir órdenes.
- Ejecutar el cambio de estado y la asignación dentro de una operación atómica.
- Garantizar que una OT asignada aparezca en la consulta personal del mecánico correspondiente.

## Contrato de endpoint

### Asignar una OT a un mecánico

```http
PATCH /api/ordenes/{id}
Authorization: Bearer <token>
Content-Type: application/json
```

Request:

```json
{
  "mecanicoId": "uuid"
}
```

El usuario autenticado debe tener el rol `JEFE_TALLER`.

Respuesta `200 OK`:

```json
{
  "id": "uuid",
  "mecanicoId": "uuid",
  "status": "ASIGNADA",
  "updatedAt": "2026-08-13T12:00:00.000Z"
}
```

### Errores esperados

- `400 Bad Request`: ID del mecánico ausente o inválido, o la OT no está en un estado asignable.
- `401 Unauthorized`: usuario no autenticado.
- `403 Forbidden`: usuario sin rol `JEFE_TALLER`.
- `404 Not Found`: OT o mecánico no encontrado.
- `409 Conflict`: OT ya asignada o conflicto de estado.
- `500 Internal Server Error`: error inesperado sin exponer detalles internos.
