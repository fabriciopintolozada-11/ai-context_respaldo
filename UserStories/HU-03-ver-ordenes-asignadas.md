# HU-03 - Ver órdenes asignadas

## Datos de la historia

- Actor: Mecánico
- Prioridad: Alta (crítica para el Sprint 1)
- Puntos de esfuerzo: 3

## Historia de usuario

Como Mecánico, quiero visualizar las órdenes de trabajo que me han sido asignadas, para enfocarme en los trabajos que me corresponden.

## Criterios de aceptación

- Dado que un mecánico inicia sesión, cuando accede a su panel, el sistema muestra única y exclusivamente las OTs que le han sido asignadas de forma explícita.
- Dado que el mecánico visualiza los detalles de una OT asignada, cuando revisa la información de repuestos o mano de obra, el sistema no muestra importes, precios de venta ni valores monetarios.

## Tareas de desarrollo

### Frontend

- Diseñar una interfaz limpia y responsiva, optimizada para la tablet del taller.
- Listar las tareas pendientes del mecánico autenticado.
- Mostrar el detalle operativo de una OT sin información monetaria.
- Ocultar las opciones y acciones que no correspondan al rol de mecánico.

### Backend

- Crear un endpoint de consulta que filtre las órdenes por el ID del usuario autenticado.
- Crear un DTO específico para el mecánico que omita intencionalmente los campos de precios.
- Garantizar que los campos de precios, importes y valores monetarios no viajen en la respuesta de la API.
- Validar que el mecánico solo pueda consultar órdenes asignadas explícitamente a su usuario.
- Aplicar autorización para impedir el acceso de usuarios no autenticados o con roles no permitidos.

## Contrato de endpoint

### Consultar órdenes asignadas al mecánico

```http
GET /api/ordenes/mecanico/{id}
Authorization: Bearer <token>
```

El backend debe validar que `{id}` corresponda al usuario autenticado o rechazar la solicitud.

Respuesta `200 OK`:

```json
[
  {
    "id": "uuid",
    "vehicleId": "uuid",
    "plate": "ABC123",
    "status": "ASIGNADA",
    "initialComplaint": "Ruido al frenar",
    "assignedAt": "2026-08-13T12:00:00.000Z"
  }
]
```

La respuesta no debe incluir precios, importes, costos ni ningún otro valor monetario.

### Errores esperados

- `401 Unauthorized`: usuario no autenticado.
- `403 Forbidden`: usuario sin rol de mecánico o intentando consultar órdenes de otro usuario.
- `404 Not Found`: mecánico no encontrado, si la implementación consulta el recurso explícitamente.
- `500 Internal Server Error`: error inesperado sin exponer detalles internos.
