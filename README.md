# Secure Event Ticketing Platform - DevOps Project

> Sample application for Uvod u DevOps - DevSecOps course at Sveučilište Algebra Bernays, Zagreb

## Project Overview

A multi-tier containerized application demonstrating secure application delivery through the complete DevOps/DevSecOps lifecycle. The platform manages event ticketing with a web UI, REST API, message queue processing, and persistent storage.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   Frontend (Node.js/Express)                │
│                     Port 3000 - Web UI                       │
└────────────────┬────────────────────────────────────────────┘
                 │ HTTP/REST
                 ▼
┌─────────────────────────────────────────────────────────────┐
│                   API (Node.js/Express)                     │
│              Port 8080 - REST Endpoints                      │
└─────┬──────────────────────────────┬────────────────────────┘
      │                              │
      ▼                              ▼
┌──────────────────┐        ┌──────────────────┐
│   PostgreSQL     │        │     Redis        │
│   Port 5432      │        │   Port 6379      │
│   Persistence    │        │  Queue/Cache     │
└──────────────────┘        └──────────────────┘
      ▲                              ▲
      │                              │
      └──────────────┬───────────────┘
                     │
                     ▼
        ┌──────────────────────────┐
        │  Worker (Node.js)        │
        │  Queue Processor         │
        └──────────────────────────┘
```

### Services

| Service | Technology | Port | Role |
|---------|-----------|------|------|
| **Frontend** | Node.js + Express | 3000 | Web UI for browsing events and purchasing tickets |
| **API** | Node.js + Express | 8080 | REST API for events, ticket purchases, health checks |
| **Worker** | Node.js | - | Background job processor for queue messages |
| **PostgreSQL** | Database | 5432 | Persistent order storage |
| **Redis** | Cache/Queue | 6379 | Message queue and caching layer |

## Part 1: Local Development Environment

### Prerequisites

- Docker Desktop 4.20+ or Docker Engine + Docker Compose 2.20+
- Git
- Internet connection (for downloading images)

### Quick Start

1. **Clone the repository:**
   ```bash
   git clone https://github.com/matej-basic/devops-project-app.git
   cd devops-project-app
   ```

2. **Create environment file:**
   ```bash
   cp .env.example .env
   ```

3. **Build and start all services:**
   ```bash
   docker compose up --build
   ```

   The application will be available at:
   - **Frontend UI:** http://localhost:3000
   - **API:** http://localhost:8080

### Development Workflow

#### Hot-Reload (Auto-Restart on Code Changes)

The `docker-compose.yml` is configured with `develop` watch mode for automatic reloading:

```yaml
develop:
  watch:
    - path: ./api/src
      action: sync
    - path: ./frontend/src
      action: sync
    - path: ./worker/src
      action: sync
```

This means changes to source files automatically sync into running containers and trigger reloads via `nodemon`.

#### Starting Services

**With builds (first run or after Dockerfile changes):**
```bash
docker compose up --build
```

**Without rebuilds (faster subsequent runs):**
```bash
docker compose up
```

**In detached mode (background):**
```bash
docker compose up -d
```

#### Stopping Services

**Stop all services (data persists):**
```bash
docker compose stop
```

**Stop and remove containers (volumes remain):**
```bash
docker compose down
```

**Complete cleanup (removes volumes too):**
```bash
docker compose down -v
```

### Validation

#### Health Checks

All services have embedded health checks. View container status:

```bash
docker compose ps
```

Expected output shows all services as "Up" with health status.

#### API Endpoints

**1. Health Check (always first):**
```bash
curl http://localhost:8080/healthz
```

Response:
```json
{"status": "ok", "service": "api"}
```

**2. Readiness Check (dependencies ready):**
```bash
curl http://localhost:8080/readyz
```

Response:
```json
{"status": "ready"}
```

**3. List Events:**
```bash
curl http://localhost:8080/events
```

Response:
```json
[
  {
    "id": "evt-1001",
    "name": "DevSecOps Bootcamp",
    "location": "Zagreb",
    "availableTickets": 150
  },
  ...
]
```

**4. Purchase Ticket:**
```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{
    "eventId": "evt-1001",
    "customerEmail": "student@example.com",
    "quantity": 2
  }'
```

Response:
```json
{
  "message": "Order queued",
  "orderId": "66b7978f-34c6-4296-8404-3ebf7ad16d4f"
}
```

**5. View Processed Orders:**
```bash
curl http://localhost:8080/tickets/orders
```

Response:
```json
[
  {
    "order_id": "66b7978f-34c6-4296-8404-3ebf7ad16d4f",
    "event_id": "evt-1001",
    "customer_email": "student@example.com",
    "quantity": 2,
    "status": "processed",
    "created_at": "2026-03-05T19:58:40.954Z"
  }
]
```

#### UI Walkthrough

1. Navigate to http://localhost:3000
2. Select an event from dropdown
3. Enter email address (default: student@example.com)
4. Enter quantity
5. Click **Purchase**
6. View confirmation with Order ID

### Environment Configuration

**Local development (.env file):**

```env
# Database
POSTGRES_DB=ticketing
POSTGRES_USER=ticketing_user
POSTGRES_PASSWORD=change_me_local
POSTGRES_HOST=postgres
POSTGRES_PORT=5432

# Cache/Queue
REDIS_HOST=redis
REDIS_PORT=6379

# Services
API_PORT=8080
FRONTEND_PORT=3000
API_BASE_URL=http://api:8080

# Queue
QUEUE_NAME=ticket_orders

# Mode
NODE_ENV=development
```

**Override specific variables:**
```bash
POSTGRES_PASSWORD=my_secure_pass docker compose up
```

### Data Persistence

**Database volumes:**
- `postgres_data` - PostgreSQL tablespaces (survives `docker compose down`)
- `redis_data` - Redis snapshot (survives `docker compose down`)

**To reset data:**
```bash
docker compose down -v
```

### Troubleshooting

#### Services fail to start

1. **Check service logs:**
   ```bash
   docker compose logs api
   docker compose logs postgres
   docker compose logs redis
   ```

2. **Common issues:**
   - Port already in use: Change `FRONTEND_PORT`, `API_PORT`, `POSTGRES_PORT` in `.env`
   - Insufficient disk space: Run `docker system prune`
   - Permission denied: Run with `sudo` or add user to docker group

#### Connectivity issues between services

Services communicate via Docker network `ticketing-network`. Verify:

```bash
docker network inspect ticketing-network
```

All services should appear in the "Containers" section.

#### Database not initializing

1. Check postgres logs:
   ```bash
   docker compose logs postgres
   ```

2. Verify init.sql exists:
   ```bash
   cat infra/postgres/init.sql
   ```

3. Recreate with fresh volume:
   ```bash
   docker compose down -v
   docker compose up
   ```

#### Redis connection errors

Restart Redis service:
```bash
docker compose restart redis
docker compose logs redis
```

### Security Features (Local Dev)

- ✅ Non-root user (UID 1001) in containers
- ✅ Multi-stage Docker builds (reduced image size)
- ✅ Health checks for all services
- ✅ Secrets in `.env` (not in code)
- ✅ Network isolation via Docker bridge
- ⚠️ Note: Local .env has default credentials — never use in production!

### Performance Tips

1. **Use named volumes for database:**
   - Faster than bind mounts on Docker Desktop for macOS/Windows
   - Already configured in `docker-compose.yml`

2. **Disable hot-reload if not needed:**
   ```bash
   docker compose up --no-build
   ```

3. **Monitor resource usage:**
   ```bash
   docker stats
   ```

## Next Steps

- **Part 2:** See [`docs/PRODUCTION_DEPLOYMENT.md`](docs/PRODUCTION_DEPLOYMENT.md) for Kubernetes deployment
- **Security:** See [`docs/security/IMAGE_SCAN_REPORT.md`](docs/security/IMAGE_SCAN_REPORT.md) for vulnerability scanning
- **CI/CD:** See [`.github/workflows/`](.github/workflows/) for automated build and test pipeline

## Project Requirements Checklist

### Part 1 - Local Development ✅

- [x] Containerize all services with Dockerfiles
- [x] Multi-stage builds with minimal runtime images
- [x] Non-root user in all containers
- [x] docker-compose.yml with all services
- [x] Hot-reload support for development
- [x] Environment variables through .env
- [x] Volume persistence for PostgreSQL
- [x] Health checks for each service
- [x] Developer documentation
- [x] Functional validation runbook

### Part 2 - Production (See PRODUCTION_DEPLOYMENT.md)

- [ ] Kubernetes manifests with Deployments, Services
- [ ] ConfigMap and Secret objects
- [ ] Liveness and readiness probes
- [ ] Ingress with external access
- [ ] Resource requests and limits
- [ ] Rolling updates and rollback procedures
- [ ] Image vulnerability scanning
- [ ] RBAC and least-privilege service accounts
- [ ] Network policies for traffic segmentation

## Contributing

1. Create feature branch: `git checkout -b feature/name`
2. Commit changes: `git commit -am 'Add feature'`
3. Push branch: `git push origin feature/name`
4. Submit pull request

## License

Educational project for Sveučilište Algebra Bernays, Zagreb.

## Support

For issues, create a GitHub issue or contact the course instructors.
