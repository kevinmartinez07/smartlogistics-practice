# SmartLogistics — Gestión de Almacenes Autónomos

![Java 21](https://img.shields.io/badge/Java-21-%23ED8B00?logo=openjdk)
![Spring Boot 3.3](https://img.shields.io/badge/Spring_Boot-3.3-%236DB33F?logo=springboot)
![Next.js 14](https://img.shields.io/badge/Next.js-14-%23000000?logo=nextdotjs)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-%234169E1?logo=postgresql)
![Redis](https://img.shields.io/badge/Redis-7-%23DC382D?logo=redis)
![MongoDB](https://img.shields.io/badge/MongoDB-7-%2347A248?logo=mongodb)
![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3.13-%23FF6600?logo=rabbitmq)
![Docker](https://img.shields.io/badge/Docker-Compose-%232496ED?logo=docker)
![Unreal Engine 5](https://img.shields.io/badge/Unreal_Engine-5-%230E1128?logo=unrealengine)

Sistema de gestión logística para centros de distribución masivos que utilizan **robots autónomos** para el picking de productos. Coordina órdenes de despacho en tiempo real, valida disponibilidad y estado de carga de robots, previene colisiones en pasillos de alta densidad y registra cada movimiento para análisis de eficiencia.

El proyecto integra una simulación 3D en Unreal Engine, un dashboard web para la operación del almacén y una plataforma backend distribuida con observabilidad completa.

---

## Problema

Un centro de distribución masivo utiliza robots autónomos para el picking de productos. Sin coordinación en tiempo real se presentan:

- **Colisiones** entre robots en pasillos de alta densidad.
- **Tiempos muertos** por rutas subóptimas.
- **Falta de trazabilidad** sobre el rendimiento logístico.
- **Asignación ineficiente** de órdenes sin considerar el estado real de los robots.
- **Inventario desactualizado** durante operaciones simultáneas.

### Reglas de Negocio

- No se puede asignar una orden de despacho a un robot si su nivel de batería es inferior al umbral configurado.
- Los productos marcados como **frágiles** deben ser transportados a velocidad reducida.
- Cada ruta completada debe enviarse asíncronamente al servicio de analítica.
- La planificación de rutas debe evitar celdas bloqueadas y ubicaciones de estanterías.

## Solución

Arquitectura de **microservicios** con comunicación **síncrona (REST)** para operaciones que requieren respuesta inmediata y **asíncrona (RabbitMQ)** para procesos desacoplados. Los servicios siguen el patrón de **Arquitectura Hexagonal**, separando dominio, aplicación e infraestructura.

Incluye:

- Simulación 3D en Unreal Engine 5.7 para visualizar la flota en tiempo real.
- Dashboard web en Next.js para gestionar el almacén.
- Comunicación de eventos mediante RabbitMQ, AMQP y Web STOMP.
- Streaming de telemetría mediante Server-Sent Events.
- Stack de observabilidad con Prometheus, Grafana, Loki, OpenTelemetry y Jaeger.

---

## Arquitectura

### Diagrama de Contenedores (C4 Nivel 2)

```text
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                   NGINX API Gateway (:8086)                          │
│                        JWT Validation · CORS · Routing · SSE Proxy                   │
└──┬──────────┬──────────────────┬──────────────────┬──────────────────┬──────────────┘
   │          │                  │                  │                  │
   ▼          ▼                  ▼                  ▼                  ▼
┌──────┐ ┌──────────┐ ┌──────────────┐ ┌────────────────┐ ┌──────────────────┐
│ Web  │ │ Identity │ │ Warehouse    │ │ Robot Status   │ │ Logistics         │
│ UI   │ │ :8084    │ │ Core :8081  │ │ :8082         │ │ Analytics :8083   │
│Next  │ │ REST     │ │ REST + AMQP │ │ REST + AMQP   │ │ AMQP Consumer     │
└──┬───┘ └────┬─────┘ └──────┬───────┘ └───────┬────────┘ └────────┬─────────┘
   │          │              │                 │                   │
   │          ▼              ▼                 ▼                   ▼
   │   ┌──────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
   │   │PostgreSQL│ │ PostgreSQL   │ │    Redis     │ │    MongoDB       │
   │   │ auth_db  │ │ warehouse_db │ │ (estado de   │ │ (rutas, muestras │
   │   │ (:5433)  │ │ (:5432)      │ │  robots)     │ │  de congestión)  │
   │   └──────────┘ └──────────────┘ └──────────────┘ └──────────────────┘
   │                         │              │                   │
   │                         ▼              ▼                   │
   │              ┌──────────────────────────────────────────────┘
   │              │             RABBITMQ (:5672)                  │
   │              │       Exchange: logistics.exchange           │
   │              │                                               │
   │              │   order.dispatched                            │
   │              │   package.dispatched                          │
   │              │   route.completed  ───────────────► Analytics  │
   │              │   robot.telemetry  ◄────────────── Sim/Status  │
   │              │   robot.command    ───────────────► Simulation │
   │              └──────────────────────────────┬────────────────┘
   │                                             │
   │                                             ▼
   │                                  ┌────────────────────┐
   └─────────────────────────────────►│  Unreal Engine     │
                                      │  Simulation        │
                                      │  STOMP/WS + HTTP  │
                                      └────────────────────┘
```

Los diagramas editables y sus exportaciones están disponibles en [`docs/`](docs/):

- [Contexto del sistema](docs/c4-system-context.drawio)
- [Contenedores del sistema](docs/c4-container-diagram.drawio)

### Flujo de Comunicación

| Tipo | Protocolo | Origen → Destino | Propósito |
|------|-----------|------------------|-----------|
| **Síncrono** | REST (HTTP) | Warehouse Core → Robot Status | Verificar batería antes de asignar una orden |
| **Síncrono** | REST (HTTP) | Web UI → Identity | Login, registro y validación de JWT |
| **Síncrono** | REST (HTTP) | Nginx → Warehouse Core | CRUD de órdenes, inventario, layouts y rutas |
| **Asíncrono** | AMQP (RabbitMQ) | Warehouse Core → Robot Status | Notificar nuevas órdenes despachadas |
| **Asíncrono** | AMQP (RabbitMQ) | Warehouse Core → Logistics Analytics | Enviar rutas completadas para analítica |
| **Asíncrono** | AMQP (RabbitMQ) | Robot Status → Simulation | Enviar comandos a los robots |
| **Asíncrono** | STOMP/WS (RabbitMQ) | Simulation → Robot Status | Enviar telemetría de batería y posición |
| **Tiempo real** | SSE (HTTP) | Robot Status → Web UI | Transmitir telemetría sin refrescar la interfaz |

---

## Microservicios

| Servicio | Puerto | Base de Datos | Responsabilidad |
|----------|--------|---------------|----------------|
| **ms-warehouse-core** | `8081` | PostgreSQL (`warehouse_db`) | Órdenes, inventario, layouts, planificación Dijkstra y Outbox |
| **ms-robot-status** | `8082` | Redis | Estado de flota, batería, ubicación, comandos y telemetría SSE |
| **ms-logistics-analytics** | `8083` | MongoDB (`logistics_analytics`) | Consume `route.completed` y persiste rutas y congestión |
| **ms-identity** | `8084` | PostgreSQL (`auth_db`) | Registro, login, hashing y validación de tokens JWT |
| **Web UI** | `3000` | - | Dashboard operativo construido con Next.js |

---

## Tecnologías

### Backend

- **Lenguaje:** Java 21.
- **Framework:** Spring Boot 3.3.6, Spring Data JPA, Spring WebFlux, Spring AMQP y Spring Security Crypto.
- **Arquitectura:** Hexagonal, Ports & Adapters y separación por dominio, aplicación e infraestructura.
- **Bases de datos:** PostgreSQL 16, Redis 7 y MongoDB 7.
- **Mensajería:** RabbitMQ 3.13 con AMQP y Web STOMP.
- **Migraciones:** Flyway.
- **Build:** Maven con Wrapper.

### Frontend

- **Framework:** Next.js 14.2 con App Router y TypeScript.
- **Estilos:** Tailwind CSS 3.4.
- **Tiempo real:** Server-Sent Events (SSE).

### Infraestructura

- **Contenedores:** Docker Compose.
- **API Gateway:** Nginx 1.27 con validación JWT mediante `auth_request`.
- **Red:** Bridge `smartlogistic-net`.

### Observabilidad

- **Métricas:** Prometheus y Micrometer.
- **Dashboards:** Grafana preconfigurado.
- **Logs:** Loki y Promtail.
- **Trazabilidad distribuida:** OpenTelemetry Collector y Jaeger.

### Simulación

- **Motor:** Unreal Engine 5.7.
- **Comunicación:** STOMP WebSocket y HTTP REST.

---

## Requisitos Previos

- [Docker](https://docs.docker.com/get-docker/) y [Docker Compose](https://docs.docker.com/compose/install/).
- Java 21 o superior para desarrollo local.
- Node.js 20 o superior y npm.
- Unreal Engine 5.7 para ejecutar la simulación.
- Git.

---

## Inicio Rápido

### 1. Clonar el repositorio

```bash
git clone https://github.com/kevinmartinez07/smartlogistics-practice.git
cd smartlogistics-practice
```

### 2. Configurar variables de entorno

```bash
cd smartlogistic
cp .env.example .env
```

Edita `.env` y reemplaza los valores `change-this-*` antes de iniciar los servicios.

### 3. Iniciar backend e infraestructura

```bash
docker compose up --build -d
docker compose ps
```

Esto inicia las bases de datos, RabbitMQ, los cuatro microservicios, Nginx y el stack de observabilidad.

### 4. Iniciar el frontend

En otra terminal:

```bash
cd smartlogistics-practice/smartlogistic-web
npm ci
npm run dev
```

El frontend corre en `http://localhost:3000` y utiliza el gateway en `http://localhost:8086`.

### 5. Acceder a las interfaces

| Interfaz | URL | Uso |
|----------|-----|-----|
| **Web UI** | [http://localhost:3000](http://localhost:3000) | Gestión del almacén |
| **API Gateway** | [http://localhost:8086](http://localhost:8086) | API unificada |
| **Grafana** | [http://localhost:3001](http://localhost:3001) | Métricas, logs y dashboards |
| **Jaeger** | [http://localhost:16686](http://localhost:16686) | Trazas distribuidas |
| **RabbitMQ UI** | [http://localhost:15672](http://localhost:15672) | Administración del broker |
| **Prometheus** | [http://localhost:9090](http://localhost:9090) | Consultas de métricas |

Las credenciales se configuran en el `.env` local y no deben subirse al repositorio.

### 6. Detener todo

```bash
cd smartlogistic
docker compose down
```

Para eliminar también los volúmenes de bases de datos:

```bash
docker compose down -v
```

---

## Desarrollo Local

### Compilar un microservicio individual

```bash
cd smartlogistic/ms-warehouse-core
./mvnw clean package -DskipTests
```

### Ejecutar pruebas

```bash
cd smartlogistic/ms-warehouse-core
./mvnw test
```

Los demás servicios utilizan el mismo ciclo Maven desde sus respectivos directorios.

### Ejecutar el frontend

```bash
cd smartlogistic-web
npm ci
npm run dev
```

---

## Observabilidad

### Grafana

Dashboards preconfigurados para monitorear:

- **Visión general** de la plataforma.
- **ms-warehouse-core:** órdenes, rutas e inventario.
- **ms-robot-status:** flota, batería y telemetría.
- **ms-logistics-analytics:** eventos procesados y congestión.
- **ms-identity:** usuarios y autenticación.

### Jaeger

Permite seguir una operación de extremo a extremo: crear orden, validar robot, planificar la ruta y publicar los eventos asociados.

### Logs

Loki y Promtail agregan los logs de los contenedores para consultarlos desde Grafana Explore.

---

## Simulación Unreal Engine 5

El proyecto de simulación se encuentra en `Simulation/` y:

- Construye el almacén a partir del layout del backend.
- Se conecta a RabbitMQ mediante STOMP WebSocket.
- Recibe comandos de navegación y asignación de órdenes.
- Reporta batería, posición y estado al servicio `ms-robot-status`.
- Utiliza un fallback HTTP para consultar misiones pendientes.

Para abrirla, ejecuta `Simulation/Simulation.uproject` con Unreal Engine 5.7.

---

## Decisiones Arquitectónicas

### Comunicación síncrona y asíncrona

La validación de disponibilidad del robot y del inventario necesita respuesta inmediata, por lo que se realiza mediante REST. Las rutas completadas, comandos y actualizaciones de estado se distribuyen mediante RabbitMQ para no bloquear la operación principal.

### Outbox Pattern

Warehouse Core registra los eventos en la misma unidad de trabajo que la operación de negocio. Un proceso posterior publica los eventos pendientes, reduciendo el riesgo de confirmar una operación sin notificarla.

### Dijkstra y layout del almacén

La planificación de rutas utiliza una representación de cuadrícula. El algoritmo evita celdas bloqueadas, obstáculos y ubicaciones que no pueden atravesarse, y genera pasos que la simulación puede ejecutar.

### Persistencia poliglota

- PostgreSQL para datos transaccionales de identidad y almacén.
- Redis para lecturas rápidas del estado actual de robots.
- MongoDB para documentos de rutas y muestras de congestión.

### Arquitectura Hexagonal

El dominio no conoce Spring, PostgreSQL, Redis, RabbitMQ, HTTP ni SSE. La aplicación define puertos y los adaptadores de infraestructura implementan esos contratos, facilitando pruebas y sustitución de tecnologías.

### Atributos de calidad

- **Disponibilidad:** la asignación no depende del servicio de analítica.
- **Latencia:** la consulta del estado del robot se mantiene separada de procesos secundarios.
- **Integridad:** los eventos operativos se registran mediante Outbox antes de su publicación.
- **Observabilidad:** métricas, logs y trazas cubren los flujos entre servicios.

---

## Flujo de Trabajo Git

El repositorio utiliza `dev` como rama de integración y `main` para versiones estables:

```text
feature/<capacidad>  →  Pull Request  →  dev  →  Pull Request de release  →  main
```

Las funcionalidades se integran por bloques independientes y `dev` se promueve periódicamente a `main` cuando existe un incremento funcional validable.

---

## Seguridad

La configuración incluida está orientada a desarrollo local. Antes de desplegar públicamente:

- Cambia todas las credenciales del archivo `.env`.
- Genera un secreto JWT largo y aleatorio.
- Restringe CORS a dominios concretos.
- Protege los endpoints SSE y de misiones.
- No expongas bases de datos ni consolas administrativas a Internet.
- Fija y revisa las versiones de imágenes Docker.
- Evita tokens en parámetros de URL cuando exista una alternativa segura.

---

## Estructura del Proyecto

```text
smartlogistics-practice/
├── smartlogistic/                      # Backend e infraestructura
│   ├── docker-compose.yml              # Orquestación local
│   ├── .env.example                    # Variables de entorno
│   ├── nginx/                          # API Gateway
│   ├── rabbitmq/                       # Configuración del broker
│   ├── observability/                  # Prometheus, Grafana, Loki, Jaeger y OTEL
│   ├── scripts/                        # Datos iniciales
│   ├── ms-identity/                    # Autenticación
│   ├── ms-warehouse-core/              # Órdenes, inventario y rutas
│   ├── ms-robot-status/                # Estado de robots y telemetría
│   └── ms-logistics-analytics/         # Analítica logística
├── smartlogistic-web/                  # Frontend Next.js
├── Simulation/                         # Simulación Unreal Engine 5
├── docs/                               # Diagramas C4
├── .gitignore
└── LICENSE
```

---

## Licencia

Este repositorio se distribuye bajo la licencia MIT. Consulta [LICENSE](LICENSE).
