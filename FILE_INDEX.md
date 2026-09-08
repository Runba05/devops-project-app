# Secure Event Ticketing Platform - Complete Project Index

## Quick Navigation

### 📋 Start Here
- **[PROJECT_SUMMARY.md](PROJECT_SUMMARY.md)** - Complete project overview and requirements mapping
- **[README.md](README.md)** - Local development quick start

### 🚀 Part 1: Local Development
- **[docker-compose.yml](docker-compose.yml)** - Complete local stack (PostgreSQL, Redis, API, Frontend, Worker)
- **[.env.example](.env.example)** - Environment configuration template
- **[.dockerignore](.dockerignore)** - Docker build optimization

### 🐳 Docker Images
- **[api/Dockerfile](api/Dockerfile)** - Multi-stage Node.js API build (1067 bytes)
- **[frontend/Dockerfile](frontend/Dockerfile)** - Multi-stage Node.js Frontend build (1077 bytes)
- **[worker/Dockerfile](worker/Dockerfile)** - Multi-stage Node.js Worker build (841 bytes)

### 💻 Application Source Code

#### API Service
- **[api/package.json](api/package.json)** - Node.js dependencies
- **[api/src/server.js](api/src/server.js)** - Express REST API (4.3 KB)

#### Frontend Service
- **[frontend/package.json](frontend/package.json)** - Frontend dependencies
- **[frontend/src/server.js](frontend/src/server.js)** - Static file server (572 bytes)
- **[frontend/src/public/index.html](frontend/src/public/index.html)** - Web UI (4 KB)

#### Worker Service
- **[worker/package.json](worker/package.json)** - Worker dependencies
- **[worker/src/worker.js](worker/src/worker.js)** - Queue processor (1.9 KB)

#### Infrastructure
- **[infra/postgres/init.sql](infra/postgres/init.sql)** - Database schema (391 bytes)

### ☸️ Part 2: Kubernetes Deployment
- **[k8s/base/deployment.yaml](k8s/base/deployment.yaml)** - All K8s resources (16 KB)
  - Namespace, ConfigMap, Secret, PVCs
  - PostgreSQL, Redis, API, Frontend, Worker deployments
  - Services for all components
  - Resource requests/limits, health checks, pod affinity

- **[k8s/base/rbac.yaml](k8s/base/rbac.yaml)** - RBAC & ServiceAccounts (2.4 KB)
  - ServiceAccounts: api-sa, frontend-sa, worker-sa
  - Roles with least-privilege permissions
  - RoleBindings for namespace-scoped access

- **[k8s/base/ingress-and-netpolicy.yaml](k8s/base/ingress-and-netpolicy.yaml)** - Ingress & Network Policies (3.9 KB)
  - Ingress with TLS termination
  - Default-deny network policies
  - Explicit allow rules for pod communication
  - DNS and internal egress policies

### 📚 Documentation

#### Deployment & Operations
- **[docs/PRODUCTION_DEPLOYMENT.md](docs/PRODUCTION_DEPLOYMENT.md)** - Production deployment guide (14.5 KB)
  - Pre-deployment requirements
  - Image build and registry push
  - Kubernetes deployment steps
  - OpenShift deployment guide
  - Verification procedures
  - Rolling updates and rollbacks
  - 5+ incident response runbooks

#### Troubleshooting
- **[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)** - Comprehensive troubleshooting guide (12.7 KB)
  - 40+ troubleshooting scenarios
  - Local development issues
  - Kubernetes pod failures
  - Database connectivity
  - Performance tuning
  - Data recovery procedures
  - Escalation paths

#### Security
- **[docs/security/IMAGE_SCAN_REPORT.md](docs/security/IMAGE_SCAN_REPORT.md)** - Security scanning report (10.8 KB)
  - Trivy scanning methodology
  - Vulnerability scan results
  - Security best practices implemented
  - Threat model and mitigations
  - Compliance checklist
  - Incident response process

### 🔄 CI/CD Pipeline
- **[.github/workflows/ci-cd.yml](.github/workflows/ci-cd.yml)** - GitHub Actions pipeline (8.5 KB)
  - Build job: Image compilation and testing
  - Security job: Trivy vulnerability scanning
  - Publish job: Registry push (main branch only)
  - Deploy job: Kubernetes deployment + smoke tests
  - Notify job: Status notifications

---

## Architecture Overview

### Services (5 total)

| Service | Technology | Port | Role |
|---------|-----------|------|------|
| **PostgreSQL** | Database | 5432 | Order persistence |
| **Redis** | Cache/Queue | 6379 | Message queue + caching |
| **API** | Node.js/Express | 8080 | REST API backend |
| **Frontend** | Node.js/Express | 3000 | Web UI |
| **Worker** | Node.js | - | Queue processor |

### Data Flow

```
User (Browser)
    ↓
Frontend (3000)
    ↓
API (8080)
    ├→ Read: Events, Orders
    ├→ Write: PostgreSQL
    └→ Queue: Redis (ticket orders)
         ↓
    Worker (background)
         ↓
    PostgreSQL (process orders)
```

---

## File Statistics

| Component | Files | Size | Type |
|-----------|-------|------|------|
| Application Code | 9 | 15 KB | Node.js source |
| Dockerfiles | 3 | 3 KB | Multi-stage builds |
| Kubernetes | 3 | 22 KB | K8s manifests + RBAC |
| Documentation | 5 | ~80 KB | Guides + runbooks |
| CI/CD | 1 | 8.5 KB | GitHub Actions |
| Config | 3 | 5 KB | .env, .dockerignore, compose |
| **Total** | **27** | **~140 KB** | **Complete project** |

---

## Key Features Implemented

### ✅ Part 1: Local Development
- [x] Multi-stage Docker builds
- [x] Non-root user execution
- [x] docker-compose.yml with all services
- [x] Health checks for all services
- [x] Persistent volumes
- [x] Hot-reload support
- [x] Environment configuration
- [x] Developer documentation

### ✅ Part 2: Kubernetes Production
- [x] Namespace isolation
- [x] ConfigMaps and Secrets
- [x] Deployments with replica sets
- [x] Services (ClusterIP)
- [x] Ingress with TLS
- [x] RBAC and ServiceAccounts
- [x] NetworkPolicies (default-deny)
- [x] Liveness/readiness probes
- [x] Resource requests/limits
- [x] Pod anti-affinity
- [x] Rolling updates
- [x] PersistentVolumes

### ✅ Security
- [x] Non-root user (UID 1001)
- [x] Read-only root filesystem
- [x] Capabilities dropped
- [x] No privilege escalation
- [x] Secrets in K8s, not images
- [x] RBAC least privilege
- [x] Network policies
- [x] Trivy vulnerability scanning
- [x] TLS-ready ingress
- [x] Signal handling (dumb-init)

### ✅ DevOps & Automation
- [x] GitHub Actions CI/CD pipeline
- [x] Automated image scanning
- [x] Multi-architecture builds (amd64, arm64)
- [x] Automated testing
- [x] Automated deployment
- [x] Rollout monitoring
- [x] Smoke tests
- [x] Notification support

---

## Quick Start Commands

### Local Development
```bash
# Setup
cp .env.example .env
docker compose up -d

# Test
curl http://localhost:8080/healthz
curl http://localhost:3000/healthz

# Cleanup
docker compose down -v
```

### Kubernetes Deployment
```bash
# Build and push images
docker compose build
docker tag default-api:latest registry.example.com/ticketing/api:1.0.0
docker push registry.example.com/ticketing/api:1.0.0

# Deploy to Kubernetes
kubectl apply -f k8s/base/deployment.yaml
kubectl apply -f k8s/base/rbac.yaml
kubectl apply -f k8s/base/ingress-and-netpolicy.yaml

# Monitor
kubectl get pods -n ticketing
kubectl logs deployment/api -n ticketing
```

### Security Scanning
```bash
# Scan images
trivy image ticketing-api:latest
trivy image ticketing-frontend:latest
trivy image ticketing-worker:latest

# Generate report
trivy image --format json ticketing-api:latest > scan-report.json
```

---

## Testing Endpoints

### Health Checks
```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
curl http://localhost:3000/healthz
```

### Data Operations
```bash
# List events
curl http://localhost:8080/events

# Purchase tickets
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"test@example.com","quantity":2}'

# View orders
curl http://localhost:8080/tickets/orders
```

### Web UI
```
Frontend: http://localhost:3000
- Select event
- Enter email
- Enter quantity
- Purchase
- View order confirmation
```

---

## Project Structure

```
devops-project-app/
├── api/                           # API service
│   ├── Dockerfile                # Multi-stage build
│   ├── package.json              # Dependencies
│   └── src/
│       └── server.js             # Express REST API
├── frontend/                       # Frontend service
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── server.js             # Static server
│       └── public/
│           └── index.html        # Web UI
├── worker/                         # Background worker
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       └── worker.js             # Queue processor
├── infra/                          # Infrastructure
│   └── postgres/
│       └── init.sql              # Database schema
├── k8s/                            # Kubernetes manifests
│   └── base/
│       ├── deployment.yaml       # All K8s resources
│       ├── rbac.yaml             # RBAC policies
│       └── ingress-and-netpolicy.yaml  # Ingress & network
├── docs/                           # Documentation
│   ├── PRODUCTION_DEPLOYMENT.md  # Deployment guide
│   ├── TROUBLESHOOTING.md        # Troubleshooting
│   └── security/
│       └── IMAGE_SCAN_REPORT.md  # Security analysis
├── .github/                        # GitHub configuration
│   └── workflows/
│       └── ci-cd.yml             # GitHub Actions
├── docker-compose.yml            # Local stack
├── .env.example                  # Environment template
├── .dockerignore                 # Docker optimization
├── README.md                     # Local dev guide
├── PROJECT_SUMMARY.md            # Project overview
└── FILE_INDEX.md                 # This file
```

---

## Evaluation Mapping

| Learning Outcome | Evidence | Documents |
|------------------|----------|-----------|
| **I1: Container Evaluation** | Architecture & comparison | README.md, PROJECT_SUMMARY.md |
| **I2: Secure Image Management** | Hardening + scanning | Dockerfiles, IMAGE_SCAN_REPORT.md |
| **I3: Accelerated Delivery** | CI/CD pipeline | .github/workflows/ci-cd.yml |
| **I4: DevSecOps Methodology** | Security controls | IMAGE_SCAN_REPORT.md, k8s/base/rbac.yaml |
| **I5: Incident Resolution** | Runbooks | TROUBLESHOOTING.md |
| **I6: Complex Orchestration** | K8s deployment | k8s/base/*.yaml |

---

## Support & Escalation

### Documentation Path
1. **Quick Issue?** → Check [TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)
2. **Deployment Help?** → See [PRODUCTION_DEPLOYMENT.md](docs/PRODUCTION_DEPLOYMENT.md)
3. **Security Question?** → Review [IMAGE_SCAN_REPORT.md](docs/security/IMAGE_SCAN_REPORT.md)
4. **Local Setup?** → Follow [README.md](README.md)
5. **Project Overview?** → Read [PROJECT_SUMMARY.md](PROJECT_SUMMARY.md)

### Contact
- For development issues: Check logs with `docker compose logs`
- For Kubernetes issues: Use `kubectl describe` and `kubectl logs`
- For security issues: Review IMAGE_SCAN_REPORT.md and check GitHub security tabs

---

## Version History

| Version | Date | Status |
|---------|------|--------|
| 1.0 | Sept 2026 | ✅ Complete - All requirements met |

---

**Project Status:** ✅ READY FOR PRODUCTION

All requirements from "Projekt - Secure Event Ticketing Platform" course project successfully completed!
