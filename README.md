# SmartLogistics Practice

![Java 21](https://img.shields.io/badge/Java-21-%23ED8B00?logo=openjdk&logoColor=white)
![Spring Boot 3.3](https://img.shields.io/badge/Spring_Boot-3.3-%236DB33F?logo=springboot&logoColor=white)
![Next.js 14](https://img.shields.io/badge/Next.js-14-%23000000?logo=nextdotjs&logoColor=white)
![PostgreSQL 16](https://img.shields.io/badge/PostgreSQL-16-%234169E1?logo=postgresql&logoColor=white)
![Redis 7](https://img.shields.io/badge/Redis-7-%23DC382D?logo=redis&logoColor=white)
![MongoDB 7](https://img.shields.io/badge/MongoDB-7-%2347A248?logo=mongodb&logoColor=white)
![RabbitMQ 3.13](https://img.shields.io/badge/RabbitMQ-3.13-%23FF6600?logo=rabbitmq&logoColor=white)
![Docker Compose](https://img.shields.io/badge/Docker-Compose-%232496ED?logo=docker&logoColor=white)
![Unreal Engine 5](https://img.shields.io/badge/Unreal_Engine-5-%230E1128?logo=unrealengine&logoColor=white)

Plataforma distribuida para coordinar operaciones de almacenes autónomos con robots de picking. El sistema combina gestión de órdenes, inventario, planificación de rutas, telemetría en tiempo real, analítica logística y una simulación 3D conectada al backend.

Este repositorio funciona como un laboratorio práctico para explorar arquitectura de microservicios, comunicación síncrona y asíncrona, integración de múltiples motores de persistencia, observabilidad y desarrollo incremental con Gitflow.

## Índice

- [Problema](#problema)
- [Solución](#solución)
- [Arquitectura](#arquitectura)
- [Flujos principales](#flujos-principales)
- [Servicios](#servicios)
- [Tecnologías](#tecnologías)
- [Inicio rápido](#inicio-rápido)
- [Interfaces locales](#interfaces-locales)
- [Desarrollo](#desarrollo)
- [Observabilidad](#observabilidad)
- [Simulación 3D](#simulación-3d)
- [Decisiones técnicas](#decisiones-técnicas)
- [Seguridad](#seguridad)
- [Estructura](#estructura)
- [Licencia](#licencia)

## Problema

Un centro de distribución con robots autónomos necesita coordinar órdenes y movimientos en un entorno con alta concurrencia. Sin una plataforma centralizada aparecen problemas como:

- Asignación de órdenes a robots con batería insuficiente.
- Colisiones o rutas que atraviesan zonas ocupadas del almacén.
- Tiempos muertos por rutas subóptimas.
- Inventario desactualizado durante operaciones simultáneas.
- Falta de trazabilidad sobre rutas, eventos y rendimiento.
- Interfaces que dependen de refrescos manuales para conocer el estado de la operación.

### Reglas operativas

- Un robot no puede recibir una orden si su batería está por debajo del umbral configurado.
- Los productos frágiles se transportan con una política de velocidad reducida.
- Las rutas completadas generan eventos para alimentar analítica de congestión.
- Las operaciones críticas se validan de forma síncrona; los procesos secundarios se desacoplan mediante eventos.

## Solución

SmartLogistics separa las responsabilidades en servicios especializados:

1. `ms-identity` gestiona usuarios, contraseñas y tokens JWT.
2. `ms-warehouse-core` coordina órdenes, inventario, layouts y rutas.
3. `ms-robot-status` mantiene el estado actual de la flota y expone telemetría.
4. `ms-logistics-analytics` procesa eventos de rutas y genera datos de congestión.
5. El dashboard web permite operar el almacén desde una interfaz única.
6. La simulación Unreal representa visualmente el almacén y sus robots.

Las decisiones inmediatas utilizan REST. Los cambios de estado y procesos que pueden ejecutarse de forma desacoplada utilizan RabbitMQ. El frontend recibe actualizaciones en vivo mediante Server-Sent Events.

## Arquitectura

```text
                                      +----------------------+
                                      |  Next.js Dashboard   |
                                      |      :3000           |
                                      +----------+-----------+
                                                 |
                                                 | REST / SSE
                                                 v
                                      +----------------------+
                                      |   Nginx API Gateway   |
                                      |        :8086          |
                                      | JWT validation / CORS |
                                      +----+------+------+-----+
                                           |      |      |
                              REST         |      |      | SSE
                                           v      v      v
                                  +---------+  +---+---+  +---------+
                                  | Identity|  |Warehouse| | Robot  |
                                  |  :8084  |  | Core :8081| |Status |
                                  +----+----+  +---+---+  | :8082 |
                                       |           |       +----+----+
                                       v           v            |
                                  +---------+  +---+---+        |
                                  | Auth DB |  |Warehouse|       |
                                  |Postgres |  | DB      |       |
                                  +---------+  |Postgres |       |
                                               +---+---+        |
                                                   |            v
                                                   |      +-----+------+
                                                   +----->| RabbitMQ   |
                                                          | AMQP/STOMP |
                                                          +--+------+--+
                                                             |      |
                                                             v      v
                                                    +--------+  +---+--------+
                                                    |Analytics|  | Unreal     |
                                                    | :8083   |  | Simulation |
                                                    +----+----+  +------------+
                                                         |
                                                         v
                                                    +---------+
                                                    | MongoDB |
                                                    +---------+

                                  Robot state and fast telemetry: Redis
```

Los diagramas editables y sus exportaciones están disponibles en [`docs/`](docs/):

- [Contexto del sistema](docs/c4-system-context.drawio)
- [Diagrama de contenedores](docs/c4-container-diagram.drawio)

## Flujos principales

### Creación y asignación de una orden

1. El usuario crea una orden desde el dashboard.
2. Nginx valida el token JWT y enruta la solicitud.
3. Warehouse Core valida el inventario disponible.
4. Warehouse Core consulta síncronamente el estado del robot.
5. Se valida la batería y las reglas de asignación.
6. Se calcula una ruta evitando obstáculos y celdas no transitables.
7. La operación se persiste junto con su evento Outbox.
8. El evento se publica en RabbitMQ para que Robot Status despache la misión.

### Telemetría en tiempo real

1. La simulación Unreal publica posición, batería y estado del robot.
2. Robot Status actualiza Redis y publica eventos de estado.
3. El adaptador SSE envía los cambios al dashboard.
4. El usuario visualiza la flota sin refrescar la página.

### Analítica de rutas

1. Warehouse Core publica `route.completed`.
2. Analytics consume el evento desde RabbitMQ.
3. Se persisten la ruta y sus puntos en MongoDB.
4. Los datos quedan disponibles para métricas y mapas de congestión.

## Servicios

| Servicio | Puerto | Persistencia | Responsabilidad |
| --- | ---: | --- | --- |
| `ms-identity` | `8084` | PostgreSQL | Registro, login, hashing y validación JWT |
| `ms-warehouse-core` | `8081` | PostgreSQL + Flyway | Órdenes, inventario, layouts, rutas y Outbox |
| `ms-robot-status` | `8082` | Redis | Estado de robots, comandos y telemetría SSE |
| `ms-logistics-analytics` | `8083` | MongoDB | Eventos de rutas y muestras de congestión |
| Nginx gateway | `8086` | - | Enrutamiento, CORS y frontera de autenticación |
| Next.js dashboard | `3000` | - | Operación visual del almacén |

## Tecnologías

### Backend

- Java 21.
- Spring Boot 3.3.
- Spring Web, Data JPA, WebFlux, AMQP y Actuator.
- Arquitectura hexagonal con separación de dominio, aplicación e infraestructura.
- Dijkstra y planificación basada en una cuadrícula del almacén.
- Flyway para migraciones PostgreSQL.
- JWT y BCrypt para autenticación.

### Frontend

- Next.js 14 con App Router.
- TypeScript.
- Tailwind CSS.
- Server-Sent Events para información operacional en tiempo real.

### Infraestructura

- Docker Compose.
- Nginx 1.27 como API Gateway.
- RabbitMQ 3.13 con AMQP y Web STOMP.
- PostgreSQL 16, Redis 7 y MongoDB 7.

### Observabilidad

- Prometheus y Micrometer para métricas.
- Grafana con dashboards preconfigurados.
- Loki y Promtail para logs.
- OpenTelemetry Collector y Jaeger para trazas distribuidas.

### Simulación

- Unreal Engine 5.7.
- Cliente HTTP para integración operacional.
- Cliente RabbitMQ Web STOMP para comandos y eventos en tiempo real.

## Inicio rápido

### Requisitos

- Docker Desktop con integración WSL2 o Docker Engine.
- Docker Compose.
- Node.js 20 o superior.
- Java 21 o superior para desarrollo local.
- Unreal Engine 5.7 para abrir la simulación.

### Configurar el backend

```bash
cd smartlogistic
cp .env.example .env
```

Reemplaza los valores `change-this-*` antes de iniciar el entorno.

### Iniciar infraestructura y servicios

```bash
docker compose up --build -d
docker compose ps
```

El primer arranque puede tardar varios minutos porque compila los servicios Java y descarga las imágenes de infraestructura.

### Iniciar el dashboard

```bash
cd smartlogistic-web
npm ci
npm run dev
```

El dashboard se ejecuta en `http://localhost:3000` y utiliza el gateway disponible en `http://localhost:8086`.

### Detener el entorno

```bash
cd smartlogistic
docker compose down
```

Para eliminar también los volúmenes locales:

```bash
docker compose down -v
```

## Interfaces locales

| Interfaz | URL | Uso |
| --- | --- | --- |
| Dashboard | [localhost:3000](http://localhost:3000) | Gestión del almacén |
| API Gateway | [localhost:8086](http://localhost:8086) | API unificada |
| Grafana | [localhost:3001](http://localhost:3001) | Métricas, logs y dashboards |
| Jaeger | [localhost:16686](http://localhost:16686) | Trazas distribuidas |
| RabbitMQ | [localhost:15672](http://localhost:15672) | Administración del broker |
| Prometheus | [localhost:9090](http://localhost:9090) | Consultas de métricas |

Las credenciales se definen únicamente en el archivo local `.env`, que no debe versionarse.

## Desarrollo

El flujo del repositorio utiliza `dev` como rama de integración y `main` para versiones estables:

```text
feature/<capability>  ->  PR  ->  dev  ->  release PR  ->  main
```

Reglas recomendadas:

- Crear cada cambio desde `dev`.
- Mantener los PRs enfocados en una capacidad concreta.
- Ejecutar las pruebas antes de solicitar revisión.
- Promover incrementos funcionales de `dev` a `main` sin esperar a completar todo el roadmap.
- Usar commits descriptivos con prefijos `feat`, `fix`, `docs`, `chore` o `test`.

### Ejecutar pruebas backend

```bash
cd smartlogistic/ms-warehouse-core
./mvnw test

cd ../ms-identity
./mvnw.cmd test
```

Los demás servicios siguen el mismo ciclo Maven desde sus respectivos directorios.

### Validar el dashboard

```bash
cd smartlogistic-web
npm run build
```

## Observabilidad

El stack incluye dashboards preconfigurados para:

- Estado general de la plataforma.
- Órdenes, rutas e inventario.
- Estado, batería y telemetría de robots.
- Eventos procesados y congestión.
- Autenticación y actividad del servicio de identidad.

Las trazas permiten seguir una operación desde la solicitud inicial del dashboard hasta la validación del robot, la planificación de ruta y la publicación del evento.

## Simulación 3D

El proyecto Unreal está en [`Simulation/`](Simulation/). Para abrirlo:

1. Instala Unreal Engine 5.7.
2. Abre `Simulation/Simulation.uproject`.
3. Inicia el backend y RabbitMQ.
4. Configura el host del gateway y del broker según el entorno local.
5. Ejecuta la simulación para recibir misiones y publicar telemetría.

La simulación construye el entorno del almacén, recibe comandos de robots mediante Web STOMP y utiliza el fallback HTTP para consultar misiones pendientes cuando es necesario.

## Decisiones técnicas

### REST para decisiones inmediatas

La disponibilidad de un robot y la validación de inventario necesitan una respuesta antes de aceptar una operación. Por eso Warehouse Core consulta Robot Status mediante REST.

### RabbitMQ para desacoplamiento

Las rutas completadas, comandos y actualizaciones de estado se distribuyen como eventos. Esto permite que analítica y visualización evolucionen sin bloquear el flujo operacional.

### Outbox para eventos confiables

Warehouse Core registra el evento en la misma unidad de trabajo que la operación de negocio. Un proceso posterior publica los eventos pendientes, reduciendo el riesgo de confirmar una operación sin notificarla.

### Persistencia poliglota

- PostgreSQL se utiliza para datos transaccionales.
- Redis se utiliza para lecturas rápidas del estado actual de robots.
- MongoDB se utiliza para documentos de rutas y muestras de analítica.

### Arquitectura hexagonal

El dominio no depende directamente de Spring, PostgreSQL, Redis, RabbitMQ ni HTTP. Los adaptadores de infraestructura implementan puertos definidos por la aplicación, facilitando pruebas y sustitución de componentes.

## Seguridad

La configuración incluida está orientada a desarrollo local. Antes de desplegar públicamente:

- Cambia todas las credenciales del archivo `.env`.
- Genera un secreto JWT largo y aleatorio.
- Restringe CORS a dominios concretos.
- Protege los endpoints SSE y de misiones.
- No expongas bases de datos ni consolas administrativas a Internet.
- Usa imágenes Docker fijadas a versiones revisadas.
- Revisa logs para evitar tokens o credenciales en URLs.

## Estructura

```text
smartlogistics-practice/
├── smartlogistic/
│   ├── docker-compose.yml
│   ├── nginx/
│   ├── rabbitmq/
│   ├── observability/
│   ├── scripts/
│   ├── ms-identity/
│   ├── ms-warehouse-core/
│   ├── ms-robot-status/
│   └── ms-logistics-analytics/
├── smartlogistic-web/
├── Simulation/
├── docs/
├── .env.example
├── .gitignore
└── LICENSE
```

## Licencia

Este repositorio se distribuye bajo la licencia MIT. Consulta [LICENSE](LICENSE).
