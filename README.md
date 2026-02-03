# Emergency Alerts System

A real-time emergency alert management system for creating, approving, and delivering geographically-targeted alerts to affected populations.

## What This Does

This system helps emergency responders and government officials:

- **Create alerts** with geographic boundaries (GeoJSON polygons), severity levels, and expiry times
- **Approve or reject** alerts through a controlled workflow
- **Deliver** approved alerts via Azure Communication Services (email/SMS)
- **Monitor** alert status and delivery in real-time through a SignalR-powered dashboard
- **Track** geographic patterns, SLA breaches, and operational metrics via automated workflows

## Architecture Overview

The system runs on Azure Kubernetes Service (AKS) with three main components:

```
┌─────────────┐      ┌──────────────┐      ┌────────────┐
│   Frontend  │─────▶│   Backend    │─────▶│ PostgreSQL │
│ React + TS  │      │  .NET 8 API  │      │ + PostGIS  │
└─────────────┘      └──────────────┘      └────────────┘
                            │                      │
                            │                      │
                            ▼                      ▼
                     ┌──────────┐          ┌───────────┐
                     │  Drasi   │◀─────────│ CDC Stream│
                     │ Queries  │          └───────────┘
                     └──────────┘
                            │
                            ▼
                     ┌──────────────────┐
                     │ Azure Comm Svc   │
                     │ Email/SMS        │
                     └──────────────────┘
```

See [Architecture Diagrams](docs/diagrams/emergency-alerts-architecture.drawio) for detailed C4 Context, Container, and Lifecycle views.

### Key Technologies

- **Backend**: .NET 8 (C#) with DDD-lite architecture (Domain, Application, Infrastructure layers)
- **Frontend**: React 19 + TypeScript + Vite + Fluent UI v9
- **Database**: PostgreSQL with PostGIS extension for geospatial queries
- **Infrastructure**: Azure (AKS, App Configuration, Key Vault, ACR, Azure Maps)
- **Real-time**: SignalR for dashboard updates, Drasi for CDC-based workflows
- **Deployment**: Bicep (IaC), Kubernetes manifests, GitHub Actions workflows

## Project Structure

```
├── backend/              # .NET 8 API and domain logic
│   ├── src/
│   │   ├── EmergencyAlerts.Api/          # REST API + SignalR hub
│   │   ├── EmergencyAlerts.Application/  # Commands, queries, services
│   │   ├── EmergencyAlerts.Domain/       # Domain entities, value objects
│   │   └── EmergencyAlerts.Infrastructure/ # Repositories, EF Core, ACS
│   └── tests/             # Unit, integration, and load tests
│
├── frontend/             # React dashboard
│   ├── src/
│   │   ├── components/   # UI components (Fluent UI v9)
│   │   ├── pages/        # Dashboard, alert creation, approval
│   │   ├── services/     # API client, SignalR hub
│   │   └── types/        # TypeScript types
│   └── tests/            # Unit tests (Vitest) and E2E (Playwright)
│
├── infrastructure/       # Azure and Kubernetes configs
│   ├── bicep/           # Azure resources (AKS, PostgreSQL, App Config)
│   ├── k8s/             # Deployments, services, ingress, network policies
│   └── drasi/           # CDC sources, continuous queries, reactions
│
├── docs/                # Architecture and runbooks
│   └── diagrams/        # Draw.io architecture diagrams
```

## How It Works

### 1. Alert Lifecycle

```
Create → Pending Approval → Approved → Delivered
                ↓              ↓
            Rejected      Cancelled
```

**State Transitions:**
- **Create**: Operator submits alert with headline, description, severity, channel, geographic areas, and expiry
- **Approve/Reject**: Approver reviews and makes decision (with optional rejection reason)
- **Deliver**: Drasi CDC query detects approved alerts and triggers delivery via Azure Communication Services
- **Cancel**: Operator can cancel approved/delivered alerts to stop further processing

### 2. Real-time Workflows (Drasi)

Drasi monitors PostgreSQL changes and triggers automated actions:

- **Delivery Trigger**: Sends approved alerts to Azure Communication Services
- **SLA Monitoring**: Detects alerts nearing approval deadlines
- **Geographic Correlation**: Identifies overlapping alert areas
- **Severity Escalation**: Tracks severity changes and patterns
- **Expiry Warnings**: Notifies operators of soon-to-expire alerts

See [Drasi README](infrastructure/drasi/README.md) for deployment and query details.

### 3. Geospatial Features

- **Polygon Validation**: GeoJSON polygons must have ≥3 points, no self-intersections
- **Area Queries**: PostGIS spatial queries for overlapping regions and affected populations
- **Azure Maps**: Frontend displays alert areas on interactive maps

### 4. Security and Authentication

- **Managed Identity**: AKS pods use Workload Identity for Azure service access
- **App Configuration**: Dynamic config refresh (CORS, rate limits, feature flags)
- **Key Vault**: Secrets stored in Azure Key Vault, CSI Driver mounts them to pods
- **Network Policies**: Pod-level isolation with allow-list rules
- **Auth**: Optional Entra ID integration (configurable via `Auth:AllowAnonymous`)

## Quick Start

### Prerequisites

- Azure subscription with Contributor access
- Azure CLI 2.60+
- kubectl 1.29+
- Drasi CLI
- .NET 8 SDK (for local development)
- Node.js 20+ (for frontend development)

### Deploy to Azure

See [Infrastructure Deployment Runbook](infrastructure/DEPLOYMENT_RUNBOOK.md) for complete step-by-step instructions.

**TL;DR:**
```bash
# 1. Deploy Azure infrastructure
cd infrastructure/bicep
az deployment sub create --template-file main.bicep --location uksouth

# 2. Configure AKS
./scripts/setup-aks-post-deployment.ps1

# 3. Deploy Drasi sources, queries, reactions
cd infrastructure/drasi
drasi apply -f sources/postgres-cdc.yaml
drasi apply -f queries/emergency-alerts.yaml
drasi apply -f reactions/emergency-alerts-http.yaml

# 4. Deploy application
kubectl apply -f infrastructure/k8s/
```

### Local Development

**Backend:**
```bash
cd backend
dotnet restore
dotnet build
dotnet run --project src/EmergencyAlerts.Api
# API runs at https://localhost:7240
```

**Frontend:**
```bash
cd frontend
npm install
npm run dev
# Dashboard runs at http://localhost:5173
```

**Database (requires PostgreSQL with PostGIS):**
```bash
cd backend/src/EmergencyAlerts.Infrastructure
dotnet ef database update
```

## API Reference

**Key Endpoints:**
- `POST /api/v1/alerts` - Create new alert (returns 201 with ETag)
- `GET /api/v1/alerts` - List alerts with pagination and filters
- `GET /api/v1/alerts/{id}` - Get alert by ID (includes ETag)
- `POST /api/v1/alerts/{id}/approval` - Approve or reject alert
- `PUT /api/v1/alerts/{id}/cancel` - Cancel alert (requires valid ETag)
- `GET /health/ready` - Readiness probe (Kubernetes)
- `GET /health/live` - Liveness probe (Kubernetes)

**SignalR Hub:**
- `/api/hubs/alerts` - Real-time dashboard updates

## Common Tasks

| Task | Guide |
|------|-------|
| Deploy to production | [DEPLOYMENT_RUNBOOK.md](infrastructure/DEPLOYMENT_RUNBOOK.md) |
| Update Drasi queries | [drasi/README.md](infrastructure/drasi/README.md) |
| Configure CORS/auth | [AKS_POST_DEPLOYMENT_SETUP.md](infrastructure/AKS_POST_DEPLOYMENT_SETUP.md) |
| Troubleshoot deployments | [DEPLOYMENT_SAFETY_CHECKLIST.md](.github/DEPLOYMENT_SAFETY_CHECKLIST.md) |
| Run load tests | [backend/tests/Load/README.md](backend/tests/Load/README.md) |
| Migrate from Elastic Cluster | [ELASTIC_CLUSTER_MIGRATION.md](infrastructure/docs/ELASTIC_CLUSTER_MIGRATION.md) |

## Testing

**Backend Tests:**
```bash
cd backend
dotnet test                                    # All tests
dotnet test --filter Category=Unit            # Unit tests only
dotnet test --filter Category=Integration     # Integration tests
```

**Frontend Tests:**
```bash
cd frontend
npm test              # Unit tests (Vitest)
npm run test:e2e      # E2E tests (Playwright)
```

**Load Tests:**
```bash
cd backend/tests/Load
k6 run api-load-test.js
```

## Configuration

Key configuration sources (in precedence order):

1. **Azure App Configuration** (dynamic, refreshes every 5 minutes)
   - CORS origins, feature flags, rate limits, delivery mode
2. **Kubernetes ConfigMaps** (mounted as env vars)
   - API URLs, SignalR hub endpoints
3. **Azure Key Vault** (via CSI Driver)
   - Database connection strings, ACS keys, Drasi reaction tokens
4. **appsettings.json** (fallback for local development)

## Observability

- **Logs**: Structured JSON logs with correlation IDs (`X-Correlation-ID`)
- **Metrics**: Prometheus metrics exposed at `/metrics` (if enabled)
- **Health Checks**: `/health/ready` and `/health/live` for Kubernetes probes
- **Tracing**: Request/response logging with anonymized PII

## Contributing

See [.github/copilot-instructions.md](.github/copilot-instructions.md) for development standards and conventions.

**Before submitting PRs:**
- Run linters: `./scripts/lint-all.ps1`
- Ensure tests pass: `dotnet test` and `npm test`
- Update relevant documentation (README, ADRs, runbooks)
- Follow [Conventional Commits](https://www.conventionalcommits.org/)

## Documentation

- **Architecture**: [docs/diagrams/](docs/diagrams/) (Draw.io C4 models)
- **Specifications**: [specs/001-emergency-alert-core/](specs/001-emergency-alert-core/)
- **Runbooks**: [infrastructure/docs/runbooks/](infrastructure/docs/runbooks/)
- **Migration Guides**: [infrastructure/docs/](infrastructure/docs/)

## License

[Add license information here]

---

**Questions or Issues?**

Check the [runbooks](infrastructure/docs/runbooks/) first, then open an issue with:
- What you tried
- Expected vs actual behavior
- Logs with correlation IDs
- Environment (dev/staging/prod)
