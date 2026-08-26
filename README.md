# Plaza de Comidas — Pragma PowerUp

Backend de una plataforma que centraliza los pedidos de una plazoleta de comidas con varios restaurantes. Reto desarrollado en **arquitectura hexagonal (puertos y adaptadores)**, repartido en 4 microservicios independientes, cada uno en su propio repositorio.

Este repositorio (`documentation`) es el punto de entrada del proyecto: no contiene código de negocio, solo la documentación transversal que amarra los otros 4.

## Microservicios

| Microservicio | Repositorio | Puerto | Base de datos | Responsabilidad |
|---|---|---|---|---|
| **Usuarios** | [users-microservice](https://github.com/Plaza-comidas-microservices/users-microservice) | `8081` | MySQL (`plazoleta_users`) | Autenticación, emisión de JWT, cuentas de Administrador/Propietario/Empleado/Cliente |
| **Plazoleta** | [mall-microservice](https://github.com/Plaza-comidas-microservices/mall-microservice) | `8082` | MySQL (`plazoleta_mall`) | Restaurantes, platos, ciclo de vida completo de los pedidos, reporte de eficiencia |
| **Mensajería** | [messaging-microservice](https://github.com/Plaza-comidas-microservices/messaging-microservice) | `8083` | — (sin persistencia) | Envío de SMS (Twilio) cuando un pedido queda listo |
| **Trazabilidad** | [traceability-microservice](https://github.com/Plaza-comidas-microservices/traceability-microservice) | `8084` | MongoDB (`traceability`) | Historial de cambios de estado de cada pedido, consulta de eficiencia por empleado |

Cada uno sigue la misma estructura interna (`domain` → `application` → `infrastructure`) y valida el JWT de forma independiente, sin depender de una librería compartida — así cada microservicio puede desplegarse y evolucionar por su cuenta.

## Arquitectura

`Navegador` → llama directamente a cada uno de los 4 microservicios. `mall-microservice` es el que más se comunica hacia los otros tres (valida propietarios/clientes contra `users`, notifica a `messaging`, registra y consulta trazabilidad contra `traceability`).

Ver `ARQUITECTURA.pdf` y `diagrma_flujo.jpeg` para el diagrama de contenedores y el flujo de estados de un pedido (`PENDIENTE → EN_PREPARACION → LISTO → ENTREGADO`, con `CANCELADO` como salida alterna desde `PENDIENTE`).

## Historias de usuario

| HU | Descripción | Repo(s) involucrados |
|---|---|---|
| HU1 | Crear propietario | users |
| HU2 | Crear restaurante | mall, users |
| HU3-4 | Crear / modificar plato | mall |
| HU5 | Autenticación (JWT) | users, mall |
| HU6 | Crear empleado | users |
| HU7 | Habilitar/deshabilitar plato | mall |
| HU8 | Crear cuenta cliente | users |
| HU9-10 | Listar restaurantes / platos | mall |
| HU11 | Realizar pedido | mall |
| HU12-13 | Listar pedidos / asignarse y cambiar estado | mall, users |
| HU14 | Notificar pedido listo (SMS + PIN) | mall, messaging, users |
| HU15 | Entregar pedido (validar PIN) | mall |
| HU16 | Cancelar pedido | mall |
| HU17 | Consultar trazabilidad del pedido | mall, traceability |
| HU18 | Consultar eficiencia de pedidos | mall, traceability |

## Cómo levantar todo el sistema localmente

1. **Bases de datos**: MySQL y MongoDB corriendo como servicios locales. Crear las bases `plazoleta_mall` y `plazoleta_users` en MySQL (MongoDB crea su base sola).
2. **Variables de entorno**: cada uno de los 4 repos tiene su propio `.env` (ignorado por git, no se commitea):
   - `mall-microservice/.env` → `DB_PASSWORD`, `JWT_SECRET`
   - `users-microservice/.env` → `DB_PASSWORD`, `JWT_SECRET`
   - `traceability-microservice/.env` → `JWT_SECRET`
   - `messaging-microservice/.env` → `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_PHONE_NUMBER` (y opcionalmente `SPRING_PROFILES_ACTIVE=log` para simular el envío de SMS sin usar Twilio)

   ⚠️ El `JWT_SECRET` debe ser **exactamente el mismo valor** en `mall`, `users` y `traceability` — es la llave compartida que valida los tokens entre microservicios.

3. **Compilar y testear** (por repo, con el `.env` cargado en la terminal):
   ```bash
   ./gradlew test
   ```
4. **Levantar** (orden sugerido, aunque ninguno depende de que otro esté arriba al arrancar):
   ```bash
   ./gradlew bootRun
   ```
   `users` → `mall` → `messaging` → `traceability`.

## Documentación y pruebas

- **Swagger** de cada microservicio: `http://localhost:<puerto>/swagger-ui/index.html`
- [`PlazaComidas.postman_collection.json`](./PlazaComidas.postman_collection.json) — colección de Postman con el flujo completo de punta a punta (login, creación de cuentas, restaurante, platos, pedido, asignación, entrega, cancelación, trazabilidad y eficiencia), con captura automática de tokens e ids entre requests.
- [`Estudio.html`](./Estudio.html) — guía de repaso de arquitectura, decisiones de diseño y patrones usados en el proyecto.

## Convenciones de trabajo

- Arquitectura hexagonal en los 4 repos: el dominio (`domain/`) nunca depende de tipos de framework.
- Una rama por HU (`feature/HUxx-descripcion-corta`), commits en formato `feat: descripción breve`.
- Tests unitarios (JUnit 5 + Mockito) por cada caso de uso, happy path + sad paths.
- Documentación de API con OpenAPI/Swagger en cada microservicio.
