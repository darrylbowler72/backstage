# Backstage Architecture Documentation

## Table of Contents
1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Technology Stack](#technology-stack)
4. [Backend Architecture](#backend-architecture)
5. [Frontend Architecture](#frontend-architecture)
6. [Configuration Management](#configuration-management)
7. [Data Flow](#data-flow)
8. [Deployment Architecture](#deployment-architecture)
9. [Security](#security)
10. [Extensibility](#extensibility)

## Overview

This Backstage instance is configured as an internal developer portal for the `darrylbowler72` organization. It provides a unified interface for software catalog management, documentation, scaffolding, CI/CD monitoring, and Kubernetes resource management.

**Backstage Version:** 1.45.0
**Organization:** darrylbowler72
**Primary Integration:** GitHub
**Deployment Model:** Monorepo with containerized production deployment

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Browser (Port 3000)                      │
│              React SPA with Material-UI Components              │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTP/REST API
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                    Backend (Port 7007)                           │
│                   Node.js Express Server                         │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Catalog    │  │  Scaffolder  │  │   TechDocs   │          │
│  │    Plugin    │  │    Plugin    │  │    Plugin    │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                   │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐          │
│  │     Auth     │  │   Search     │  │  Kubernetes  │          │
│  │    Plugin    │  │    Plugin    │  │    Plugin    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Notifications│  │   Signals    │  │  Permission  │          │
│  │    Plugin    │  │    Plugin    │  │    Plugin    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
┌────────▼────────┐ ┌───▼────────┐ ┌───▼──────────┐
│   PostgreSQL    │ │   GitHub   │ │   Docker     │
│   (Production)  │ │    API     │ │  (TechDocs)  │
│     SQLite      │ └────────────┘ └──────────────┘
│  (Development)  │
└─────────────────┘
```

### Component Interactions

1. **Frontend → Backend**: RESTful API calls with JWT authentication
2. **Backend → GitHub**: OAuth authentication, repository discovery, API operations
3. **Backend → Database**: Entity storage, user sessions, search indexing
4. **Backend → Docker**: TechDocs generation in isolated containers
5. **Backend → Kubernetes**: Cluster resource monitoring and display

## Technology Stack

### Core Technologies

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| Runtime | Node.js | 20/22 | JavaScript runtime for both frontend and backend |
| Package Manager | Yarn | 4.4.1 (Berry) | Dependency management and monorepo orchestration |
| Build System | Backstage CLI | 0.34.5 | Build, test, and development tooling |
| Type System | TypeScript | 5.8.0 | Type-safe JavaScript development |

### Backend Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Framework | Express.js | HTTP server and routing |
| Database (Dev) | SQLite (better-sqlite3) | In-memory data storage for development |
| Database (Prod) | PostgreSQL | Persistent data storage for production |
| Authentication | Passport.js | OAuth and authentication strategies |
| Container Runtime | Docker | TechDocs generation |
| Bundler | Backstage Backend System | Plugin orchestration |

### Frontend Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Framework | React 18 | UI component library |
| UI Library | Material-UI v4 | Component design system |
| Routing | React Router v6 | Client-side routing |
| Bundler | Rspack | Fast module bundling |
| State Management | React Context | Application state |

## Backend Architecture

### Plugin System

The backend uses Backstage's new backend system with modular plugins. All plugins are registered in `packages/backend/src/index.ts` using the `backend.add()` pattern.

#### Core Plugins

**1. App Backend (`@backstage/plugin-app-backend`)**
- Serves the frontend React application as static assets
- Provides the main HTTP server endpoint

**2. Proxy Backend (`@backstage/plugin-proxy-backend`)**
- Routes API requests from frontend to external services
- Handles CORS and authentication headers
- Configuration: `app-config.yaml` → `proxy.endpoints`

**3. Catalog (`@backstage/plugin-catalog-backend`)**
- Central service for entity management
- Maintains software catalog: Components, Systems, APIs, Resources, Users, Groups
- Modules:
  - `catalog-backend-module-scaffolder-entity-model`: Template entity support
  - `catalog-backend-module-github`: GitHub repository discovery
  - `catalog-backend-module-logs`: Error logging and monitoring

**4. Scaffolder (`@backstage/plugin-scaffolder-backend`)**
- Software template execution engine
- GitHub integration for repository creation and PRs
- Notification integration for template execution updates
- Actions available: 40+ built-in actions including:
  - GitHub: repo creation, webhooks, branch protection, issues
  - Filesystem: read, write, rename, delete
  - Catalog: register, fetch, write
  - Debugging: log, wait

**5. TechDocs (`@backstage/plugin-techdocs-backend`)**
- Technical documentation system
- Builder: Local (Docker-based)
- Generator: Docker containers for mkdocs
- Publisher: Local filesystem storage
- Supports Markdown with MkDocs Material theme

**6. Auth (`@backstage/plugin-auth-backend`)**
- Authentication and authorization
- Providers:
  - **GitHub OAuth**: Production authentication with `clientId` and `clientSecret`
  - **Guest**: Development authentication (no credentials required)
- JWT token issuance for service-to-service communication
- Session management via signing keys

**7. Permission (`@backstage/plugin-permission-backend`)**
- Role-based access control (RBAC) framework
- Current policy: Allow-all (development mode)
- Production requires custom permission policies

**8. Search (`@backstage/plugin-search-backend`)**
- Full-text search engine
- Engine: PostgreSQL (production), falls back for SQLite in dev
- Collators:
  - Software Catalog: Indexes all catalog entities
  - TechDocs: Indexes documentation content
- Indexing frequency: Every 10 minutes with 3-second initial delay

**9. Kubernetes (`@backstage/plugin-kubernetes-backend`)**
- Kubernetes cluster integration
- Displays cluster resources for catalog entities
- Configuration: `app-config.yaml` → `kubernetes` (currently not configured)

**10. Notifications & Signals (`@backstage/plugin-notifications-backend`, `@backstage/plugin-signals-backend`)**
- Real-time notification system
- WebSocket-based signal broadcasting
- Event bus integration (when available)
- Notification cleanup: Daily at midnight

### Database Schema

#### Development (SQLite)
- Connection: In-memory (`:memory:`)
- Data persistence: None (resets on restart)
- Use case: Local development and testing

#### Production (PostgreSQL)
- Connection: Environment variables (`POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_USER`, `POSTGRES_PASSWORD`)
- Tables managed by plugins:
  - `entities`: Catalog entity storage
  - `auth_sessions`: User authentication sessions
  - `search_index`: Full-text search index
  - `scheduled_tasks`: Task scheduler state
  - Plus plugin-specific tables

### API Endpoints

| Endpoint | Plugin | Purpose |
|----------|--------|---------|
| `/api/catalog/*` | Catalog | Entity CRUD operations |
| `/api/techdocs/*` | TechDocs | Documentation serving |
| `/api/auth/*` | Auth | Authentication flows |
| `/api/scaffolder/*` | Scaffolder | Template execution |
| `/api/search/*` | Search | Search queries |
| `/api/kubernetes/*` | Kubernetes | K8s resource queries |
| `/api/permission/*` | Permission | Authorization checks |
| `/api/notifications/*` | Notifications | Notification management |
| `/api/signals/*` | Signals | WebSocket connections |

### Scheduled Tasks

| Task | Frequency | Plugin | Purpose |
|------|-----------|--------|---------|
| `github-provider:providerId:refresh` | 30 minutes | Catalog | GitHub repository discovery |
| `catalog_orphan_cleanup` | 30 seconds | Catalog | Remove orphaned entities |
| `notification-cleaner` | Daily (midnight) | Notifications | Cleanup old notifications |
| `search_index_software_catalog` | 10 minutes | Search | Reindex catalog entities |
| `search_index_techdocs` | 10 minutes | Search | Reindex documentation |

## Frontend Architecture

### Application Structure

```
packages/app/
├── src/
│   ├── App.tsx                    # Root application component
│   ├── components/
│   │   ├── Root/                  # Root layout and navigation
│   │   ├── catalog/
│   │   │   └── EntityPage.tsx     # Entity detail pages
│   │   └── search/
│   │       └── SearchPage.tsx     # Search interface
│   └── apis.ts                    # API factory configurations
└── public/                        # Static assets
```

### Installed Plugins

**Core Plugins:**
- `@backstage/plugin-catalog`: Software catalog browser
- `@backstage/plugin-catalog-import`: Register new components
- `@backstage/plugin-scaffolder`: Software templates
- `@backstage/plugin-techdocs`: Documentation viewer
- `@backstage/plugin-search`: Search interface
- `@backstage/plugin-user-settings`: User preferences

**Additional Plugins:**
- `@backstage/plugin-api-docs`: API documentation
- `@backstage/plugin-catalog-graph`: Entity relationship visualization
- `@backstage/plugin-kubernetes`: Kubernetes resource viewer
- `@backstage/plugin-org`: Organization hierarchy
- `@backstage/plugin-notifications`: Notification center
- `@backstage/plugin-signals`: Real-time updates
- `@backstage-community/plugin-github-actions`: CI/CD monitoring

### Entity Page Customization

The `EntityPage.tsx` defines different layouts for each entity kind:

#### Component Pages (by type)

**Service Component:**
- Tabs: Overview, CI/CD, Kubernetes, API, Dependencies, Docs
- Overview includes: About card, Catalog graph, GitHub Actions runs, Links, Subcomponents
- CI/CD displays GitHub Actions workflows
- Kubernetes tab shows cluster resources (conditional)

**Website Component:**
- Tabs: Overview, CI/CD, Kubernetes, Dependencies, Docs
- Similar to service but without API tab

**Default Component:**
- Tabs: Overview, Docs

#### Other Entity Pages

- **API**: Overview with providers/consumers, Definition tab with API spec
- **System**: Overview with components/APIs/resources, Diagram tab with full system graph
- **Domain**: Overview with systems list
- **User**: Profile and ownership information
- **Group**: Profile, ownership, members, links

### UI Customization Points

- **EntitySwitch**: Conditional rendering based on entity kind/type
- **EntityLayout**: Tab-based navigation for entity pages
- **Catalog Graph**: Relationship visualization with customizable relations
- **TechDocs Addons**: Report Issue button for documentation feedback

## Configuration Management

### Configuration Files

**1. `app-config.yaml` (Base)**
- Development configuration
- Organization settings
- Plugin configurations
- GitHub integration
- Database: SQLite in-memory

**2. `app-config.local.yaml` (Local Overrides)**
- Git-ignored file for personal settings
- Local environment variables
- Development secrets

**3. `app-config.production.yaml` (Production)**
- Production-specific settings
- PostgreSQL database configuration
- External URLs
- Guest-only authentication

### Configuration Layers

The configuration system uses a layered approach with environment variable substitution:

```
Base Config (app-config.yaml)
    ↓
Local Overrides (app-config.local.yaml)
    ↓
Environment Variables (${VAR_NAME})
    ↓
Production Config (app-config.production.yaml)
```

### Required Environment Variables

**Development:**
```bash
GITHUB_TOKEN              # GitHub Personal Access Token for API access
GITHUB_CLIENT_ID          # GitHub OAuth App Client ID
GITHUB_CLIENT_SECRET      # GitHub OAuth App Client Secret
```

**Production:**
```bash
# All development variables plus:
POSTGRES_HOST            # PostgreSQL server hostname
POSTGRES_PORT            # PostgreSQL server port
POSTGRES_USER            # PostgreSQL username
POSTGRES_PASSWORD        # PostgreSQL password
```

### GitHub Integration Configuration

**Organization:** darrylbowler72

**Catalog Discovery:**
- Scans all repositories in the organization
- Looks for `catalog-info.yaml` files
- Filter: Discovers repos on any default branch (main or master)
- Schedule: Every 30 minutes with 3-minute timeout

**Scaffolder Actions:**
- Repository creation
- Branch protection setup
- Webhook configuration
- GitHub Actions workflow dispatch
- Pull request creation
- Issue management
- Deploy key and environment setup

## Data Flow

### Catalog Discovery Flow

```
┌──────────────┐
│   GitHub     │
│     API      │
└──────┬───────┘
       │ 1. Scheduled fetch (every 30 min)
       │
┌──────▼───────────────────────────────┐
│  GitHub Catalog Provider             │
│  - Discovers repositories            │
│  - Finds catalog-info.yaml files     │
└──────┬───────────────────────────────┘
       │ 2. Parse YAML entities
       │
┌──────▼───────────────────────────────┐
│  Catalog Processing Engine           │
│  - Validates entity schemas          │
│  - Resolves entity references        │
│  - Builds relationship graph         │
└──────┬───────────────────────────────┘
       │ 3. Store processed entities
       │
┌──────▼───────────────────────────────┐
│  Database (PostgreSQL/SQLite)        │
│  - Stores entities                   │
│  - Maintains indexes                 │
└──────┬───────────────────────────────┘
       │ 4. Publish to search
       │
┌──────▼───────────────────────────────┐
│  Search Indexer                      │
│  - Full-text indexing                │
│  - Updates every 10 minutes          │
└──────────────────────────────────────┘
```

### Authentication Flow

```
┌───────────┐                    ┌────────────┐
│  Browser  │                    │   GitHub   │
└─────┬─────┘                    └──────┬─────┘
      │ 1. Request /auth/github         │
      ▼                                  │
┌────────────┐                           │
│  Backend   │                           │
│ Auth Plugin│                           │
└─────┬──────┘                           │
      │ 2. Redirect to GitHub OAuth      │
      ├──────────────────────────────────▶
      │                                   │
      │ 3. User authorizes               │
      │◀──────────────────────────────────┤
      │ 4. Callback with code             │
      ▼                                   │
┌────────────┐                           │
│  Backend   │ 5. Exchange code for token│
│ Auth Plugin├──────────────────────────▶│
└─────┬──────┘◀──────────────────────────┤
      │ 6. Get user profile              │
      ▼                                   │
┌────────────┐                           │
│  Catalog   │ 7. Lookup/create user     │
│   Plugin   │                           │
└─────┬──────┘                           │
      │ 8. Issue JWT token               │
      ▼                                   │
┌───────────┐                            │
│  Browser  │◀────────────────────────────
│  (Cookie) │
└───────────┘
```

### Scaffolder Template Execution Flow

```
User selects template
        ↓
Fill template parameters
        ↓
┌─────────────────────────────┐
│   Scaffolder Backend        │
│   1. Validate parameters    │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│   Execute Actions           │
│   - fetch:template          │
│   - github:repo:create      │
│   - catalog:register        │
└──────────┬──────────────────┘
           │
    ┌──────┴──────┬──────────────┐
    │             │              │
    ▼             ▼              ▼
┌────────┐   ┌────────┐    ┌──────────┐
│ GitHub │   │ Catalog│    │Notification│
│  Repo  │   │ Entity │    │  System   │
└────────┘   └────────┘    └──────────┘
```

## Deployment Architecture

### Development Deployment

**Command:** `yarn start`

- Frontend: Hot-reloading development server on port 3000
- Backend: Node.js process on port 7007
- Database: SQLite in-memory
- TechDocs: Docker-based generation on-demand
- CORS: Enabled for `http://localhost:3000`

### Production Deployment

#### Docker Build Process

The multi-stage Dockerfile (`packages/backend/Dockerfile`) creates an optimized production image:

**Stage 1: Base Image**
- Node.js 22 Bookworm Slim
- Python 3, g++, build-essential (for native modules)
- libsqlite3-dev (for better-sqlite3)

**Stage 2: Dependency Installation**
- Copy Yarn configuration (.yarn, .yarnrc.yml)
- Extract skeleton.tar.gz (package.json files)
- Install production dependencies via `yarn workspaces focus --production`

**Stage 3: Application Bundle**
- Copy backend bundle (bundle.tar.gz)
- Copy configuration files (app-config.yaml, app-config.production.yaml)
- Copy examples directory

**Runtime Configuration:**
- Port: 7007
- User: node (non-root)
- Command: `node packages/backend --config app-config.yaml --config app-config.production.yaml`
- Environment: `NODE_ENV=production`, `NODE_OPTIONS=--no-node-snapshot`

#### Build Requirements

Before building the Docker image:

```bash
yarn install --immutable    # Install all dependencies
yarn tsc                    # Compile TypeScript
yarn build:backend          # Build backend and create skeleton/bundle archives
```

Build command:
```bash
docker build -f packages/backend/Dockerfile -t backstage .
# Or
yarn build-image
```

#### Container Deployment

The container expects:
- **Configuration files**: `app-config.yaml`, `app-config.production.yaml`
- **Environment variables**: Database credentials, GitHub tokens
- **Network**: Port 7007 accessible to clients
- **Volumes** (optional):
  - TechDocs storage
  - Plugin data directories

Example container run:
```bash
docker run -p 7007:7007 \
  -e POSTGRES_HOST=db.example.com \
  -e POSTGRES_PORT=5432 \
  -e POSTGRES_USER=backstage \
  -e POSTGRES_PASSWORD=secret \
  -e GITHUB_TOKEN=ghp_xxx \
  backstage
```

### Scalability Considerations

**Current State:**
- Single-instance deployment
- Stateful (file-based TechDocs storage)
- Session state in database

**For Production Scale:**
- Load balancer required for multiple backend instances
- External TechDocs storage (S3, GCS, Azure Blob)
- Redis for session sharing
- PostgreSQL connection pooling
- Separate search engine (Elasticsearch recommended)

## Security

### Authentication

**Development:**
- Guest authentication enabled (no credentials)
- GitHub OAuth for realistic testing

**Production:**
- Guest authentication enabled (consider disabling)
- GitHub OAuth required for GitHub operations
- JWT tokens for service-to-service auth

### Authorization

**Current State:**
- Permission system enabled
- Policy: Allow-all (no access restrictions)

**Production Recommendations:**
- Implement custom permission policies
- Define resource-level permissions
- Role-based access control (RBAC)
- Conditional rules for sensitive operations

### Security Hardening Checklist

- [ ] Disable guest authentication in production
- [ ] Implement custom permission policies
- [ ] Rotate GitHub tokens regularly
- [ ] Enable backend-to-backend authentication (`backend.auth.keys`)
- [ ] Use environment variables for all secrets (never commit)
- [ ] Configure CSP headers appropriately
- [ ] Enable HTTPS/TLS for all production traffic
- [ ] Set up GitHub webhook secret validation
- [ ] Implement rate limiting on API endpoints
- [ ] Regular security updates for dependencies
- [ ] Database connection encryption (SSL/TLS)
- [ ] Audit logging for sensitive operations

### Data Security

**Sensitive Data:**
- GitHub Personal Access Tokens
- OAuth client secrets
- Database credentials
- User session tokens

**Protection Mechanisms:**
- Environment variable substitution (not committed)
- Secret redaction in logs (3 secrets redacted at startup)
- Database encryption at rest (dependent on PostgreSQL config)
- HTTPS for data in transit

## Extensibility

### Adding New Plugins

#### Backend Plugin

1. Install the plugin package:
   ```bash
   yarn workspace backend add @backstage/plugin-example-backend
   ```

2. Register in `packages/backend/src/index.ts`:
   ```typescript
   backend.add(import('@backstage/plugin-example-backend'));
   ```

3. Add configuration to `app-config.yaml` if needed

#### Frontend Plugin

1. Install the plugin package:
   ```bash
   yarn workspace app add @backstage/plugin-example
   ```

2. Add to appropriate location:
   - **New page**: Add route in `App.tsx`
   - **Entity tab**: Add to `EntityPage.tsx`
   - **Search result**: Configure in search settings

### Custom Plugins

Create new plugins using the Backstage CLI:

```bash
yarn new
# Select: plugin (for frontend) or backend-plugin
```

Plugin structure:
```
plugins/
└── my-plugin/
    ├── src/
    │   ├── plugin.ts        # Plugin definition
    │   ├── components/      # React components
    │   └── api/             # API clients
    └── package.json
```

### Customization Points

**1. Entity Processing**
- Custom entity providers (beyond GitHub)
- Entity processors for transformation
- Custom entity types and relations

**2. Software Templates**
- Custom scaffolder actions
- Organization-specific templates
- Template parameter validation

**3. TechDocs**
- Custom MkDocs themes
- TechDocs addons for extra features
- Custom documentation processors

**4. Search**
- Additional search collators
- Custom search result components
- External search engine integration

**5. Theme and Branding**
- Custom Material-UI theme
- Logo and color scheme
- Custom homepage layout

### GitHub Integration Extensibility

Current GitHub integrations:
- Repository discovery
- OAuth authentication
- Scaffolder actions (repository creation, webhooks, etc.)

Potential additions:
- GitHub Organizations plugin (team sync)
- GitHub Insights plugin (metrics and analytics)
- GitHub Security plugin (vulnerability scanning)
- Custom GitHub Actions integration

## Monitoring and Observability

### Logging

**Current State:**
- Console logging with structured format
- Log levels: info, warn, error
- Plugin-specific log prefixes
- Request logging via rootHttpRouter

**Log Information:**
- Timestamp (ISO 8601)
- Log level
- Plugin/component name
- Message
- Additional context fields

### Health Checks

**Available Endpoints:**
- Backend: `http://localhost:7007/healthcheck` (via app-backend plugin)

### Metrics

**Task Scheduler:**
- Task execution metrics
- Task duration
- Task success/failure rates

**Search:**
- Index size
- Index update duration
- Query performance

### Troubleshooting

**Common Issues:**

1. **Catalog not discovering repositories**
   - Check `GITHUB_TOKEN` has correct permissions
   - Verify organization name in config
   - Check scheduled task logs

2. **Authentication failures**
   - Verify OAuth app configuration
   - Check callback URLs
   - Review auth logs

3. **TechDocs not generating**
   - Ensure Docker is running
   - Check Docker permissions
   - Verify mkdocs.yml in repositories

4. **Search not working**
   - Verify search collators are running
   - Check search index state
   - Review search backend logs

## Maintenance

### Database Migrations

Handled automatically by plugins on startup. Check logs for:
```
[catalog] info Performing database migration
```

### Backup Strategy

**Development:**
- No backup needed (in-memory database)

**Production:**
- PostgreSQL backup (pg_dump recommended)
- Backup frequency: Daily minimum
- Backup retention: Organization policy
- Include: All database tables, not just entities

### Update Strategy

**Dependencies:**
```bash
# Check for outdated packages
yarn upgrade-interactive

# Update Backstage packages
yarn backstage-cli versions:bump
```

**Testing Updates:**
1. Update in development environment
2. Run full test suite: `yarn test:all`
3. Verify integration tests: `yarn test:e2e`
4. Manual smoke testing
5. Deploy to staging
6. Production deployment

### Performance Optimization

**Current Optimizations:**
- Rspack for fast frontend bundling
- PostgreSQL search engine in production
- Cached build stages in Docker
- Lazy loading of frontend plugins

**Additional Optimizations:**
- CDN for static assets
- Backend response caching
- Database query optimization
- Entity processing parallelization

## Appendix

### Port Assignments

| Service | Port | Purpose |
|---------|------|---------|
| Frontend (dev) | 3000 | React development server |
| Backend | 7007 | Backend API server |

### File Structure

```
backstage/
├── .github/               # GitHub Actions workflows
├── .yarn/                 # Yarn Berry cache
├── docs/                  # Project documentation
│   └── architecture.md    # This file
├── examples/              # Example catalog entities
├── packages/
│   ├── app/              # Frontend application
│   │   ├── public/       # Static assets
│   │   └── src/          # React components
│   │       ├── components/
│   │       │   ├── Root/
│   │       │   └── catalog/
│   │       │       └── EntityPage.tsx
│   │       ├── App.tsx
│   │       └── apis.ts
│   └── backend/          # Backend application
│       ├── Dockerfile    # Production container
│       └── src/
│           └── index.ts  # Plugin registration
├── plugins/              # Custom plugins (empty)
├── app-config.yaml       # Base configuration
├── app-config.production.yaml  # Production config
├── backstage.json        # Backstage version
├── package.json          # Root package manifest
└── yarn.lock            # Dependency lock file
```

### Reference Documentation

- [Backstage Official Documentation](https://backstage.io/docs)
- [Backstage Architecture](https://backstage.io/docs/overview/architecture-overview)
- [New Backend System](https://backstage.io/docs/backend-system/)
- [Plugin Development](https://backstage.io/docs/plugins/)
- [GitHub Integration](https://backstage.io/docs/integrations/github/)

### Version History

| Date | Version | Changes |
|------|---------|---------|
| 2025-12-01 | 1.0 | Initial architecture documentation |

---

**Maintained by:** darrylbowler72 organization
**Last Updated:** 2025-12-01
**Backstage Version:** 1.45.0
