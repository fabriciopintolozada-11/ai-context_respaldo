# Reglas de Desarrollo del Backend

Este documento define las reglas para desarrollar el backend del proyecto `pdis`. El backend debe implementarse con TypeScript sobre Node.js usando NestJS como framework, aplicando programacion orientada a objetos, arquitectura por capas y PostgreSQL como base de datos.

## 1. Framework y estructura NestJS

- Usar NestJS como framework oficial para la API y respetar su modelo de modulos, controladores y providers.
- Organizar la aplicacion por modulos funcionales o dominios; cada modulo debe declarar solo sus propios controladores, providers e importaciones.
- Usar decoradores de NestJS (`@Module`, `@Controller`, `@Injectable`) para declarar componentes.
- Resolver las dependencias mediante el sistema de inyeccion de NestJS. No instanciar manualmente servicios, repositorios o clientes de infraestructura.
- Mantener `main.ts` como punto de configuracion global de la aplicacion.
- Usar `ConfigModule` para la configuracion y validar las variables de entorno al iniciar la aplicacion.
- Usar guards, pipes, interceptors, filters y middleware de NestJS para las responsabilidades transversales correspondientes.
- No colocar reglas de negocio en controladores, guards o pipes; estas deben permanecer en el dominio o en los casos de uso.

## 2. Principios generales

- Usar TypeScript en todo el codigo de produccion. Evitar archivos `.js`, salvo configuraciones o scripts que lo requieran.
- Mantener `strict` habilitado en `tsconfig.json` y corregir los errores de tipos en lugar de ocultarlos.
- Preferir soluciones simples, cohesivas y faciles de probar.
- Aplicar SOLID, especialmente responsabilidad unica, inversion de dependencias y separacion de interfaces.
- No mezclar reglas de negocio con detalles de HTTP, ORM, SQL o infraestructura.
- No incluir secretos, credenciales ni cadenas de conexion en el repositorio.

## 3. Programacion orientada a objetos

- Modelar los conceptos principales del dominio mediante clases con responsabilidades claras.
- Encapsular el estado: usar propiedades privadas o protegidas y exponer solo operaciones necesarias.
- Preferir composicion sobre herencia cuando no exista una relacion de sustitucion real.
- Definir interfaces para contratos entre capas, especialmente repositorios y servicios externos.
- Las clases deben depender de abstracciones, no de implementaciones concretas.
- Evitar clases de utilidad estaticas con estado global y objetos que concentren responsabilidades no relacionadas.
- Los constructores deben recibir sus dependencias mediante inyeccion de dependencias.

## 4. Arquitectura por capas

La estructura puede organizarse por modulo o dominio, manteniendo estas responsabilidades:

```text
src/
├── domain/          # Entidades, value objects, reglas y contratos del dominio
├── application/     # Casos de uso, servicios de aplicacion y DTOs
├── infrastructure/  # PostgreSQL, repositorios concretos, configuracion y servicios externos
└── presentation/    # Rutas, controladores, middlewares y validacion HTTP
```

- **Dominio:** no depende de frameworks, HTTP, PostgreSQL ni ORM.
- **Aplicacion:** coordina casos de uso mediante interfaces del dominio; no debe ejecutar SQL ni acceder directamente a `req` o `res`.
- **Infraestructura:** implementa los contratos definidos por capas internas y contiene los detalles de persistencia.
- **Presentacion:** transforma solicitudes en entradas de casos de uso y respuestas de aplicacion en respuestas HTTP.
- Respetar el flujo de dependencias hacia el interior: `presentation -> application -> domain`; `infrastructure` implementa contratos internos.
- Un controlador no debe contener reglas de negocio ni consultas a la base de datos.
- Un repositorio no debe decidir reglas propias del caso de uso.

## 5. Convenciones de codigo

- Usar nombres descriptivos en ingles para clases, interfaces, funciones, variables y archivos, salvo que el dominio requiera terminos en espanol.
- Usar `PascalCase` para clases, tipos e interfaces; `camelCase` para variables, funciones y metodos; `UPPER_SNAKE_CASE` solo para constantes globales.
- Nombrar interfaces por su responsabilidad (`UserRepository`) y no con prefijos innecesarios como `IUserRepository`.
- Mantener funciones y metodos pequenos, con una sola responsabilidad.
- Usar tipos explicitos en contratos publicos y evitar `any`. Para datos desconocidos usar `unknown` y validarlos.
- Usar `async/await` para operaciones asincronas y propagar errores de forma controlada.
- Centralizar la configuracion y validar las variables de entorno al iniciar la aplicacion.
- Formatear y verificar el codigo con herramientas automatizadas de linting y formato configuradas en el proyecto.

## 6. API y errores

- Versionar los endpoints cuando sea necesario y mantener una convencion consistente de rutas y respuestas.
- Validar y normalizar toda entrada externa antes de entregarla a un caso de uso.
- Usar DTOs para transportar datos entre la capa de presentacion y la aplicacion; no exponer entidades directamente.
- Usar codigos HTTP adecuados y respuestas de error con una estructura uniforme.
- Definir errores de dominio y de aplicacion para distinguir fallos esperados de errores inesperados.
- No devolver stack traces, credenciales, consultas SQL ni informacion sensible al cliente.
- Implementar un middleware centralizado para registrar y transformar errores no controlados.

## 7. PostgreSQL y persistencia

- Usar PostgreSQL como unica fuente persistente de datos, salvo decision documentada.
- Gestionar el esquema mediante migraciones versionadas y revisables; no modificar manualmente bases compartidas.
- Usar consultas parametrizadas o el mecanismo seguro del ORM para evitar inyeccion SQL.
- Mantener la logica de acceso a datos en repositorios o adaptadores de infraestructura.
- Usar transacciones para operaciones que deban ser atomicas y definir explicitamente sus limites.
- Definir claves primarias, foraneas, restricciones `NOT NULL`, `UNIQUE` y `CHECK` cuando correspondan.
- Crear indices basados en consultas reales y revisar el impacto de cambios de esquema.
- Usar tipos compatibles con PostgreSQL y mapearlos explicitamente a tipos de dominio.
- Configurar un pool de conexiones, liberar recursos correctamente y no abrir conexiones por solicitud.
- No registrar valores sensibles ni datos personales innecesarios en consultas o logs.

## 8. Seguridad

- Validar autenticacion y autorizacion en cada caso de uso que lo requiera; no confiar solo en el frontend.
- Aplicar minimo privilegio a usuarios, roles y conexiones de PostgreSQL.
- Mantener dependencias actualizadas y revisar vulnerabilidades antes de incorporar paquetes.
- Configurar CORS, limites de solicitudes, encabezados de seguridad y politicas de acceso segun el entorno.
- Sanitizar datos que se registren y evitar exponer tokens, contrasenas o informacion personal.

## 9. Pruebas y calidad

- Cada caso de uso debe tener pruebas unitarias de sus reglas principales y escenarios de error.
- Probar repositorios con una base PostgreSQL de pruebas o un entorno aislado equivalente; no depender de datos de desarrollo.
- Cubrir los endpoints criticos con pruebas de integracion.
- No aceptar cambios que fallen en compilacion, linting o pruebas automatizadas.
- Toda nueva funcionalidad debe incluir actualizacion de migraciones, validaciones y documentacion correspondiente.

## 10. Flujo para cambios

1. Revisar este documento y la documentacion relevante de `ai-context` antes de modificar el backend.
2. Identificar la capa y el modulo afectados antes de escribir codigo.
3. Definir o actualizar contratos e interfaces antes de implementar adaptadores concretos.
4. Implementar el caso de uso y sus reglas sin acoplarlo a HTTP o PostgreSQL.
5. Agregar validaciones, manejo de errores y pruebas.
6. Ejecutar compilacion, linting y pruebas antes de considerar terminado el cambio.
7. Actualizar este documento si cambia una decision arquitectonica o una convencion del proyecto.
