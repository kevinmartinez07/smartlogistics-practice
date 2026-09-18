# SmartLogistics Practice

SmartLogistics Practice is a hands-on distributed-systems project for managing autonomous warehouse operations. It combines a web dashboard, Spring Boot services, event-driven messaging, observability, and an Unreal Engine simulation.

The project is organized as a monorepo so each subsystem can be developed and tested independently while still running as a complete local environment.

## What This Project Demonstrates

- Hexagonal architecture and domain-oriented service boundaries.
- Synchronous REST communication for decisions that need an immediate response.
- Asynchronous RabbitMQ events for robot commands, route completion, and analytics.
- PostgreSQL, Redis, and MongoDB used according to workload.
- JWT authentication through an Nginx API gateway.
- Server-Sent Events for live warehouse and robot updates.
- Prometheus, Grafana, Loki, OpenTelemetry, and Jaeger observability.
- Unreal Engine visualization connected to the operational platform.

## Architecture

```text
Next.js dashboard
        |
        v
Nginx API gateway ---- ms-identity ---- PostgreSQL
        |
        +------------ ms-warehouse-core ---- PostgreSQL
        |                       |
        |                       +---------- RabbitMQ ---------- ms-logistics-analytics ---- MongoDB
        |                                    |
        +------------ ms-robot-status -------+---- Unreal Engine simulation
                              |
                             Redis
```

The warehouse service handles orders, inventory, layout data, route planning, and the outbox publisher. The robot service manages current fleet state and live telemetry. Analytics consumes completed-route events without blocking operational flows.

Detailed C4 diagrams are available in [`docs/`](docs/).

## Repository Layout

```text
smartlogistics-practice/
├── smartlogistic/       # Backend services and local infrastructure
├── smartlogistic-web/   # Next.js dashboard
├── Simulation/          # Unreal Engine simulation
└── docs/                # Technical architecture diagrams
```

## Local Requirements

- Docker and Docker Compose
- Java 21 or newer
- Node.js 20 or newer
- Unreal Engine 5.7 for the simulation

## Quick Start

Copy the example configuration and change every development secret before starting the stack:

```bash
cd smartlogistic
cp .env.example .env
docker compose up --build -d
```

The backend gateway is available at `http://localhost:8086`. The web dashboard runs separately during development:

```bash
cd smartlogistic-web
npm ci
npm run dev
```

The dashboard is available at `http://localhost:3000`.

## Services

| Component | Default port | Responsibility |
| --- | ---: | --- |
| `ms-identity` | 8084 | User registration, login, and JWT validation |
| `ms-warehouse-core` | 8081 | Orders, inventory, layouts, and route planning |
| `ms-robot-status` | 8082 | Robot state, commands, and telemetry |
| `ms-logistics-analytics` | 8083 | Route events and congestion analytics |
| Nginx gateway | 8086 | API routing and authentication boundary |
| Next.js dashboard | 3000 | Operations interface |
| Grafana | 3001 | Dashboards and log exploration |
| RabbitMQ management | 15672 | Broker administration |
| Jaeger | 16686 | Distributed traces |

## Development Workflow

The default integration branch is `dev`. Feature branches are opened from `dev`, tested locally, and merged through pull requests. Stable milestones are promoted from `dev` to `main` through release pull requests instead of waiting for the entire roadmap.

Suggested branch names:

```text
feature/<capability>
fix/<issue>
docs/<topic>
```

## Testing

Each backend service contains its own tests. The Maven wrapper is available in the services that provide local Maven wrapper scripts. The frontend uses the standard Next.js build command:

```bash
cd smartlogistic/ms-warehouse-core
./mvnw test

cd ../../../smartlogistic-web
npm run build
```

## Unreal Engine Simulation

Open `Simulation/Simulation.uproject` with Unreal Engine 5.7. The simulation builds the warehouse environment, receives robot commands through RabbitMQ Web STOMP, and publishes telemetry through the robot-status service.

## Configuration and Security

- Never commit `.env` or other local environment files.
- Replace all values in `.env.example` before using the stack outside local development.
- Do not expose database or administration ports on an untrusted network.
- Restrict CORS and protect SSE endpoints before deploying publicly.
- Pin and review container image versions for production use.

## License

This repository is distributed under the MIT License. See [LICENSE](LICENSE).
