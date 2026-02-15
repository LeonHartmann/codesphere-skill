# Landscape Deployments Reference

## Contents
- When to use landscape
- Shared vs independent resources
- Networking (private communication, public access, path-based routing)
- Complete examples (frontend+backend, app+Postgres, Ollama+WebUI, n8n+Ollama)
- Migrating from single-service to landscape
- Managed services (Postgres, MySQL, Redis, MongoDB)
- Service naming conventions

Landscape deployments run multiple independently-scaling services within a single Codesphere workspace. Requires `schemaVersion: v0.2`.

## When to Use Landscape

- Frontend + backend as separate services
- App server + background worker
- App + managed database (Postgres, Redis, MySQL, MongoDB)
- Separating IDE resources from application runtime (prevents OOM crashes affecting development)
- Any multi-service architecture within one workspace

## Shared Resources

All services in a landscape share:
- **Filesystem**: `/home/user/app` and `/nix/store`
- **Prepare stage**: Dependencies installed once, available to all services
- **Test stage**: Runs once before any service starts
- **Environment variables**: Set once, available to all services

Each service gets independent:
- **CPU and RAM** (based on `plan` setting)
- **Replica count**
- **Deployment mode** (always-on vs standby)
- **Network configuration**

## Networking

### Private Communication Between Services

Services communicate internally via:
```
http://ws-server-[WorkspaceId]-[serviceName]:3000
```

Example: If workspace ID is `abc123` and service is named `backend`:
```
http://ws-server-abc123-backend:3000
```

### Public Access URLs

Public services are accessible at:
```
<workspace-id>-<port>-<service-name>.codesphere.com
```

### Path-Based Routing

Route different URL paths to different services:

```yaml
run:
  frontend:
    steps:
      - command: npm run start:frontend
    plan: 8
    replicas: 1
    network:
      paths:
        - port: 3000
          path: /
          stripPath: false
  api:
    steps:
      - command: npm run start:api
    plan: 8
    replicas: 1
    network:
      paths:
        - port: 3000
          path: /api
          stripPath: true
```

**`stripPath: true`** removes the path prefix before forwarding. A request to `/api/users` arrives at the service as `/users`.

**`stripPath: false`** keeps the path prefix. A request to `/app/dashboard` arrives as `/app/dashboard`. The application must handle the full path.

### Important Networking Rules

- Each service MUST serve on `0.0.0.0:3000`
- `isPublic: true` makes a service externally accessible
- `isPublic: false` means only reachable via private networking
- Applications using path routing must include their path prefix in their routes (unless `stripPath: true`)

## Complete Landscape Examples

### Frontend + Backend

```yaml
schemaVersion: v0.2
prepare:
  steps:
    - name: Install dependencies
      command: npm install
test:
  steps: []
run:
  frontend:
    steps:
      - name: Start frontend
        command: npm run start:frontend
    plan: 8
    replicas: 1
    network:
      ports:
        - port: 3000
          isPublic: true
      paths:
        - port: 3000
          path: /
          stripPath: false
  backend:
    steps:
      - name: Start backend
        command: npm run start:backend
    plan: 8
    replicas: 2
    network:
      ports:
        - port: 3000
          isPublic: false
      paths:
        - port: 3000
          path: /api
          stripPath: true
```

### App + Managed PostgreSQL

```yaml
schemaVersion: v0.2
prepare:
  steps:
    - name: Install dependencies
      command: npm install
    - name: Run migrations
      command: npm run db:migrate
test:
  steps: []
run:
  app:
    steps:
      - name: Start application
        command: npm start
    plan: 8
    replicas: 1
    network:
      ports:
        - port: 3000
          isPublic: true
      paths:
        - port: 3000
          path: /
          stripPath: false
  database:
    provider:
      name: postgres
      version: v1
    plan:
      id: 0
      parameters:
        storage: 10000
        cpu: 5
        memory: 500
    config:
      version: "17.6"
      userName: app
      databaseName: mydb
    secrets:
      userPassword: <set-a-secure-password>
      superuserPassword: <set-a-secure-password>
```

### Ollama + Open WebUI

```yaml
schemaVersion: v0.2
prepare:
  steps:
    - name: Install ollama
      command: nix-env -iA nixpkgs.ollama
    - name: Install process-compose
      command: nix-env -iA nixpkgs.process-compose
    - name: Install uv
      command: nix-env -iA nixpkgs.uv
    - name: Install open-webui
      command: uv add open-webui && uv sync
test:
  steps: []
run:
  ollama:
    steps:
      - name: Run ollama
        command: process-compose -t=False -f process-compose.models.yaml
    plan: 22
    replicas: 1
    isPublic: false
  open-webui:
    steps:
      - name: Run open-webui
        command: >
          . .env && export OLLAMA_BASE_URL=$OLLAMA_BASE_URL &&
          uv run open-webui serve --port 3000
    plan: 22
    replicas: 1
    isPublic: true
    network:
      paths:
        - port: 3000
          path: /
          stripPath: false
```

### n8n + Ollama (Full Example)

```yaml
schemaVersion: v0.2
prepare:
  steps:
    - name: Install Node.js 24
      command: nix-env -iA nixpkgs.nodejs_24
    - name: Install dependencies
      command: npm install
test:
  steps: []
run:
  n8n-frontend:
    steps:
      - name: n8n start
        command: N8N_PORT=3000 ./node_modules/n8n/bin/n8n start
    plan: 8
    replicas: 1
    network:
      ports:
        - port: 3000
          isPublic: false
      paths:
        - port: 3000
          path: /
          stripPath: false
  ollama-service:
    steps:
      - command: process-compose -t=False -f process-compose.models.yaml
    plan: 9
    replicas: 1
    network:
      ports:
        - port: 3000
          isPublic: false
      paths:
        - port: 3000
          path: /model
          stripPath: true
```

## Migrating to Landscape

If you have an existing single-service `ci.yml` and want to migrate:

1. Use the **"Migrate to Landscape"** button in the CI editor (Setup > CI), OR
2. Delete the existing `ci.yml` and create a new one with `schemaVersion: v0.2`

**Important:** An existing single-service `ci.yml` may block landscape parsing. If migration button doesn't work, delete and recreate.

## Managed Services

Available managed services through landscape deployments:

| Service | Provider Name | Notes |
|---------|--------------|-------|
| PostgreSQL | `postgres` | Versions configurable (e.g., "17.6") |
| MySQL | `mysql` | Via marketplace |
| Redis | `redis` | Via marketplace |
| MongoDB | `mongodb` | Via marketplace |

Managed services are configured under `run.<serviceName>` with a `provider` block instead of `steps`. They get their own `plan`, `config`, and `secrets`.

Connection details for managed services are automatically available to other services in the same landscape via environment variables.

## Service Naming

- Service names should match monorepo directory names where possible
- Use lowercase with hyphens (e.g., `my-service`, not `myService`)
- Each service name becomes part of the internal networking URL
