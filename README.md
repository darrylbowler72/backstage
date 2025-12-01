# Backstage Developer Portal

[![Backstage](https://img.shields.io/badge/Backstage-1.45.0-blue)](https://backstage.io)
[![Node.js](https://img.shields.io/badge/Node.js-20%20%7C%2022-green)](https://nodejs.org)
[![Yarn](https://img.shields.io/badge/Yarn-4.4.1-2c8ebb)](https://yarnpkg.com)

A comprehensive developer portal built on [Backstage](https://backstage.io) for the **darrylbowler72** organization. This platform provides a unified interface for software catalog management, technical documentation, software templates, CI/CD monitoring, and Kubernetes resource management.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Development](#development)
- [Deployment](#deployment)
- [Project Structure](#project-structure)
- [Documentation](#documentation)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

## Overview

This Backstage instance serves as the central hub for developer tools and services, integrating with GitHub to provide:

- **Service Catalog**: Automatic discovery and management of software components, APIs, systems, and resources
- **Technical Documentation**: TechDocs integration with MkDocs for documentation as code
- **Software Templates**: Scaffolding tools for creating new projects with best practices
- **CI/CD Monitoring**: GitHub Actions integration for workflow visibility
- **Kubernetes Integration**: Real-time cluster resource monitoring
- **Search**: Full-text search across catalog entities and documentation
- **Authentication**: GitHub OAuth and guest authentication

**Organization:** darrylbowler72
**Backstage Version:** 1.45.0
**Primary Integration:** GitHub

## Features

### 🗂️ Software Catalog

- **Automatic GitHub Discovery**: Scans repositories every 30 minutes for `catalog-info.yaml` files
- **Entity Management**: Components, Systems, APIs, Resources, Users, and Groups
- **Relationship Visualization**: Interactive graph showing entity dependencies
- **Organization Hierarchy**: Team and user management

### 📚 Technical Documentation

- **TechDocs**: Documentation as code using MkDocs
- **Docker-based Generation**: Isolated, reproducible documentation builds
- **Report Issues**: Direct feedback mechanism for documentation
- **Search Integration**: Full-text search across all docs

### 🛠️ Software Templates

- **GitHub Integration**: Create repositories, branches, and pull requests
- **Scaffolder Actions**: 40+ built-in actions including:
  - Repository creation and management
  - File operations (read, write, rename, delete)
  - Catalog registration
  - Webhook and branch protection setup
- **Notification System**: Real-time template execution updates

### 🔄 CI/CD Integration

- **GitHub Actions Plugin**: View workflows and recent runs
- **Entity-level Monitoring**: CI/CD status directly on component pages
- **Workflow Dispatch**: Trigger workflows from the UI

### ☸️ Kubernetes

- **Cluster Integration**: View pods, deployments, services, and more
- **Multi-cluster Support**: Configure multiple Kubernetes clusters
- **Entity Linking**: Connect catalog components to K8s resources

### 🔍 Search

- **PostgreSQL Engine**: Production-ready search with full-text indexing
- **Multiple Collators**: Catalog entities and TechDocs indexed separately
- **Auto-indexing**: Updates every 10 minutes

### 🔐 Authentication & Authorization

- **GitHub OAuth**: Production authentication with organization access
- **Guest Provider**: Development authentication (no credentials required)
- **Permission Framework**: Extensible RBAC system (currently allow-all)

### 🔔 Notifications & Signals

- **Real-time Updates**: WebSocket-based notification system
- **Event Bus**: Cross-plugin event communication
- **Notification Center**: Centralized notification management

## Prerequisites

Before you begin, ensure you have the following installed:

- **Node.js**: Version 20 or 22 ([Download](https://nodejs.org))
- **Yarn**: Version 4.4.1 (installed automatically with corepack)
- **Docker**: Required for TechDocs generation ([Download](https://www.docker.com))
- **Git**: For version control ([Download](https://git-scm.com))

### Required Environment Variables

Create a `.env` file or set these environment variables:

```bash
# GitHub Integration (Required)
GITHUB_TOKEN=ghp_your_personal_access_token_here
GITHUB_CLIENT_ID=your_oauth_app_client_id
GITHUB_CLIENT_SECRET=your_oauth_app_client_secret

# PostgreSQL (Production Only)
POSTGRES_HOST=your_postgres_host
POSTGRES_PORT=5432
POSTGRES_USER=backstage
POSTGRES_PASSWORD=your_secure_password
```

#### GitHub Personal Access Token

Create a token at: https://github.com/settings/tokens

Required scopes:
- `repo` (Full control of private repositories)
- `read:org` (Read org and team membership)
- `read:user` (Read user profile data)
- `user:email` (Access user email addresses)

#### GitHub OAuth Application

Create an OAuth app at: https://github.com/organizations/darrylbowler72/settings/applications

- **Homepage URL**: `http://localhost:3000` (dev) or your production URL
- **Authorization callback URL**: `http://localhost:7007/api/auth/github/handler/frame`

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/darrylbowler72/backstage.git
cd backstage
```

### 2. Install Dependencies

```bash
yarn install
```

This will install all dependencies for the monorepo using Yarn workspaces.

### 3. Configure Environment Variables

```bash
# Copy the example config (if provided) or create app-config.local.yaml
cp app-config.yaml app-config.local.yaml

# Edit with your credentials
# Note: app-config.local.yaml is git-ignored for security
```

Add your GitHub credentials to `app-config.local.yaml`:

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

auth:
  providers:
    github:
      development:
        clientId: ${GITHUB_CLIENT_ID}
        clientSecret: ${GITHUB_CLIENT_SECRET}
```

### 4. Start the Application

```bash
yarn start
```

This starts both the frontend (port 3000) and backend (port 7007) in development mode.

Access the application at: **http://localhost:3000**

### 5. Sign In

- Click "Sign In" in the top right
- Choose "GitHub" or "Guest" authentication
- For Guest: No credentials needed (development only)
- For GitHub: Authorize the OAuth application

## Configuration

Configuration is managed through layered YAML files with environment variable substitution.

### Configuration Files

| File | Purpose | Git Tracked |
|------|---------|-------------|
| `app-config.yaml` | Base configuration for development | ✅ Yes |
| `app-config.local.yaml` | Local overrides (secrets) | ❌ No |
| `app-config.production.yaml` | Production settings | ✅ Yes |

### Key Configuration Sections

#### Catalog Discovery

Configure GitHub repository discovery in `app-config.yaml`:

```yaml
catalog:
  providers:
    github:
      providerId:
        organization: 'darrylbowler72'
        filters:
          repository: '.*'
        schedule:
          frequency: { minutes: 30 }
          timeout: { minutes: 3 }
```

#### Database

**Development** (SQLite):
```yaml
backend:
  database:
    client: better-sqlite3
    connection: ':memory:'
```

**Production** (PostgreSQL):
```yaml
backend:
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: ${POSTGRES_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
```

#### TechDocs

```yaml
techdocs:
  builder: 'local'
  generator:
    runIn: 'docker'  # Requires Docker
  publisher:
    type: 'local'
```

## Development

### Common Commands

```bash
# Start development server
yarn start

# Type checking
yarn tsc
yarn tsc:full  # Without skipLibCheck

# Linting
yarn lint           # Check changed files
yarn lint:all       # Check all files
yarn fix            # Auto-fix issues

# Testing
yarn test           # Run tests for changed packages
yarn test:all       # Run all tests with coverage
yarn test:e2e       # Run end-to-end tests

# Building
yarn build:backend  # Build backend package
yarn build:all      # Build all packages

# Code formatting
yarn prettier:check

# Clean build artifacts
yarn clean

# Create new plugin
yarn new
```

### Development Workflow

1. **Start the dev server**: `yarn start`
2. **Make changes**: Edit files in `packages/app` or `packages/backend`
3. **Hot reload**: Frontend changes reload automatically
4. **Backend changes**: Require server restart
5. **Type check**: Run `yarn tsc` before committing
6. **Test**: Run `yarn test` for affected packages

### Adding Plugins

#### Backend Plugin

```bash
# Install the plugin
yarn workspace backend add @backstage/plugin-example-backend

# Register in packages/backend/src/index.ts
backend.add(import('@backstage/plugin-example-backend'));
```

#### Frontend Plugin

```bash
# Install the plugin
yarn workspace app add @backstage/plugin-example

# Add to App.tsx or EntityPage.tsx as needed
```

### Creating Custom Plugins

```bash
# Use the interactive CLI
yarn new

# Select plugin type:
# - plugin (frontend)
# - backend-plugin
# - backend-plugin-module
```

## Deployment

### Docker Deployment

#### Prerequisites

Before building the Docker image, you must build the backend:

```bash
yarn install --immutable
yarn tsc
yarn build:backend
```

This generates:
- `packages/backend/dist/skeleton.tar.gz` - Package structure
- `packages/backend/dist/bundle.tar.gz` - Application bundle

#### Build Image

```bash
# From project root
docker build -f packages/backend/Dockerfile -t backstage .

# Or use the npm script
yarn build-image
```

#### Run Container

```bash
docker run -d \
  -p 7007:7007 \
  -e POSTGRES_HOST=your_db_host \
  -e POSTGRES_PORT=5432 \
  -e POSTGRES_USER=backstage \
  -e POSTGRES_PASSWORD=secure_password \
  -e GITHUB_TOKEN=ghp_your_token \
  --name backstage \
  backstage
```

### Production Considerations

- **Database**: Use PostgreSQL with SSL/TLS
- **Secrets**: Use environment variables or secret management
- **Storage**: External TechDocs storage (S3, GCS, Azure Blob)
- **Load Balancing**: Multiple backend instances with shared session store
- **HTTPS**: Terminate SSL at load balancer or reverse proxy
- **Authentication**: Disable guest provider in production
- **Permissions**: Implement custom permission policies
- **Monitoring**: Set up logging, metrics, and health checks

## Project Structure

```
backstage/
├── .github/              # GitHub Actions workflows
├── .yarn/                # Yarn Berry cache and releases
├── docs/                 # Project documentation
│   ├── architecture.md   # Detailed architecture documentation
│   └── index.md          # Documentation index
├── examples/             # Example catalog entities and templates
├── packages/
│   ├── app/              # Frontend React application
│   │   ├── public/       # Static assets
│   │   └── src/
│   │       ├── components/
│   │       │   ├── Root/
│   │       │   └── catalog/
│   │       │       └── EntityPage.tsx  # Entity detail pages
│   │       ├── App.tsx               # Main app component
│   │       └── apis.ts               # API configurations
│   └── backend/          # Backend Node.js application
│       ├── Dockerfile    # Production container definition
│       └── src/
│           └── index.ts  # Plugin registration and startup
├── plugins/              # Custom plugins (empty - reserved)
├── .gitignore            # Git ignore rules
├── app-config.yaml       # Base configuration
├── app-config.production.yaml  # Production configuration
├── backstage.json        # Backstage version metadata
├── CLAUDE.md             # Claude Code project instructions
├── package.json          # Root package manifest
├── README.md             # This file
└── yarn.lock             # Dependency lock file
```

### Key Directories

- **`packages/app/`**: React frontend with routing, components, and plugins
- **`packages/backend/`**: Node.js backend with API, database, and plugin system
- **`plugins/`**: Custom plugin development (currently empty)
- **`docs/`**: Project documentation including architecture details
- **`examples/`**: Sample catalog entities and templates for reference

## Documentation

- **[Architecture Documentation](docs/architecture.md)**: Comprehensive technical architecture, deployment, and operations guide
- **[Claude Code Instructions](CLAUDE.md)**: Project overview and development guidelines for Claude Code
- **[Backstage Official Docs](https://backstage.io/docs)**: Official Backstage documentation
- **[GitHub Integration](https://backstage.io/docs/integrations/github/)**: GitHub integration guide
- **[Plugin Development](https://backstage.io/docs/plugins/)**: Creating custom plugins

## Troubleshooting

### Common Issues

#### Port Already in Use

```bash
# Check what's using the port
netstat -ano | findstr :3000
netstat -ano | findstr :7007

# Kill the process using the port
taskkill /PID <process_id> /F
```

#### Catalog Not Discovering Repositories

1. Verify `GITHUB_TOKEN` has correct permissions
2. Check organization name in `app-config.yaml`
3. Ensure repositories contain `catalog-info.yaml`
4. Review backend logs for discovery errors

#### TechDocs Not Generating

1. Ensure Docker is running: `docker info`
2. Check Docker permissions
3. Verify `mkdocs.yml` exists in repositories
4. Review TechDocs backend logs

#### Authentication Failures

1. Verify OAuth app configuration in GitHub
2. Check callback URL matches exactly
3. Review auth backend logs
4. Ensure `GITHUB_CLIENT_ID` and `GITHUB_CLIENT_SECRET` are set

#### Search Not Working

1. Check search collators are running in backend logs
2. Verify search backend plugin is initialized
3. Wait for initial indexing (10 minutes)
4. Check PostgreSQL connection in production

### Getting Help

- Check backend logs in the terminal
- Review browser console for frontend errors
- Enable debug logging in `app-config.yaml`:
  ```yaml
  backend:
    listen:
      port: 7007
    log:
      level: debug
  ```

## Contributing

### Development Guidelines

1. **Follow TypeScript best practices**: Use types, avoid `any`
2. **Write tests**: Add tests for new features
3. **Run linters**: `yarn lint` and `yarn prettier:check`
4. **Type check**: `yarn tsc` before committing
5. **Document changes**: Update README and docs as needed
6. **Commit messages**: Use conventional commits format

### Commit Message Format

```
type(scope): subject

body

footer
```

Examples:
- `feat(catalog): add GitHub org provider`
- `fix(auth): resolve OAuth callback issue`
- `docs(readme): update installation instructions`

### Pull Request Process

1. Create a feature branch from `main`
2. Make your changes with clear commits
3. Run tests and linting: `yarn test && yarn lint`
4. Push your branch and create a pull request
5. Address review feedback
6. Squash and merge when approved

## License

This project is built on Backstage, which is licensed under Apache 2.0. See the [Backstage License](https://github.com/backstage/backstage/blob/master/LICENSE) for details.

---

**Built with [Backstage](https://backstage.io)**
**Maintained by:** darrylbowler72 organization
**Last Updated:** 2025-12-01
