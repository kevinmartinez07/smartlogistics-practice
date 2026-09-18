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

```bash
cd smartlogistic
cp .env.example .env
docker compose up --build -d
```

The web dashboard is developed separately:

```bash
cd smartlogistic-web
npm install
npm run dev
```

The dashboard is available at `http://localhost:3000` and the API gateway at `http://localhost:8086` by default.

## Services

| Component | Default port | Responsibility |
| --- | ---: | --- |
| `ms-identity` | 8084 | User registration, login, and JWT validation |
| `ms-warehouse-core` | 8081 | Orders, inventory, layouts, and route planning |
| `ms-robot-status` | 8082 | Robot state, commands, and telemetry |
| `ms-logistics-analytics` | 8083 | Route events and congestion analytics |
| Nginx gateway | 8086 | API routing and authentication boundary |
| Next.js dashboard | 3000 | Operations interface |

## Development Workflow

The default integration branch is `dev`. Feature branches should be opened against `dev`, tested locally, and merged through pull requests. Stable milestones are promoted from `dev` to `main` through a release pull request.

## License

This repository is distributed under the MIT License. See [LICENSE](LICENSE).
